# Module 12 — rte_hash CRUD

> **Reference code** — requires DPDK installed.

## What you learn

The complete `rte_hash` API — DPDK's production hash table used for every
O(1) policy lookup in the DP application. This module covers create, insert, single
lookup, bulk lookup, delete, iterate, and integer key tables, with a
performance comparison showing why bulk lookup is critical at 2M packets/sec.

---

## Hash tables in the real DP project

| Table | Key type | Value type | Size | Used in |
|---|---|---|---|---|
| `domain_details_table` | domain string (256B) | `filter_details*` | per group | DNS/TLS policy hot path |
| `ip4_vs_subscriber_table` | `uint32_t` IPv4 | subscriber struct | 500K | subscriber resolution |
| `connection_track_table` | connection tuple | connection state | 2M | TCP tracking |
| `tls_session_table` | connection tuple | TLS state | 2M | SNI extraction |
| `domain_sig_table` | URL string | signature ID | large | Hyperscan sig map |
| `malicious_domain_table` | domain string | block context | dynamic | malicious lookup |

All created with `rte_hash_crc` (CRC32 hardware instruction) and
`socket_id = rte_socket_id()` for NUMA-local allocation.

---

## Where this fits in the real application

```
Kafka policy update received (main lcore):
  add_domain_to_group(group, domain, &filter_details)
    → rte_hash_add_key_data(group->domain_details_table, key, fd)

Worker lcore — DNS packet arrives:
  dns_parse_message() → domain = "blocked-malware.example.com"
  │
  ├─► rte_hash_lookup_data(domain_details_table, domain, &fd)
  │     HIT  → apply fd policy (ALLOW/DROP/SINKHOLE)
  │     MISS → hs_scan_domain_group()   (Module 22)
  │
  └─► if SINKHOLE → dns_build_sinkhole_v4()  (Module 23)

Kafka policy revoke (main lcore):
  rte_hash_del_key(group->domain_details_table, domain)
```

---

## Files

| File | Purpose |
|---|---|
| `rte_hash_crud.c` | 4 demos: CRUD, bulk vs single perf, iterate, integer key |
| `Makefile` | DPDK build |

---

## Key concepts in the code

### 1. `key_len` — the most common rte_hash bug

```c
params.key_len = MAX_URL_LEN;  /* 256 bytes */

/* WRONG: only the string bytes are valid; tail is garbage */
char key[MAX_URL_LEN];
strcpy(key, "google.com");
rte_hash_add_key_data(tbl, key, data);   /* compares all 256 bytes! */

/* CORRECT: zero the entire key buffer first */
char key[MAX_URL_LEN];
memset(key, 0, MAX_URL_LEN);
strncpy(key, "google.com", MAX_URL_LEN - 1);
rte_hash_add_key_data(tbl, key, data);
```

`rte_hash` compares exactly `key_len` bytes using CRC32. If the buffer
beyond the string content contains random stack garbage, `add` and
`lookup` will hash different values — the lookup always misses even though
the key "matches". This is silent and extremely hard to debug.

In the real app, the domain buffer is always zero-initialized before
being passed to any hash operation.

### 2. Bulk lookup — why it matters at line rate

```
Single lookup at 2M DNS/sec:
  Each domain is a cache miss (~100 ns for DRAM access)
  2M × 100 ns = 200 ms of CPU stall per second → only 1 core at 20% utilisation

Bulk lookup (LOOKUP_BURST=32):
  CPU issues all 32 DRAM prefetches simultaneously
  They overlap in the memory controller
  Total time ≈ 1 DRAM access for all 32
  2M / 32 × 100 ns = 6.25 ms of CPU stall per second → 32× reduction
```

```c
/* Hot path — bulk lookup instead of single in the worker loop: */
const void *keys[BURST];
void       *data[BURST];
uint64_t    hit_mask;

/* Fill keys[] from parsed domains */
rte_hash_lookup_bulk_data(tbl, keys, BURST, &hit_mask, data);

for (int i = 0; i < BURST; i++) {
    if (hit_mask & (1ULL << i)) {
        filter_details_t *fd = data[i];
        /* apply policy */
    } else {
        /* miss: fall through to Hyperscan */
    }
}
```

In `fetch_url_policy_for_domain()` in the real app, this pattern
is used when processing a burst of DNS packets from a worker's rx_ring.

