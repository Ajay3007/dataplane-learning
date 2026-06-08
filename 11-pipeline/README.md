# Module 11 — Multi-lcore RX/TX Pipeline

> **Reference code** — requires DPDK installed.

## What you learn

How to wire together EAL (Module 08), mempool (Module 09), and port init
(Module 10) into a complete multi-lcore packet processing pipeline — the
skeleton of the DP application at runtime.

This is the structural blueprint: RX lcore polls the NIC, distributes
packets to worker lcores via `rte_ring`, workers parse + apply policy,
forward packets to a shared TX ring, and the TX lcore drains it to the NIC.

---

## Pipeline topology

```
NIC port 0
  │  (DMA fills mbufs from pool — Module 09)
  ↓
RX lcore (lcore 1)
  rte_eth_rx_burst(port=0, queue=0, mbufs, BURST_SIZE=32)
  │
  │  Round-robin distribute across worker rx_rings
  │
  ├── rx_ring[0] ──► Worker lcore 2
  ├── rx_ring[1] ──► Worker lcore 3
  ├── rx_ring[2] ──► Worker lcore 4
  └── rx_ring[3] ──► Worker lcore 5
                          │
           For each mbuf: │
             parse ETH/IP/UDP-TCP          (Module 05)
             DNS: dns_parse_message()       (Module 06)
             TLS: tls_extract_sni()         (Module 07)
             policy lookup                  (Module 22)
               ALLOW    → tx_ring
               DROP     → rte_pktmbuf_free()
               SINKHOLE → modify in-place → tx_ring  (Module 23)
                          │
                       tx_ring (shared MPSC)

                          │
TX lcore (lcore 1)
  rte_ring_dequeue_burst(tx_ring, mbufs, BURST_SIZE)
  rte_eth_tx_burst(port=0, queue=0, mbufs, nb)
  free unsent mbufs
  │
  ↓
NIC port 0 → wire
```

---

## Where this fits in the real application

This module IS the runtime structure of the DP application. Everything else is setup or detail:

| Layer | Module | Called from |
|---|---|---|
| EAL init | 08 | startup, once |
| Mempool | 09 | startup, once |
| Port init | 10 | startup, once |
| **Pipeline** | **11** | **runtime, forever** |
| rte_hash ops | 16 | inside worker policy stub |
| Hyperscan scan | 19–22 | inside worker policy stub |
| DNS sinkhole | 23 | inside worker SINKHOLE case |
| Kafka | 25–26 | main lcore control loop |

---

## Files

| File | Purpose |
|---|---|
| `pipeline.c` | Full pipeline: RX/worker/TX lcore functions, ring setup, stats, shutdown |
| `Makefile` | DPDK build |

---

## Key concepts in the code

### 1. Each lcore has exactly one job

```
RX lcore:     ONLY calls rte_eth_rx_burst() + rte_ring_enqueue()
Worker lcore: ONLY does parse + policy + ring enqueue/dequeue
TX lcore:     ONLY calls rte_ring_dequeue() + rte_eth_tx_burst()
```

If the RX lcore also runs policy logic, it slows down and the NIC RX
descriptor ring fills → `stats.imissed` increments → silent packet loss.
The role separation is what makes DPDK applications scale linearly with cores.

### 2. `rte_pause()` — never `sleep()`

```c
if (nb_rx == 0) {
    rte_pause();   /* x86 PAUSE: hints CPU it's in a spin loop */
    continue;
}
```

`sleep()` or `usleep()` yield the thread to the OS scheduler. The lcore
may not resume for milliseconds — all in-flight NIC descriptors fill up.
`rte_pause()` expands to the x86 `PAUSE` instruction: the CPU reduces
power slightly and signals the pipeline that it's spinning, but it never
leaves the core. At 25 Gbps there's always a packet within microseconds.

### 3. SPSC ring for RX→worker, MPSC for worker→TX

