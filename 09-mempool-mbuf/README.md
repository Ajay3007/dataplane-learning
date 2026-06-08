# Module 09 — Mempool + mbuf

> **Reference code** — requires DPDK installed.

## What you learn

How DPDK manages packet memory via pre-allocated mempools and `rte_mbuf`
structures — why mempools exist, how the mbuf memory layout works, the
full alloc/fill/process/free lifecycle, in-place packet modification
(the pattern used by the DNS sinkhole), hardware checksum offload flags,
and multi-segment (chained) mbufs.

---

## Why mempools — the core insight

```
Without mempools (naive approach):
  Packet arrives → malloc(1500) → process → free()
  At 2M pkts/sec: 2M malloc() calls/sec
  malloc has locks + may call brk() → becomes the bottleneck

With mempools (DPDK approach):
  Startup: pre-allocate 65535 mbufs in hugepage memory
  Runtime: "alloc" = pop pointer from lockless ring  (~10 ns)
           "free"  = push pointer back to ring        (~10 ns)
  No syscalls, no locks in the fast path
```

The mempool object ring uses the same design as Module 03 (rte_ring).
The hugepage backing means these allocations never cause TLB misses
(2MB huge pages vs 4KB normal pages = 512× fewer TLB entries needed).

---

## mbuf memory layout

```
One mbuf object in the pool:

  ┌─────────────────────────────┐  ← buf_addr (hugepage address)
  │  struct rte_mbuf  (~128B)   │  metadata: data_off, data_len, pkt_len,
  │                             │  nb_segs, port, ol_flags, hash.rss, ...
  ├─────────────────────────────┤
  │  private area  (0B default) │  app-specific metadata per mbuf
  ├─────────────────────────────┤
  │  headroom      (128B)       │  reserved for prepending headers
  ├─────────────────────────────┤  ← buf_addr + data_off
  │                             │  ← rte_pktmbuf_mtod(m, T*)
  │  packet data                │
  │  (data_len bytes)           │
  │                             │
  ├─────────────────────────────┤
  │  tailroom                   │  available for appending (answer section!)
  └─────────────────────────────┘  ← buf_addr + buf_len
```

`data_off` starts at 128 (the headroom size). `rte_pktmbuf_mtod()` returns
`buf_addr + data_off` — the first byte of the actual packet.

---

## Where this fits in the real application

```
Startup (port_init in app_main.c):
  mbuf_pool = rte_pktmbuf_pool_create("pktmbuf_pool_s0",
                  65535, 256, 0, RTE_MBUF_DEFAULT_BUF_SIZE,
                  rte_socket_id())
  rte_eth_rx_queue_setup(port, queue, nb_desc, socket, &cfg, mbuf_pool)
  ↑ NIC DMA engine will pull mbufs from this pool to fill with received packets

RX lcore:
  nb_rx = rte_eth_rx_burst(port, queue, mbufs, BURST_SIZE)
  ↑ mbufs[] now contain received packets from the pool

Worker lcore:
  eth = rte_pktmbuf_mtod(mbufs[i], eth_hdr_t *)  ← zero-copy header access
  ip4 = (ipv4_hdr_t *)((uint8_t *)eth + ETH_HDR_LEN)
  ...policy decision...

DNS sinkhole (Module 23):
  → rewrite packet in-place via rte_pktmbuf_mtod()
  → rte_pktmbuf_append() for answer section bytes
  → m->ol_flags |= TX_IPV4 | TX_IP_CKSUM | TX_UDP_CKSUM
  → rte_eth_tx_burst() sends it; NIC computes checksum in hardware

TX lcore (or DROP):
  nb_tx = rte_eth_tx_burst(port, queue, mbufs, nb_fwd)
  ↑ TX burst frees mbufs back to pool automatically after transmission
  for dropped mbufs: rte_pktmbuf_free(m)
```

---

## Key concepts in the code

### 1. Pool size: must be 2^k − 1

```c
#define POOL_NUM_MBUFS  8191   /* 2^13 - 1: correct */
#define POOL_NUM_MBUFS  8192   /* 2^13:     WRONG — rte_ring adds 1, wastes a slot */
```

