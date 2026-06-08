# Module 14 — NUMA-aware Memory Allocation

**Part A** (NUMA topology reader) — pure C, runs on any Linux box.  
**Part B** (DPDK allocation APIs) — reference code, requires DPDK.

## What you learn

Why NUMA placement matters in a dataplane application (2× memory latency
penalty for cross-socket access), how to detect your server's NUMA topology,
and which DPDK allocation API to use for every type of dataplane object —
hash tables, mempools, Hyperscan databases, per-lcore scratch spaces.

---

## The NUMA penalty — why it matters at line rate

```
Single-socket server (all RAM local):
  Any lcore reads any address → ~80 ns

Dual-socket server (cross-NUMA access):
  Socket 0 lcore reads socket 1 memory → ~170 ns  (2× slower)
  Socket 0 lcore reads socket 0 memory → ~80 ns   (baseline)

At 2M DNS lookups/sec with domain_details_table on wrong socket:
  Extra latency = 2,000,000 × (170 - 80) ns = 180 ms/sec per worker lcore
  4 worker lcores: 720 ms/sec wasted — almost 1 full CPU core consumed
  by memory bus latency alone
```

This is why every hash table, pool, and database in the DP application specifies
`socket_id = rte_socket_id()` or `rte_eth_dev_socket_id(port_id)`.

---

## Build and run

```bash
# Part A only — reads /sys topology, no DPDK needed
make
./numa_alloc

# Part A + B — DPDK allocation demos
make dpdk
sudo ./numa_alloc_dpdk
```

Part A output on a dual-socket server:
```
=== Module 14: NUMA-aware Memory Allocation ===

  CPU topology (from /sys):
  cpu      physical_id  core_id      socket
  ---      -----------  -------      ------
  cpu0     0            0            0
  cpu1     0            1            0
  cpu2     0            2            0
  ...
  cpu20    1            0            1
  cpu21    1            1            1
  ...

  NUMA node memory (hugepages):
    node0: 256 total × 2MB = 512 MB  (256 MB free)
    node1: 256 total × 2MB = 512 MB  (256 MB free)
```

---

## Key concepts

### 1. DPDK allocation hierarchy

```
hugepage memory (pre-allocated, pinned, physically contiguous)
  │
  ├─ rte_memzone: named regions, persistent, shareable across processes
  │
  └─ rte_malloc heap: anonymous, per-socket pools
       ├─ rte_malloc_socket()   → explicit socket, not zero-initialised
       └─ rte_zmalloc_socket()  → explicit socket, zero-initialised (use this)
```

### 2. Which API for each DP application object

| Object | API | Socket |
|---|---|---|
| `group_struct` | `rte_zmalloc_socket` | worker lcore's socket |
| `filter_details` | `rte_zmalloc_socket` | worker lcore's socket |
| `rte_hash` (domain table) | `rte_hash_create` → `.socket_id` | worker lcore's socket |
| `rte_mempool` (mbufs) | `rte_pktmbuf_pool_create` → `socket_id` | NIC port's socket |
| Hyperscan DB | `hs_set_allocator` → `rte_malloc_socket` | worker lcores' socket |
| Hyperscan scratch | `hs_clone_scratch` | each lcore's own socket |
| `rte_ring` (rx/tx rings) | `rte_ring_create` → `socket_id` | worker lcores' socket |

### 3. Socket ID query APIs

```c
rte_socket_id()                    /* socket of the calling lcore */
rte_lcore_to_socket_id(lcore_id)   /* socket of a specific lcore */
rte_eth_dev_socket_id(port_id)     /* socket of a NIC port */
rte_socket_count()                  /* total NUMA sockets available */
```

**The NIC's socket is the most important one.** The NIC's DMA engine
reads mbufs and writes received packets. If the mbuf pool is on the
wrong socket, every packet receive and transmit crosses the QPI/UPI.

