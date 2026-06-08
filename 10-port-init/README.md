# Module 10 — NIC Port Initialization

> **Reference code** — requires DPDK and a DPDK-bound NIC (or `--vdev net_null0`).

## What you learn

How to configure a DPDK NIC port end-to-end: detect capabilities, set up
RX/TX queues and descriptor rings, enable RSS for multi-queue distribution,
configure hardware offloads (checksum, VLAN), start the port, and verify
link state. This is the last setup step before worker lcores can call
`rte_eth_rx_burst()` / `rte_eth_tx_burst()`.

---

## Port init sequence

```
rte_eth_dev_count_avail()            → how many NICs available
rte_eth_dev_info_get(port, &info)    → query NIC capabilities
  │
  ├─ mask requested offloads against info.rx_offload_capa
  ├─ mask RSS hash types against info.flow_type_rss_offloads
  │
rte_eth_dev_configure(port, nb_rx_q, nb_tx_q, &port_conf)
  │  sets mq_mode=RSS, offloads, rss_hf
  │
rte_eth_dev_adjust_nb_rx_tx_desc()   → align descriptor counts to NIC limits
  │
rte_eth_rx_queue_setup() × nb_rx_q   → each queue gets a slice of mbuf pool
rte_eth_tx_queue_setup() × nb_tx_q
  │
rte_eth_dev_start()                  → NIC starts link negotiation + DMA
rte_eth_promiscuous_enable()         → accept all frames (not just own MAC)
  │
check_port_link_status()             → poll until UP (timeout = 9 sec)
```

---

## Where this fits in the real application

```
main() in app_main.c
  │
  ├─► config_load()            (Module 01)
  ├─► logger_init()            (Module 02)
  ├─► rte_eal_init()           (Module 08)
  ├─► rte_pktmbuf_pool_create() (Module 09)
  │
  ├─► port_init(0, mbuf_pool)  ← THIS MODULE
  │     configures port 0 with 4 RX queues + 1 TX queue
  │     RSS distributes subscriber DNS/TLS across 4 worker queues
  │
  ├─► hs_init_global_scratch()  (Module 11)
  ├─► kafka_init()                 (Module 12)
  └─► rte_eal_remote_launch()      (Module 08)
```

---

## Files

| File | Purpose |
|---|---|
| `port_init.c` | Full port init: capability check, configure, queues, start, link poll, stats |
| `Makefile` | DPDK pkg-config build |

---

## Key concepts in the code

### 1. Capability intersection — the most common bug

```c
/* WRONG: request offloads blindly */
port_conf.rxmode.offloads = RTE_ETH_RX_OFFLOAD_CHECKSUM;

/* CORRECT: mask against what the NIC actually supports */
port_conf.rxmode.offloads &= dev_info.rx_offload_capa;
```

If you request `CHECKSUM` on a NIC that doesn't support it,
`rte_eth_dev_configure()` returns `-EINVAL`. The error message
says "invalid argument" — not which offload caused the problem.
Always intersect against `dev_info` capabilities.

The same applies to RSS hash types:
```c
port_conf.rx_adv_conf.rss_conf.rss_hf &= dev_info.flow_type_rss_offloads;
```
A NIC that doesn't support `RSS_UDP` would silently send **all DNS packets
to queue 0** — one worker lcore handles all DNS while the rest are idle.

### 2. RSS — why it's critical for multi-lcore scaling

Without RSS (single RX queue):
```
All packets → queue 0 → RX lcore → ring → single worker lcore
                                            ↑ bottleneck
```

With RSS (4 RX queues, one per worker lcore):
```
DNS from 198.51.100.x → hash=0xA3 % 4 = 3 → queue 3 → worker lcore 6
DNS from 10.0.0.x    → hash=0x51 % 4 = 1 → queue 1 → worker lcore 4
HTTPS from any       → hash=0xC7 % 4 = 3 → queue 3 → worker lcore 6
```

RSS hash is computed by the NIC hardware (Toeplitz algorithm on src/dst
IP + port). Each worker lcore polls its own dedicated queue with
`rte_eth_rx_burst(port, my_queue, mbufs, BURST)` — no contention.

### 3. Descriptor ring depth and mbuf pool sizing

