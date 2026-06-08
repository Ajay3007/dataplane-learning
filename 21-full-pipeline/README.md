# Module 21 — Full Pipeline (Annotated Assembly)

> Runs standalone with `make && ./full_pipeline` (no DPDK/HS/Kafka needed).  
> Every production API call is annotated with its module number.

## What this module is

A single annotated C file that assembles all 20 prior modules into one
complete dataplane application. Read it top-to-bottom to see exactly
how a production DPDK URL filtering engine is structured — from `main()`
startup through the packet hot path to graceful shutdown.

---

## Build and run

```bash
make
./full_pipeline
```

Expected output:
```
=== Module 21: Full Pipeline ===
Simulating 1000 packets across 2 workers

[1] Config loaded
[2] Logger initialized
[3] EAL initialized (simulation: using pthreads)
...
[11] Policy tables seeded: 3 domains, 2 malicious
[12] Inter-lcore rings created
[13] Launching lcores...
[14] Main lcore entering control loop

[lcore 0] RX started
[lcore 3] Worker started
[lcore 4] Worker started
[lcore 1] TX started
[Stats] dns=180  sinkhole=62  drop=0
[Main] Simulating Kafka policy update (SYNC_COMPLETE)...
[Main] Policy updated: new-blocked.com → blacklist
...
[Shutdown] Signalling lcores...

[Final Statistics]
  rx=1000  tx=762  drop=238
  dns=800  tls=200  sinkholes=238
  cdr_records=1000
```

---

## Full system architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     DP Application                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  lcore 0: MAIN CONTROL LOOP                             │    │
│  │                                                          │    │
│  │  rd_kafka_consumer_poll()     ← Module 20               │    │
│  │    apply_policy_message()                                │    │
│  │      BEGIN_SYNC → pending table                          │    │
│  │      ADD_DOMAIN → pending hash insert                    │    │
│  │      SYNC_COMPLETE → atomic_swap + RCU synchronize       │    │
│  │                                                          │    │
│  │  rd_kafka_poll(cdr_producer)  ← Module 19                │    │
│  │  cdr_batch_flush_if_timeout() ← Module 19                │    │
│  │  print_stats()                ← Module 13                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────┐  rings  ┌──────────────────┐  ring  ┌──────┐  │
│  │ lcore 1: RX  │ ──────► │ lcores 3-6:      │ ──────►│ TX   │  │
│  │              │         │ WORKERS           │        │      │  │
│  │ rte_eth_     │         │                   │        │rte_  │  │
│  │ rx_burst()   │         │ PACKET HOT PATH:  │        │eth_  │  │
│  │ Module 10    │         │ parse ETH/IP/UDP  │        │tx_   │  │
│  │              │         │ Module 05         │        │burst │  │
│  │ distribute   │         │                   │        │Mod10 │  │
│  │ round-robin  │         │ DNS: dns_parse()  │        │      │  │
│  │ Module 11    │         │ Module 06         │        │free  │  │
│  └──────────────┘         │                   │        │unsent│  │
│                            │ TLS: hs_scan()    │        │Mod 09│  │
│                            │ Module 16         │        └──────┘  │
│                            │                   │                  │
│                            │ policy_lookup()   │                  │
│                            │ Module 17:        │                  │
│                            │  check_malicious()│                  │
│                            │  rte_hash_lookup  │                  │
│                            │  hs_scan_group()  │                  │
│                            │                   │                  │
│                            │ if SINKHOLE:      │                  │
│                            │  dns_sinkhole()   │                  │
│                            │  Module 18        │                  │
│                            │                   │                  │
│                            │ cdr_batch_add()   │                  │
│                            │ Module 19         │                  │
│                            └──────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
        │ CDR (Module 19)              ▲ Policy updates (Module 20)
        ▼                              │
   ┌──────────┐                 ┌──────────────────┐
   │  Kafka   │                 │  Provisioning    │
   │  Broker  │                 │  Module          │
   │          │                 │                  │
   └──────────┘                 └──────────────────┘
```

---

## Complete module dependency map

```
Module 21 (Full Pipeline)
  │
  ├── Module 01  config_load()
  ├── Module 02  logger_init()
  │
  ├── Module 08  rte_eal_init() + lcore launch
  │    └── Module 14  NUMA socket allocation
  │
  ├── Module 09  rte_pktmbuf_pool_create()
  ├── Module 10  port_init() → rte_eth_dev_configure/start
  │
  ├── Module 03  ring_create() (replaces rte_ring)
  ├── Module 04  hashmap (replaces rte_hash)
  ├── Module 12  rte_hash CRUD (production domain_details_table)
  │
  ├── Module 15  hs_create_db() → domainsPatternDB
  ├── Module 16  hs_init_global_scratch() + hs_clone_scratch_for_lcore()
  │
  ├── Module 19  kafka_producer_init() → CDR export
  ├── Module 20  kafka_consumer_init() → policy sync
  │
  └── PER-PACKET HOT PATH (worker lcore):
       ├── Module 05  ETH/IP/UDP-TCP header parse
       ├── Module 06  dns_parse_message()
       ├── Module 07  tls_extract_sni_from_match()
       ├── Module 13  atomic counter increments
       ├── Module 16  hs_scan_payload() + hs_scan_domain_group()
       ├── Module 17  url_policy_for_dns()
       ├── Module 18  dns_build_sinkhole_v4/dns_build_sinkhole_v6()
       └── Module 19  cdr_batch_add()