```c
/* CORRECT: pool on the same socket as the NIC */
int nic_sock = rte_eth_dev_socket_id(port_id);
pool = rte_pktmbuf_pool_create("pool", n, cache, 0,
                                RTE_MBUF_DEFAULT_BUF_SIZE,
                                nic_sock);               /* ← matches NIC */

/* WRONG: pool on SOCKET_ID_ANY */
pool = rte_pktmbuf_pool_create("pool", n, cache, 0,
                                RTE_MBUF_DEFAULT_BUF_SIZE,
                                rte_socket_id());  /* ← main lcore's socket,
                                                       may differ from NIC */
```

### 4. `rte_memzone` vs `rte_malloc_socket`

```c
/* rte_malloc_socket: anonymous, freed with rte_free() */
void *p = rte_zmalloc_socket("filter_details", size, align, sock);
rte_free(p);   /* explicitly freed when no longer needed */

/* rte_memzone: named, never freed (persists until EAL cleanup) */
const struct rte_memzone *mz = rte_memzone_reserve("global_cfg", size, sock, 0);
/* Access from anywhere: */
const struct rte_memzone *mz2 = rte_memzone_lookup("global_cfg");
assert(mz2->addr == mz->addr);
```

Use `rte_memzone` when:
- Data must be accessible by name from multiple modules (no pointer passing)
- A monitoring/secondary process needs to read the same memory
- Config or global state that never changes after init

### 5. Hyperscan NUMA allocation in the DP application

```c
/* Custom allocator wrappers — redirect Hyperscan's internal malloc to DPDK */
static void *hs_rte_malloc(size_t size) {
    return rte_malloc_socket("hs_internal", size, 0, rte_socket_id());
}
static void hs_rte_free(void *ptr) { rte_free(ptr); }

/* Set before any hs_compile call */
hs_set_allocator(hs_rte_malloc, hs_rte_free);

/* Now the compiled database lives in hugepage memory on the correct socket */
hs_compile_multi(patterns, flags, ids, count, HS_MODE_BLOCK,
                  NULL, &db, &compile_error);

/* Per-lcore scratch: clone from global, each on its lcore's socket */
RTE_LCORE_FOREACH_WORKER(lcore_id) {
    int lsock = rte_lcore_to_socket_id(lcore_id);
    /* hs_set_allocator with lsock before clone */
    hs_clone_scratch(global_scratch, &per_lcore_scratch[lcore_id]);
}
```

Without this, Hyperscan uses `malloc()` (system allocator), which allocates
from normal 4KB pages on the process's default NUMA node — not necessarily
the scanning lcore's socket.

### 6. Detecting the NUMA topology of your server

```bash
# Show NUMA nodes and their CPUs
numactl --hardware

# Show which NUMA node a PCI device (NIC) is on
cat /sys/bus/pci/devices/0000:01:00.0/numa_node

# Show hugepage allocation per node
cat /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages

# DPDK binding status (shows socket column)
dpdk-devbind --status
```

On a single-socket development machine, NUMA doesn't matter. But on
production servers (dual Xeon, dual EPYC), getting NUMA wrong silently
halves throughput. Always check before deploying.

---

## Common NUMA mistakes in DPDK code

| Mistake | Symptom | Fix |
|---|---|---|
| `rte_malloc()` instead of `rte_malloc_socket()` for hash tables | 2× lookup latency on dual-socket | Use `rte_malloc_socket(..., rte_socket_id())` |
| mbuf pool on wrong socket from NIC | High imissed, NIC DMA stalls | `rte_pktmbuf_pool_create(..., rte_eth_dev_socket_id(port))` |
| Hyperscan DB on socket 0, workers on socket 1 | hs_scan takes 2× longer | Use `hs_set_allocator` to redirect to NUMA-local hugepages |
| Rings created on SOCKET_ID_ANY | Random penalty | `rte_ring_create(..., rte_socket_id(), ...)` |

---

## Next module

**Module 15 — Hyperscan: Compile Patterns**: The first Hyperscan module.
Compile single and multi-pattern databases (`hs_compile` / `hs_compile_multi`),
understand the difference between regex and literal (`hs_compile_lit_multi`),
handle compile errors, and serialize/deserialize a database for persistence.
This is `hs_create_db()` from `domain_scan.c`.
