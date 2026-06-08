# Module 08 — DPDK EAL Initialization

> **Reference code** — requires DPDK installed. Read the code as a reference
> when building a real DPDK application. The concepts and API calls are
> production-accurate.

## What you learn

How to initialize the DPDK Environment Abstraction Layer (`rte_eal_init`),
build an EAL argument vector programmatically from config, assign CPU cores
to roles (RX / TX / WORKER), launch per-lcore functions, and shut down cleanly.

This is the first code that runs in every DPDK application after the
application-level setup (config, logger). Everything in subsequent modules —
mempools, ports, rings, Hyperscan — only works after `rte_eal_init` succeeds.

---

## What rte_eal_init actually does

```
rte_eal_init(argc, argv)
  │
  ├─ Locks hugepages
  │    mmap() pre-allocated 2MB pages into the process address space.
  │    These are pinned in physical memory — the OS never pages them out.
  │    All rte_mbuf packet buffers live here (zero kernel allocation at runtime).
  │
  ├─ Sets CPU affinity
  │    Each lcore in the list is pinned to a physical CPU core via
  │    pthread_setaffinity_np(). From this point on, the OS never
  │    migrates these threads — they run on their assigned physical core only.
  │
  ├─ Initialises per-lcore memory
  │    rte_malloc zones per NUMA socket. Subsequent rte_malloc_socket()
  │    calls allocate from the correct socket's hugepage pool.
  │
  ├─ Probes PCI devices (NICs)
  │    Finds NIC devices bound with dpdk-devbind --bind=vfio-pci.
  │    DPDK takes exclusive ownership — the kernel network stack can no
  │    longer see this NIC.
  │
  └─ Returns: number of EAL args consumed
               (so app can parse its own args from argv[ret..argc])
```

---

## Where this fits in the real application

```
main()
  │
  ├─► config_load()           (Module 01)
  ├─► logger_init()           (Module 02)
  │
  ├─► build_eal_args()        ← reads cores, socket_mem from config
  ├─► rte_eal_init()          ← THIS MODULE
  │
  ├─► rte_pktmbuf_pool_create()   (Module 09)
  ├─► port_init()                 (Module 10)
  ├─► hs_init_global_scratch() (Module 11)
  ├─► kafka_init()                (Module 12)
  ├─► rte_ring_create()           (Module 03 concept)
  │
  ├─► assign_lcore_roles()    ← THIS MODULE
  ├─► rte_eal_remote_launch() ← THIS MODULE (one call per worker lcore)
  │
  ├─► main lcore control loop (Kafka poll, stats, OAM)
  │
  └─► rte_eal_cleanup()       ← THIS MODULE
```

---

## Files

| File | Purpose |
|---|---|
| `eal_init.c` | Full EAL init, lcore topology, role assignment, launch, shutdown |
| `Makefile` | DPDK build via pkg-config (RedHat/Rocky compatible) |

---

## Setup (RedHat / Rocky Linux)

```bash
# Install DPDK
dnf install dpdk dpdk-devel

# Allocate hugepages (2MB pages, 512 = 1 GB)
echo 512 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# Mount hugepage filesystem (if not auto-mounted)
mount -t hugetlbfs nodev /dev/hugepages

# For NIC binding (optional — needed for real packet IO)
dpdk-devbind --status
dpdk-devbind --bind=vfio-pci <PCI_addr>

# Build and run
make
sudo ./eal_init
```

---

## Key concepts in the code

### 1. EAL arg vector — built by the app, not the shell

```c
/* Instead of running: ./dp_app -l 0-7 --socket-mem 2048 */
/* The app builds argv[] programmatically from config: */

eal_add_arg("dp_app");
eal_add_arg("-l");
eal_add_arg(config_get_string(&cfg, "eal", "cores", "0-3"));
eal_add_arg("--socket-mem");
eal_add_arg(config_get_string(&cfg, "eal", "socket_mem", "1024"));
...
ret = rte_eal_init(eal_argc, eal_argv);
```

This is the pattern in `app_main.c`. Operators configure cores and memory
in `dp_app.conf` — they never need to know DPDK command-line flags.

### 2. Lcore vs CPU core