```c
/* rx_ring: single producer (RX lcore), single consumer (one worker) */
rx_rings[i] = rte_ring_create(name, RING_SIZE, socket,
                               RING_F_SP_ENQ | RING_F_SC_DEQ);

/* tx_ring: multiple producers (all workers), single consumer (TX lcore) */
tx_ring = rte_ring_create("tx_ring", RING_SIZE, socket, 0); /* MPSC default */
```

SPSC is faster than MPMC (no CAS atomic needed — just head/tail with
memory barriers). Use the most restrictive type you can justify.

### 4. `unlikely()` — branch prediction hints

```c
if (unlikely(mbuf->nb_segs > 1))    /* rarely true at MTU=1500 */
    return DECISION_DROP;

if (unlikely(rte_ring_enqueue(...) != 0))  /* rarely true if ring sized correctly */
    rte_pktmbuf_free(m);
```

`unlikely(x)` expands to `__builtin_expect((x), 0)`. It tells the CPU
branch predictor to optimise for the false case. On the hot path (millions
of packets/sec), branch mispredictions cost ~15 cycles each. `unlikely()`
reduces mispredictions for rare error paths to near zero.

### 5. Free unsent TX mbufs — the leak you'll hit

```c
uint16_t nb_tx = rte_eth_tx_burst(port, queue, mbufs, nb);

/* MANDATORY: free what the NIC didn't send */
for (uint16_t i = nb_tx; i < nb; i++)
    rte_pktmbuf_free(mbufs[i]);
```

`rte_eth_tx_burst()` may return `nb_tx < nb` when the NIC TX descriptor
ring is full (back-pressure). The NIC only frees mbufs it actually DMA'd.
If you don't free the unsent ones, the pool slowly drains to zero and
`rte_pktmbuf_alloc()` starts returning NULL — silent stall of the pipeline.

This is one of the most common bugs in new DPDK code.

### 6. Shutdown ordering — drain the rings

```
Stop workers → wait for workers to exit →
Stop TX lcore (TX drains tx_ring before exiting) →
wait for TX →
rte_eth_dev_stop() → rte_eal_cleanup()
```

Never stop the TX lcore before workers. Workers may enqueue a final burst
to tx_ring after the stop signal. If TX exits first, those packets are
stuck in the ring and freed without being transmitted — the client's
DNS response (sinkhole) is never delivered.

### 7. Where parsers and policy plug in

```c
/* worker_lcore_func inner loop — where all module connections happen: */

int decision = parse_and_decide(mbufs[i], &is_dns, &is_tls);

/*
 * parse_and_decide() currently calls:
 *   - Inline ETH/IP/UDP-TCP parsing     (Module 05)
 *   - Stub DNS qname extraction          (Module 06 replaces this)
 *   - Stub TLS SNI detection             (Module 07 replaces this)
 *   - policy_stub()                      (Module 22 replaces this with:)
 *       rte_hash_lookup_data(domain_details_table, domain, &fd)
 *       → hs_scan_domain_group() on hash miss
 *
 * SINKHOLE case calls:
 *   dns_build_sinkhole_v4/dns_build_sinkhole_v6()  (Module 23)
 */
```

---

## Scaling to higher throughput

For > 25 Gbps or more workers, replace the RX lcore round-robin with RSS:

```
With RSS (Module 10):
  NIC queues packets by hash(src_ip, src_port) → queue N
  Each worker polls its own NIC queue directly:

  worker_lcore_func():
    nb_rx = rte_eth_rx_burst(port_id, my_queue, mbufs, BURST)
    // No rx_ring needed — worker gets packets straight from NIC
```

This eliminates the RX lcore and rx_rings entirely, removing one ring
hop from the critical path. The DP application uses a distributor-based approach
for its RSS-alternative distribution.

---

## Next module

**Module 12 — rte_hash CRUD**: Deep dive into DPDK's `rte_hash` table
operations — the exact API used for `domain_details_table` in each
`group_struct`. This replaces the `policy_stub()` in this module with
a real O(1) hash table lookup.