```

---

## Startup sequence (exact order, production)

```
Step  What                          Module  Why order matters
----  ---                           ------  -----------------
 1    config_load()                  01     Everything reads from config
 2    logger_init()                  02     All following steps log errors
 3    rte_eal_init()                 08     Hugepages, CPU affinity, NIC probe
 4    rte_pktmbuf_pool_create()      09     NIC needs pool for RX descriptors
 5    port_init()                    10     Needs pool from step 4
 6    NUMA alloc: hash tables        12/14  Must know socket from step 3
 7    hs_create_db()                 15     Patterns loaded, DB compiled
 8    hs_init_global_scratch()       16     Needs DB from step 7
 9    kafka_producer_init()          19     CDR topic ready before workers
10    kafka_consumer_init()          20     Read initial policy before workers
11    Seed policy tables             12/20  Workers need policies from day 1
12    rte_ring_create()              03/11  Workers need rings before launch
13    rte_eal_remote_launch()        08/11  TX first, then workers, then RX
14    main control loop              19/20  Poll Kafka, flush CDR, print stats
```

**Step 13 launch order (TX → workers → RX) is critical:**
- If RX launches before workers: packets pile up in rings until workers start
- If TX launches after workers: workers can't enqueue (tx_ring is NULL)
- Correct order ensures pipeline is ready before packets flow

---

## Shutdown sequence (exact order, production)

```
SIGTERM received → g_shutdown = 1

1. Stop RX lcore first
   → RX stops generating/receiving packets
   → Existing packets in rx_rings continue to drain

2. Wait for RX lcore (rte_eal_wait_lcore(rx_id))

3. Stop worker lcores (set running = 0)
   → Workers finish processing in-flight packets
   → CDR batches flushed before exit

4. Wait for workers (rte_eal_wait_lcore(worker_id))

5. Stop TX lcore LAST
   → TX drains remaining packets from tx_ring before exiting

6. Wait for TX (rte_eal_wait_lcore(tx_id))

7. Kafka flush (Module 19):
   rd_kafka_flush(rk_producer, 5000)   ← drain CDR queue
   rd_kafka_consumer_close(rk_consumer)

8. Hyperscan cleanup (Module 15/16):
   hs_free_scratch() per lcore
   hs_free_database() per group + global

9. rte_eth_dev_stop() + rte_eth_dev_close()  ← Module 10

10. rte_eal_cleanup()  ← Module 08
```

**Step 5 (TX last) is critical:**
Workers may have enqueued a final burst after getting the stop signal.
If TX stops first, those packets are stuck in the ring and freed without
being transmitted — the sinkhole response never reaches the client.

---

## The packet hot path — annotated call graph

```
worker_lcore_func():
  │
  ring_dequeue_burst(rx_ring)                ← Module 03
  │
  for each mbuf:
    │
    ├── [nb_segs > 1 guard]                  ← Module 09
    │
    ├── ETH parse: rte_pktmbuf_mtod()        ← Module 05/09
    │   IP/UDP/TCP overlay
    │
    ├── if UDP dst_port == 53: DNS path
    │    │
    │    ├── dns_parse_message()             ← Module 06
    │    │   → domain, qtype, question_end
    │    │
    │    ├── rte_hash_lookup(subscriber_table)     ← Module 12
    │    │   → subscriber + group_id
    │    │
    │    └── url_policy_for_dns() ← Module 17
    │         │
    │         ├── check_malicious()          ← Module 12/20
    │         │    rte_hash_lookup(malicious)
    │         │    → PROCESS_WORKFLOW if found
    │         │
    │         └── fetch_url_policy()
    │              │
    │              ├── Tier 1: rte_hash_lookup_data()  ← Module 12
    │              │    HIT → apply_filter_details()   ← Module 17
    │              │
    │              └── Tier 2: hs_scan_domain_group() ← Module 16
    │                   HIT → apply_filter_details()
    │
    ├── if PROCESS_WORKFLOW (sinkhole):
    │    dns_build_sinkhole_v4()  ← Module 18
    │    → swap headers, inject A/AAAA answer
    │    → rte_pktmbuf_append(mbuf, answer_len)
    │    → m->ol_flags |= TX_IP_CKSUM | TX_UDP_CKSUM  ← Module 09
    │
    ├── if TCP dst_port == 443: TLS path
    │    │
    │    ├── hs_scan_payload(payload, scratch) ← Module 16
    │    │   → on_hs_match fires id=1 (TLS)
    │    │   → SNI at from+7/from+9              ← Module 07
    │    │
    │    └── url_policy_for_tls()   ← Module 17
    │
    ├── cdr_batch_add(&cdr, domain, decision)    ← Module 19
    │   → rd_kafka_produce() when batch full
    │   → rd_kafka_poll(producer, 0)
    │
    ├── if DROP: rte_pktmbuf_free(mbuf)          ← Module 09
    └── if ALLOW/SINKHOLE: ring_enqueue(tx_ring) ← Module 03
```

---

## Learning path recap

| Module | Topic | Dependencies |
|---|---|---|
| 01 | Config parser | none |
| 02 | Logger | none |
| 03 | Ring buffer | none |
| 04 | Hash map | none |
| 05 | Packet structs | none |
| 06 | DNS parser | 05 |
| 07 | TLS SNI extractor | 05 |
| 08 | DPDK EAL init | 01, 02 |
| 09 | Mempool + mbuf | 08 |
| 10 | Port init | 08, 09 |
| 11 | Multi-lcore pipeline | 08, 09, 10, 03 |
| 12 | rte_hash CRUD | 08 |
| 13 | Atomic stats | none |
| 14 | NUMA alloc | 08 |
| 15 | Hyperscan compile | none |
| 16 | Hyperscan scan | 15 |
| 17 | Two-tier policy | 12, 16 |
| 18 | DNS sinkhole | 05, 06 |
| 19 | Kafka producer | none |
| 20 | Kafka consumer | 19, 17 |
| **21** | **Full pipeline** | **all** |