`rte_pktmbuf_pool_create` internally creates an `rte_ring` of size n+1.
Since rte_ring requires power-of-2 capacity, if you pass 8192 the ring
becomes 16384 — wasting 8192 slots of pre-allocated hugepage memory.

### 2. Per-lcore cache — reducing ring contention

```
Without cache: every alloc/free touches the pool's central ring
               → ring lock contention between lcores

With cache_size=256: each lcore keeps 256 mbufs locally
               → alloc/free from local cache (no ring) until cache empties/fills
               → ring only touched when cache needs replenishment
               → typical alloc is ~3 ns instead of ~10 ns
```

Rule: `cache_size` must be `<= RTE_MEMPOOL_CACHE_MAX_SIZE` (512).
A good default is 32–256 depending on burst size.

### 3. `rte_pktmbuf_mtod` — the most-used macro

```c
/* Get typed pointer to packet start */
eth_hdr_t *eth = rte_pktmbuf_mtod(m, eth_hdr_t *);

/* Expands to: */
eth_hdr_t *eth = (eth_hdr_t *)((char *)(m)->buf_addr + (m)->data_off);
```

Every packet access in the codebase goes through this. If you accidentally
use `m->buf_addr` directly you read the mbuf struct, not the packet bytes.

### 4. `rte_pktmbuf_append` — used in DNS sinkhole

```c
/* Reserve space at the tail and return pointer to it */
char *answer_section = rte_pktmbuf_append(m, dns_answer_len);
if (!answer_section) { /* tailroom exhausted */ }
memcpy(answer_section, answer_bytes, dns_answer_len);
/* Update IP total_len and UDP dgram_len to include the new bytes */
ip4->total_len = htons(ntohs(ip4->total_len) + dns_answer_len);
```

This is exactly what `dns_build_sinkhole_v4()` does
in Module 23 — it appends the DNS answer section and adjusts length fields.

### 5. Hardware checksum offload

```c
m->ol_flags |= RTE_MBUF_F_TX_IPV4 | RTE_MBUF_F_TX_IP_CKSUM
             | RTE_MBUF_F_TX_UDP_CKSUM;
ip4->checksum = 0;
udp->checksum = rte_ipv4_phdr_cksum(ip4, m->ol_flags);
```

Setting the IP checksum to 0 and the UDP checksum to the pseudo-header
value tells the NIC TX engine to compute the real checksums in hardware.
The NIC processes `ol_flags` before DMA-ing the packet to the wire.
Software checksum would cost ~50 ns per packet — HW offload costs ~0.

### 6. Multi-segment guard in the real app

```c
nb_rx = rte_eth_rx_burst(port, queue, mbufs, BURST_SIZE);
for (int i = 0; i < nb_rx; i++) {
    if (unlikely(mbufs[i]->nb_segs > 1)) {
        /* Jumbo frame — the DP application drops it rather than paying linearise cost */
        rte_pktmbuf_free(mbufs[i]);
        continue;
    }
    /* Safe to do single-pointer parsing */
    eth = rte_pktmbuf_mtod(mbufs[i], eth_hdr_t *);
    ...
}
```

`unlikely()` is a GCC branch prediction hint — at 1500 MTU, nb_segs > 1
is extremely rare, so hinting the CPU branch predictor here saves ~1 cycle
per packet in the common case.

---

## Pool sizing reference

| Deployment | num_mbufs | Rationale |
|---|---|---|
| Dev/test (VM) | 4095 (2^12-1) | Small hugepage allocation |
| Single port, 4 queues | 16383 (2^14-1) | 4×1024 descriptors + 3× headroom |
| 4 ports, 4 queues each | 65535 (2^16-1) | 16×1024 + 3× headroom |
| DP production | 65535 | Used in app_main.c |

---

## Next module

**Module 10 — Port Init**: Configure a DPDK NIC port — set up RX/TX queues,
descriptor rings, link speed, promiscuous mode, and RSS (Receive Side Scaling)
for distributing packets across multiple RX queues. The mempool from this
module is passed to `rte_eth_rx_queue_setup()`.