A **lcore** is DPDK's abstraction for a logical CPU core (a hardware thread
in hyperthreading terms). Lcore IDs are DPDK's numbering, not the OS's.
A physical machine with 8 cores × 2 HT = 16 logical CPUs → 16 lcores (0–15).

```c
rte_lcore_count()              /* total lcores handed to DPDK */
rte_get_main_lcore()           /* main thread's lcore ID      */
rte_lcore_to_socket_id(id)     /* which NUMA socket this lcore is on */
RTE_LCORE_FOREACH_WORKER(id)   /* iterate all non-main lcores */
```

### 3. Role assignment — why it matters

The NIC delivers packets to a fixed (port, queue) pair. Only one lcore
polls each queue. If that lcore also runs policy logic, it becomes the
bottleneck and the queue fills → drops.

The solution: dedicated roles.
```
NIC port 0 queue 0 → RX lcore 1 → rte_ring → Worker lcores 3,4,5,6
                                                      ↓
                   TX lcore 2 ← rte_ring ←────────────┘
```

### 4. `rte_eal_remote_launch` vs `pthread_create`

```c
/* DPDK way — lcore already pinned to CPU core, just start the function */
rte_eal_remote_launch(lcore_worker_func, &lcore_info[id], id);

/* NOT used in DPDK for worker cores — would create an unpinned thread */
pthread_create(&tid, NULL, lcore_worker_func, &lcore_info[id]);
```

`rte_eal_remote_launch` uses a pre-created pthread that is already pinned
to the target CPU core. This avoids thread creation overhead at launch time.

### 5. `rte_pause()` — the polling spin hint

```c
while (running) {
    nb_rx = rte_eth_rx_burst(port, queue, mbufs, BURST);
    if (nb_rx == 0) {
        rte_pause();   /* x86 PAUSE instruction — reduces power, improves HT */
        continue;
    }
    /* process mbufs */
}
```

**Never** use `sleep()` or `usleep()` in an lcore function. Sleeping yields
the core to the OS scheduler. The lcore may not resume for milliseconds —
during which time the NIC RX ring fills and packets are dropped.

### 6. `__rte_cache_aligned` on `worker_lcore_info_t`

```c
typedef struct {
    unsigned int  lcore_id;
    lcore_role_t  role;
    atomic_ulong  pkt_rx;     /* worker writes; main reads for stats */
    ...
} __rte_cache_aligned worker_lcore_info_t;
```

Each `worker_lcore_info_t` is aligned to a 64-byte cache line boundary.
An array of these structs ensures adjacent workers don't share a cache line.
Without this, when lcore 3 updates `pkt_rx`, the CPU broadcasts a cache
invalidation to lcore 4 — even though lcore 4 only updates its own `pkt_rx`.

### 7. Graceful shutdown sequence

```
SIGTERM received
  → signal_handler sets force_quit = 1
  → main lcore exits control loop
  → sets lcore_info[id].running = 0 for each worker
  → worker lcores see running=0, exit their loops
  → rte_eal_wait_lcore(id) for each worker  ← must call before cleanup
  → flush CDR to Kafka
  → rte_eal_cleanup()
  → exit(0)
```

`rte_eal_cleanup()` must NOT be called until all remotely launched lcore
functions have returned. Calling it while a lcore is still running causes
a crash — the hugepage memory it's accessing gets unmapped.

---

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `No hugepages available` | Hugepages not allocated | `echo 512 > /sys/.../nr_hugepages` |
| `Cannot init EAL: invalid core list` | lcore range invalid | Check CPU count with `nproc` |
| `EAL: No probed ethernet devices` | NIC not bound to vfio-pci | `dpdk-devbind --bind=vfio-pci <addr>` |
| `rte_eal_init: Permission denied` | Not running as root | `sudo ./eal_init` or set capabilities |
| `Cannot create lock on hugepage file` | Another DPDK process running | Kill other DPDK processes first |

---

## Next module

**Module 09 — Mempool + mbuf**: Create an `rte_mempool` for packet buffers
(`rte_mbuf`), understand the mbuf lifecycle (alloc → fill → process → free),
and learn why mempools exist (pre-allocation eliminates malloc in the fast path).