### 3. `add_key_data` return value — position, not just success/fail

```c
int pos = rte_hash_add_key_data(tbl, key, data);
/* pos >= 0: success, pos is the internal slot index */
/* pos == -ENOSPC: table full */
/* pos == -EINVAL: key is NULL */
```

The position can be used to directly index a parallel data array — an
alternative pattern where data is stored separately and the position is
the index. The DP application uses the `data` pointer pattern instead, but both are valid.

### 4. `rte_hash_crc` — why always use this

```c
/* Always specify this as hash_func: */
.hash_func = rte_hash_crc,

/* NOT: */
.hash_func = rte_jhash,    /* slower: software Jenkin's hash */
.hash_func = NULL,         /* uses rte_jhash by default — still slower */
```

`rte_hash_crc` calls the x86 `CRC32` hardware instruction. It processes
8 bytes per clock cycle. For a 256-byte domain key: ~32 cycles.
`rte_jhash` processes ~1 byte per cycle for 256 bytes. The CRC32
instruction is enabled by default on all x86 CPUs since 2010 (SSE4.2).

### 5. NUMA socket placement

```c
.socket_id = (int)rte_socket_id(),
/* or for a specific NIC socket: */
.socket_id = (int)rte_eth_dev_socket_id(port_id),
```

A hash table on socket 0 queried from a lcore on socket 1:
- Each lookup crosses the QPI/UPI interconnect (~100 ns extra)
- At 2M lookups/sec: 200 ms/sec wasted on NUMA penalty

For tables queried from all worker lcores on the same socket, always
use `rte_socket_id()` of the socket hosting those lcores.

### 6. Thread safety

By default `rte_hash` is safe for concurrent readers (multiple lcores
calling `lookup_data` simultaneously). But concurrent reader + writer
(worker reads + main lcore writes during policy update) requires either:

```c
/* Option A: DPDK built-in RW concurrency (simpler): */
params.extra_flag = RTE_HASH_EXTRA_FLAGS_RW_CONCURRENCY;

/* Option B: RCU QSBR (what the DP application uses — implemented in the DP core library): */
/* Reader calls rte_rcu_qsbr_quiescent() in its loop */
/* Writer calls rte_rcu_qsbr_synchronize() before freeing old data */
```

Option A uses lock-free atomics inside rte_hash. Option B is more
flexible — it lets the writer wait for all readers to quiesce before
freeing the old data, enabling safe memory reclamation.

---

## rte_hash API quick-reference

```c
/* Create */
struct rte_hash_parameters p = {
    .name      = "my_table",
    .entries   = 65536,
    .key_len   = MAX_URL_LEN,   /* ALL keys must be exactly this length */
    .hash_func = rte_hash_crc,  /* always use CRC32 HW */
    .socket_id = rte_socket_id(),
};
struct rte_hash *tbl = rte_hash_create(&p);

/* Insert/update (upsert) */
int pos = rte_hash_add_key_data(tbl, key, data_ptr);
/* pos >= 0: ok | -ENOSPC: full | -EINVAL: null key */

/* Single lookup (hot path) */
void *data;
int pos = rte_hash_lookup_data(tbl, key, &data);
/* pos >= 0: hit, data valid | -ENOENT: miss */

/* Bulk lookup (preferred at burst processing) */
uint64_t hit_mask;
void *data_out[N];
rte_hash_lookup_bulk_data(tbl, keys, N, &hit_mask, data_out);
/* bit i of hit_mask set → data_out[i] is valid */

/* Delete */
int pos = rte_hash_del_key(tbl, key);
/* pos >= 0: deleted | -ENOENT: not found */

/* Iterate */
uint32_t iter = 0;
const void *k; void *d;
while (rte_hash_iterate(tbl, &k, &d, &iter) >= 0) { ... }

/* Count / destroy */
uint32_t n = rte_hash_count(tbl);
rte_hash_free(tbl);
```

---

## Next module

**Module 13 — Atomic Counters + Per-lcore Stats**: Per-lcore statistics
with `_Atomic` / `atomic_fetch_add` — the pattern used for all DPDK
performance counters (rx_count, tx_count, drop_count, dns_count). Includes
the lock-free stats aggregation pattern used in the main lcore control loop.