```
RX ring (1024 descriptors):
  NIC pre-fills these with mbuf pointers from the pool.
  When a packet arrives, NIC DMA fills the mbuf and marks the descriptor done.
  rte_eth_rx_burst() collects them.

  If the software is slow and all 1024 descriptors are full:
    → new packets are dropped → stats.imissed increments
    → increase RX_DESC or speed up the worker

Pool must always have more mbufs than (sum of all RX descriptor rings):
  pool_size > nb_rx_queues × RX_DESC × nb_ports × safety_factor(2×)
  4 queues × 1024 desc × 1 port × 2 = 8192 → pool of 8191 is just enough
  In DP production: pool=65535 for comfortable headroom
```

### 4. `rte_eth_dev_adjust_nb_rx_tx_desc`

Different NICs have different constraints on descriptor counts
(minimum, maximum, must-be-multiple-of). Always call this after
`rte_eth_dev_configure()`:

```c
uint16_t nb_rxd = 1024, nb_txd = 1024;
rte_eth_dev_adjust_nb_rx_tx_desc(port, &nb_rxd, &nb_txd);
/* nb_rxd / nb_txd are now adjusted to valid values for this NIC */
```

### 5. Promiscuous mode — why it's necessary

A NIC in normal mode only accepts frames where dst MAC = its own MAC
(or multicast/broadcast). The DP application sits inline between clients and the
internet — packets are addressed to the router, not to the appliance.
Promiscuous mode makes the NIC accept everything regardless of dst MAC.

### 6. TX hardware checksum offload

After DNS sinkhole response is built in Module 23:

```c
m->ol_flags |= RTE_MBUF_F_TX_IPV4 | RTE_MBUF_F_TX_IP_CKSUM
             | RTE_MBUF_F_TX_UDP_CKSUM;
ip4->checksum = 0;
udp->checksum = rte_ipv4_phdr_cksum(ip4, m->ol_flags);
```

For this to work, the TX queue must have been set up with:
```c
txq_conf.offloads = RTE_ETH_TX_OFFLOAD_IPV4_CKSUM | RTE_ETH_TX_OFFLOAD_UDP_CKSUM;
```
AND the port must have been configured with the same offloads in `port_conf.txmode`.
If either is missing, `rte_eth_tx_burst()` silently ignores the flags and
transmits with checksum=0 — every receiver drops the packet.

### 7. `stats.imissed` — the first metric to check

```c
struct rte_eth_stats stats;
rte_eth_stats_get(port, &stats);
if (stats.imissed > 0)
    LOG_WARN("Port %u: %lu packets dropped (NIC ring full)", port, stats.imissed);
```

`imissed` increments when a packet arrives but there's no free mbuf in
the pool (pool exhausted) or no free descriptor in the RX ring (software
too slow). Either way: packets are silently lost.

In the DP application, `imissed` was the first indicator when the product went live
at 50 Gbps and revealed that one lcore was occasionally taking too long
in the Hyperscan path.

---

## Testing without a physical NIC

```c
/* Add to eal_args in main(): */
"--vdev", "net_null0,copy=1",
```

`net_null` is a virtual NIC PMD that accepts TX packets and returns empty
RX bursts. `copy=1` makes TX actually copy data (catches buffer access bugs).
The entire `port_init()` code runs identically — useful for CI/CD.

---

## Queue-to-lcore assignment (set in Module 08)

```
After port_init():

  RX queue 0  →  worker lcore 3  (polls queue 0 in its RX loop)
  RX queue 1  →  worker lcore 4
  RX queue 2  →  worker lcore 5
  RX queue 3  →  worker lcore 6

  TX queue 0  →  TX lcore 2 (all workers enqueue to tx_ring; TX lcore drains it)

Inside each worker lcore function:
  nb_rx = rte_eth_rx_burst(port_id, lcore_info->queue_id, mbufs, BURST_SIZE);
```

---

## Next module

**Module 11 — Multi-lcore RX/TX Pipeline**: Wire together everything from
Modules 08–10: EAL init → mempool → port init → launch RX lcore
(polls NIC → ring) → worker lcores (dequeue → parse → policy) → TX lcore
(dequeue → NIC TX burst). The full packet pipeline skeleton.
