# DPDK Dataplane Learning

A progressive, hands-on C curriculum for learning how a high-performance
DPDK-based packet processing application is built — from a bare config parser
up to a complete multi-lcore pipeline with Hyperscan, Kafka, and DNS policy
enforcement.

Each module is **self-contained**: it has its own `Makefile`, source files,
and `README.md` with key concepts, expected output, and how the code maps
to a real production dataplane.

---

## Module Map

| # | Module | Topic | Pure C | Needs DPDK | Needs Hyperscan | Needs Kafka |
|---|--------|--------|:------:|:----------:|:---------------:|:-----------:|
| [01](#01-config-parser) | Config Parser | INI config → `config_t` struct | ✓ | | | |
| [02](#02-logger) | Logger | Thread-safe, level-filtered logger | ✓ | | | |
| [03](#03-ring-buffer) | Ring Buffer | SPSC lock-free ring (rte_ring primer) | ✓ | | | |
| [04](#04-hash-map) | Hash Map | Open-addressing map (rte_hash primer) | ✓ | | | |
| [05](#05-packet-structs) | Packet Structs | ETH / IPv4 / IPv6 / UDP / TCP / DNS headers | ✓ | | | |
| [06](#06-dns-parser) | DNS Parser | Full DNS query/response wire-format parser | ✓ | | | |
| [07](#07-tls-sni) | TLS SNI Extractor | ClientHello walker + Hyperscan-style SNI extract | ✓ | | | |
| [08](#08-dpdk-eal-init) | DPDK EAL Init | rte_eal_init, lcore topology, role assignment | | ✓ | | |
| [09](#09-mempool-mbuf) | Mempool + mbuf | rte_mempool, mbuf lifecycle, in-place rewrite | | ✓ | | |
| [10](#10-port-init) | Port Init | NIC configure, RSS, descriptor rings, link-up | | ✓ | | |
| [11](#11-pipeline) | Multi-lcore Pipeline | RX → ring → worker → ring → TX skeleton | | ✓ | | |
| [12](#12-rte-hash) | rte_hash CRUD | Insert, lookup, bulk lookup, delete, iterate | | ✓ | | |
| [13](#13-atomic-stats) | Atomic Stats | Per-lcore counters, false sharing, rate calc | ✓ | | | |
| [14](#14-numa-alloc) | NUMA Allocation | rte_malloc_socket, memzone, NUMA topology | | ✓ | | |
| [15](#15-hyperscan-compile) | Hyperscan Compile | hs_compile_multi, hs_compile_lit_multi, parseFile | | | ✓ | |
| [16](#16-hyperscan-scan) | Hyperscan Scan | Scratch, hs_clone_scratch, hs_scan, on_hs_match | | | ✓ | |
| [17](#17-policy-lookup) | Two-tier Policy | rte_hash exact match + Hyperscan fallback | | | ✓ | |
| [18](#18-dns-sinkhole) | DNS Sinkhole | In-place mbuf rewrite → walled-garden response | ✓ | | | |
| [19](#19-kafka-producer) | Kafka Producer | CDR export via librdkafka | | | | ✓ |
| [20](#20-kafka-consumer) | Kafka Consumer | Policy sync + SYNC_COMPLETE atomic swap | | | | ✓ |
| [21](#21-full-pipeline) | Full Pipeline | Annotated assembly of all 20 modules | ✓ | | | |

---

## Prerequisites

### For pure-C modules (01–07, 13, 18, 21)
```bash
# Any modern Linux with gcc
gcc --version
make --version
```

### For DPDK modules (08–12, 14)
```bash
# RedHat / Rocky Linux 8+
dnf install dpdk dpdk-devel

# Ubuntu 22.04+
apt-get install dpdk-dev

# Hugepages (1 GB)
echo 512 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

### For Hyperscan modules (15–17)
```bash
# RedHat / Rocky
dnf install hyperscan hyperscan-devel

# Ubuntu
apt-get install libhyperscan-dev
```
> Requires SSE4.2: `grep -m1 sse4_2 /proc/cpuinfo`

### For Kafka modules (19–20)
```bash
# RedHat / Rocky
dnf install librdkafka librdkafka-devel

# Ubuntu
apt-get install librdkafka-dev

# Test broker (Docker)
docker run -d -p 9092:9092 apache/kafka:3.7.0
```

---

## Learning Paths

### Path A — Pure C foundations (no extra deps)
Work through these in order on any Linux machine:

```
01 → 02 → 03 → 04 → 05 → 06 → 07 → 13 → 18 → 21
```

### Path B — Full DPDK pipeline
Complete Path A first, then:

```
08 → 09 → 10 → 11 → 12 → 14 → 15 → 16 → 17 → 19 → 20 → 21
```

### Path C — Policy engine only (Hyperscan, no DPDK)
```
04 → 15 → 16 → 17
```

---

## Module Reference

---

### 01 — Config Parser

**[`01-config-parser/`](01-config-parser/)**

Parses an INI-style config file into a `config_t` struct. Every subsystem
(EAL, ports, Kafka, Hyperscan) reads its startup values from this one
parsed structure.

**Concepts:** parse-once/read-many, section+key lookup, default fallback, inline comment stripping, type coercion at read time.

```bash
cd 01-config-parser && make && ./config_parser
```

---

### 02 — Logger

**[`02-logger/`](02-logger/)**

Thread-safe, level-filtered logger with millisecond timestamps, dual output
(stderr with ANSI colours + plain log file), and `LOG_DEBUG/INFO/WARN/ERROR/FATAL` macros.

**Concepts:** `pthread_mutex`, `vsnprintf` outside the lock, `fflush` after every write, `abort()` on FATAL, `__attribute__((format(printf)))`.

```bash
cd 02-logger && make && ./logger
```

---

### 03 — Ring Buffer

**[`03-ring-buffer/`](03-ring-buffer/)**

Lock-free SPSC ring buffer — the manual equivalent of `rte_ring`. Same three
ideas as DPDK: power-of-2 size + bitmask, separate head/tail on distinct cache
lines, acquire/release barriers.

**Concepts:** bitmask vs modulo, sentinel slot, `__ATOMIC_RELEASE`/`ACQUIRE`, false sharing, DPDK API mapping.

```bash
cd 03-ring-buffer && make && ./ring
```

---

### 04 — Hash Map

**[`04-hash-map/`](04-hash-map/)**

Open-addressing hash map with linear probing — the manual equivalent of
`rte_hash`. Teaches the tombstone delete pattern, load-factor discipline,
and why `rte_hash` uses cuckoo hashing with SIMD key comparison.

**Concepts:** FNV-1a vs CRC32, tombstone vs empty slot, 75% load limit, DPDK API mapping.

```bash
cd 04-hash-map && make && ./hashmap
```

---

### 05 — Packet Structs

**[`05-packet-structs/`](05-packet-structs/)**

Defines `eth_hdr_t`, `ipv4_hdr_t`, `ipv6_hdr_t`, `udp_hdr_t`, `tcp_hdr_t`,
`dns_hdr_t` in C and parses raw bytes into them via zero-copy pointer overlay.

**Concepts:** `__attribute__((packed))`, network byte order, `ntohs`/`ntohl`, variable-length IPv4/TCP headers, DNS qname wire format.

```bash
cd 05-packet-structs && make && ./packet_parser
```

---

### 06 — DNS Parser

**[`06-dns-parser/`](06-dns-parser/)**

Full DNS message parser: queries and responses, UDP and TCP, pointer
compression (RFC 1035 §4.1.4), CNAME chains, and case normalization.

**Concepts:** label encoding, `0xC0xx` pointer decompression, 2-byte TCP length prefix, `question_wire_end` offset for sinkhole (Module 18).

```bash
cd 06-dns-parser && make && ./dns_parser
```

---

### 07 — TLS SNI Extractor

**[`07-tls-sni/`](07-tls-sni/)**

Two approaches: (A) full ClientHello walker, (B) fixed-offset extraction
from a Hyperscan match — the exact technique used in the packet hot path.

**Concepts:** TLS record hierarchy, 24-bit handshake length, SNI extension layout, `from+7`/`from+9` offsets, TLS 1.3 SNI still in plaintext.

```bash
cd 07-tls-sni && make && ./tls_sni
```

---

### 08 — DPDK EAL Init

**[`08-dpdk-eal-init/`](08-dpdk-eal-init/)** *(requires DPDK)*

Initializes the DPDK Environment Abstraction Layer: builds `argv[]` from
config, calls `rte_eal_init`, assigns lcore roles (RX / TX / WORKER),
launches per-lcore functions, and shuts down cleanly.

**Concepts:** hugepage locking, CPU affinity, `rte_eal_remote_launch` vs `pthread_create`, `rte_pause()`, graceful shutdown ordering.

```bash
cd 08-dpdk-eal-init && make && sudo ./eal_init
```

---

### 09 — Mempool + mbuf

**[`09-mempool-mbuf/`](09-mempool-mbuf/)** *(requires DPDK)*

Covers `rte_mempool` pre-allocation, the mbuf memory layout
(headroom / data / tailroom), `rte_pktmbuf_mtod`, `rte_pktmbuf_append`
(used by the DNS sinkhole), and hardware checksum offload flags.

**Concepts:** why mempools (no malloc in fast path), `2^k - 1` pool size rule, per-lcore cache, `rte_pktmbuf_append` for answer sections, `ol_flags`.

```bash
cd 09-mempool-mbuf && make && sudo ./mempool_mbuf
```

---

### 10 — Port Init

**[`10-port-init/`](10-port-init/)** *(requires DPDK + NIC)*

Configures a DPDK NIC port end-to-end: capability intersection, RSS for
multi-queue packet distribution, descriptor ring depth, promiscuous mode,
link-state polling.

**Concepts:** offload capability masking, RSS hash types, `rte_eth_dev_adjust_nb_rx_tx_desc`, `stats.imissed`, `--vdev net_null0` for CI.

```bash
cd 10-port-init && make && sudo ./port_init
```

---

### 11 — Multi-lcore Pipeline

**[`11-pipeline/`](11-pipeline/)** *(requires DPDK)*

Wires EAL + mempool + port into a running RX→worker→TX pipeline:
RX lcore polls the NIC, round-robins into per-worker rings, workers
parse and apply policy stubs, TX lcore drains to the NIC.

**Concepts:** dedicated lcore roles, SPSC vs MPSC rings, `rte_pause()` not `sleep()`, free unsent TX mbufs, shutdown drain ordering.

```bash
cd 11-pipeline && make && sudo ./pipeline
```

---

### 12 — rte_hash CRUD

**[`12-rte-hash/`](12-rte-hash/)** *(requires DPDK)*

Covers the complete `rte_hash` API: create, insert, single lookup, **bulk
lookup** (the critical performance pattern at line rate), delete, iterate,
and integer-key tables.

**Concepts:** `key_len` zero-init bug, bulk lookup prefetch mechanics, `rte_hash_crc` vs `rte_jhash`, NUMA socket placement, `RW_CONCURRENCY` vs RCU QSBR.

```bash
cd 12-rte-hash && make && sudo ./rte_hash_crud
```

---

### 13 — Atomic Stats

**[`13-atomic-stats/`](13-atomic-stats/)**

Lock-free per-lcore statistics with C11 `_Atomic` / `stdatomic.h`.
Includes a measured false-sharing demo (4× slowdown) and atomic vs mutex
benchmark (4.6× slower).

**Concepts:** `memory_order_relaxed` for counters, `aligned(64)` to prevent false sharing, `aligned_alloc` for arrays, per-lcore vs global atomics.

```bash
cd 13-atomic-stats && make && ./atomic_stats
```

---

### 14 — NUMA-aware Allocation

**[`14-numa-alloc/`](14-numa-alloc/)** *(requires DPDK)*

Reads `/sys` NUMA topology, demonstrates `rte_zmalloc_socket`,
`rte_memzone_reserve`, and the Hyperscan custom allocator pattern that
redirects `hs_compile` internal malloc to hugepage memory on the correct
NUMA socket.

**Concepts:** ~90 ns cross-NUMA penalty, allocation hierarchy (memzone → malloc heap), `rte_eth_dev_socket_id` for NIC-local pools, `hs_set_allocator`.

```bash
cd 14-numa-alloc && make && ./numa_alloc      # Part A: topology reader (no DPDK)
cd 14-numa-alloc && make dpdk && sudo ./numa_alloc_dpdk  # Part B: DPDK APIs
```

---

### 15 — Hyperscan Compile

**[`15-hyperscan-compile/`](15-hyperscan-compile/)** *(requires Hyperscan)*

Compiles both a global regex database (`hs_compile_multi`) and a per-group
literal database (`hs_compile_lit_multi`). Includes `parseFlags`, `parseFile`,
`hs_create_db`, and database serialization for fast restart.

**Concepts:** regex vs literal mode, `HS_FLAG_SINGLEMATCH` for performance, pattern ID dispatch, `HS_MODE_BLOCK`, serialize/deserialize for ~500 ms → 5 ms restart.

```bash
cd 15-hyperscan-compile && make && ./hs_compile
```

---

### 16 — Hyperscan Scan

**[`16-hyperscan-scan/`](16-hyperscan-scan/)** *(requires Hyperscan)*

Full scan pipeline: `hs_alloc_scratch`, `hs_clone_scratch` per lcore,
`hs_scan` with `on_hs_match` and `on_hs_match_group` callbacks,
TLS SNI extraction from match offsets, and thread-safety demo.

**Concepts:** scratch is not thread-safe (one clone per lcore), `hs_alloc_scratch` grows an existing scratch, `HS_SCAN_TERMINATED` is not an error, `from`/`to` offsets.

```bash
cd 16-hyperscan-scan && make && ./hs_scan
```

---

### 17 — Two-tier Policy Lookup

**[`17-policy-lookup/`](17-policy-lookup/)** *(requires Hyperscan)*

The complete policy engine: malicious domain pre-check →
Tier 1 `rte_hash` exact match → Tier 2 Hyperscan literal fallback →
`filter_details` application → `ALLOW / DROP / SINKHOLE` decision.

**Concepts:** why two tiers (90% hash hit at ~50 ns vs 10% Hyperscan at ~5 µs), decision hierarchy (whitelist wins), multi-group evaluation, malicious table bypasses whitelists.

```bash
cd 17-policy-lookup && make && ./policy_lookup
```

---

### 18 — DNS Sinkhole

**[`18-dns-sinkhole/`](18-dns-sinkhole/)**

Rewrites a DNS query packet **in-place** into a sinkhole response with a
walled-garden IP — no new allocation, ~30 ns vs ~220 ns for alloc+copy.
Covers UDP/IPv4, UDP/IPv6, and TCP/IPv4 variants.

**Concepts:** header swap without temp bug, DNS answer section layout, `0xC00C` pointer compression, `question_wire_end` append offset, length field cascade (UDP + IP + TCP prefix).

```bash
cd 18-dns-sinkhole && make && ./dns_sinkhole
```

---

### 19 — Kafka Producer

**[`19-kafka-producer/`](19-kafka-producer/)** *(requires librdkafka)*

CDR (Charging Data Record) export via `librdkafka`: delivery report callback,
`rd_kafka_poll()` discipline, CDR batching with timer-based flush,
subscriber IP as partition key, back-pressure handling, graceful flush on shutdown.

**Concepts:** `produce()` ≠ broker receipt (only delivery callback confirms), `QUEUE_FULL` back-pressure, `MSG_F_COPY` vs `MSG_F_FREE`, `rd_kafka_flush()` before destroy.

```bash
cd 19-kafka-producer && make
./kafka_producer localhost:9092 dp_cdr
```

---

### 20 — Kafka Consumer

**[`20-kafka-consumer/`](20-kafka-consumer/)** *(requires librdkafka)*

Policy sync consumer: `BEGIN_SYNC → ADD_DOMAIN* → SYNC_COMPLETE` atomic
swap protocol, manual offset commit only at `SYNC_COMPLETE`, partition
rebalance callback, and the RCU QSBR write-side pattern for lock-free
reader/writer policy updates.

**Concepts:** why `auto.offset.reset=earliest` (full replay on restart), `enable.auto.commit=false` for transactional correctness, atomic pointer swap + RCU quiesce before free, `rd_kafka_consumer_close()` for graceful leave.

```bash
cd 20-kafka-consumer && make
./kafka_consumer --demo                          # no broker needed
./kafka_consumer localhost:9092 policy_updates   # live broker
```

---

### 21 — Full Pipeline

**[`21-full-pipeline/`](21-full-pipeline/)**

A single annotated C file that assembles all 20 modules into one complete
dataplane application. Read top-to-bottom to see every module connection —
from `main()` startup through the packet hot path to graceful shutdown.
Runs standalone (no DPDK/Hyperscan/Kafka needed).

**Covers:** exact startup order (why each step must precede the next), shutdown drain order (TX must stop last), complete packet hot-path call graph with module annotations.

```bash
cd 21-full-pipeline && make && ./full_pipeline
```

---

## Module Dependency Map

```
21 (Full Pipeline)
│
├── 01  config_load()
├── 02  logger_init()
│
├── 08  rte_eal_init()
│    └── 14  NUMA socket allocation
│
├── 09  rte_pktmbuf_pool_create()
├── 10  port_init()
│
├── 03  ring_create()   (or rte_ring)
├── 04  hashmap         (or rte_hash)
├── 12  rte_hash CRUD
│
├── 15  hs_create_db()
├── 16  hs_init_global_scratch() + hs_clone_scratch_for_lcore()
│
├── 19  kafka_producer_init()
├── 20  kafka_consumer_init()
│
└── PER-PACKET HOT PATH (worker lcore)
     ├── 05  ETH/IP/UDP-TCP parse
     ├── 06  dns_parse_message()
     ├── 07  tls_extract_sni_from_match()
     ├── 13  atomic counter increments
     ├── 16  hs_scan_payload() + hs_scan_domain_group()
     ├── 17  url_policy_for_dns()
     ├── 18  dns_build_sinkhole_v4/v6()
     └── 19  cdr_batch_add()
```

---

## Repository Structure

```
dataplane-learning/
├── 01-config-parser/       config_parser.c  sample.conf  Makefile  README.md
├── 02-logger/              logger.c  logger.h  Makefile  README.md
├── 03-ring-buffer/         ring.c  ring.h  Makefile  README.md
├── 04-hash-map/            hashmap.c  hashmap.h  Makefile  README.md
├── 05-packet-structs/      packet_parser.c  packet_structs.h  Makefile  README.md
├── 06-dns-parser/          dns_parser.c  dns_parser.h  Makefile  README.md
├── 07-tls-sni/             tls_sni.c  tls_sni.h  Makefile  README.md
├── 08-dpdk-eal-init/       eal_init.c  Makefile  README.md
├── 09-mempool-mbuf/        mempool_mbuf.c  Makefile  README.md
├── 10-port-init/           port_init.c  Makefile  README.md
├── 11-pipeline/            pipeline.c  Makefile  README.md
├── 12-rte-hash/            rte_hash_crud.c  Makefile  README.md
├── 13-atomic-stats/        atomic_stats.c  Makefile  README.md
├── 14-numa-alloc/          numa_alloc.c  Makefile  README.md
├── 15-hyperscan-compile/   hs_compile.c  sample_patterns.txt  Makefile  README.md
├── 16-hyperscan-scan/      hs_scan.c  Makefile  README.md
├── 17-policy-lookup/       policy_lookup.c  Makefile  README.md
├── 18-dns-sinkhole/        dns_sinkhole.c  Makefile  README.md
├── 19-kafka-producer/      kafka_producer.c  Makefile  README.md
├── 20-kafka-consumer/      kafka_consumer.c  Makefile  README.md
└── 21-full-pipeline/       full_pipeline.c  Makefile  README.md
```

---

## Quick Start (pure-C modules, zero extra deps)

```bash
git clone https://github.com/Ajay3007/dataplane-learning.git
cd dataplane-learning

# Run any standalone module
for dir in 01-config-parser 02-logger 03-ring-buffer 04-hash-map \
           05-packet-structs 06-dns-parser 07-tls-sni \
           13-atomic-stats 18-dns-sinkhole 21-full-pipeline; do
    echo "=== $dir ===" && cd $dir && make -s && ./* && cd ..
done
```

---

## License

MIT
