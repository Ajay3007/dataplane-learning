# Module 15 — Hyperscan: Pattern Compilation

> Requires **Hyperscan (libhs)** installed. See setup instructions below.

## What you learn

How to compile Hyperscan pattern databases — both regex (`hs_compile_multi`)
and literal (`hs_compile_lit_multi`) — including the exact `parseFlags()`,
`parseFile()`, and `hs_create_db()` implementations from `domain_scan.c`
in the DP application. Also covers DB info query, serialization for persistence, and
compile error handling.

---

## Hyperscan's role in the DP application

```
Two databases, two purposes:

1. domainsPatternDB (global, regex):
   Compiled once at startup from patterns.txt + patterns2.txt.
   Patterns match the structure of TLS ClientHello (SNI extension header),
   HTTP Host headers, and IP addresses in URLs.
   Used by: hs_scan_payload() on every packet payload.
   IDs: TLS=1, HTTP_IPV4=2, HTTP_DOMAIN=3, HTTP_IPV6=4

2. group->database (per-group, literal):
   Compiled once per enterprise group when policy syncs from Kafka.
   Contains exact domain names: "google.com", "malware.ru", etc.
   Used by: hs_scan_domain_group() as the Hyperscan fallback when
   rte_hash exact lookup misses.
   IDs: arbitrary (one per domain)
```

---

## Setup (Hyperscan installation)

```bash
# RedHat 8 / Rocky Linux 8
dnf install hyperscan hyperscan-devel

# Ubuntu 22.04+
apt-get install libhyperscan-dev

# From source (latest version):
git clone https://github.com/intel/hyperscan
cd hyperscan && mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
ninja && sudo ninja install
```

Hyperscan requires SSE4.2 support (Intel Sandy Bridge or newer).
Check: `grep -m1 'sse4_2' /proc/cpuinfo`

---

## Build and run

```bash
make
./hs_compile
```

Expected output:
```
=== Module 15: Hyperscan Pattern Compilation ===
Hyperscan version: 5.4.2 2023-01-01

Demo 1: Global threat DB (hs_compile_multi, regex)
  Compiling 4 regex patterns...
  Compilation SUCCESS
  [DB: global_threat_db]
    info: Version: 5.4.2 Features: AVX2 Mode: BLOCK
    size: 3248 bytes (3.2 KB)

Demo 2: Per-group domain DB (hs_compile_lit_multi, literals)
  Compiling 7 literal domains...
  Compilation SUCCESS
  [DB: per_group_domain_db]
    size: 856 bytes (0.8 KB)
...
```

---

## Files

| File | Purpose |
|---|---|
| `hs_compile.c` | `parseFlags`, `parseFile`, `hs_create_db`, 6 demos |
| `sample_patterns.txt` | Example patterns.txt in the real format (ID:/pattern/flags) |
| `Makefile` | Links with `-lhs` (pkg-config aware) |

---

## Key concepts

### 1. `hs_compile_multi` vs `hs_compile_lit_multi`

```c
/* REGEX mode — for patterns.txt (TLS/HTTP patterns) */
hs_compile_multi(
    patterns,    /* char *[] of regex strings */
    flags,       /* unsigned[] of HS_FLAG_* per pattern */
    ids,         /* unsigned[] of IDs per pattern */
    count,       /* number of patterns */
    HS_MODE_BLOCK,
    NULL,        /* platform: NULL = current CPU */
    &db, &err
);

/* LITERAL mode — for domain policy (no regex engine) */
hs_compile_lit_multi(
    patterns,          /* char *[] of byte strings */
    flags,
    ids,
    lens,              /* size_t[] — length of each literal */
    count,
    HS_MODE_BLOCK,
    NULL,
    &db, &err
);
```

**When to use literal:** domain names are exact strings, not patterns.
Literal mode is significantly faster to compile AND scan because Hyperscan
uses SIMD byte-comparison algorithms (Aho-Corasick with SIMD acceleration)
instead of building a full NFA/DFA. For 10000 domain literals, literal
compile takes ~50ms vs ~2s for regex.

### 2. `HS_FLAG_SINGLEMATCH` — the most important performance flag

```c
flags[i] = HS_FLAG_CASELESS | HS_FLAG_SINGLEMATCH;
```

Without `SINGLEMATCH`: Hyperscan fires a match callback for **every**
position in the data where the pattern matches. For a domain like "google"
in a 1500-byte packet, it might fire 5+ times.

With `SINGLEMATCH`: Hyperscan fires exactly **once** per scan per pattern.
Since the DP application only needs to know IF a pattern matched (not all positions),
`SINGLEMATCH` eliminates redundant callbacks and is always used.

### 3. Pattern IDs — the callback dispatch mechanism

```c
/* Assigned at compile time: */
ids[0] = HS_PATTERN_ID_TLS;         /* 1 */
ids[1] = HS_PATTERN_ID_HTTP_DOMAIN; /* 3 */

/* In the on_hs_match callback (domain_scan.c): */
int on_hs_match(unsigned int id, unsigned long long from,
              unsigned long long to, unsigned int flags, void *ctx)
{
    switch (id) {
    case HS_PATTERN_ID_TLS:        /* 1 */
        /* read SNI at from+7 / from+9 (Module 07 pattern) */
        break;
    case HS_PATTERN_ID_HTTP_DOMAIN: /* 3 */
        /* extract Host: header domain */
        break;
    }
    return 0;  /* 0 = continue scanning, non-zero = stop */
}
```

The ID is the only way to identify which pattern fired. Design your ID
scheme carefully — IDs must be consistent between compile and scan.

### 4. `parseFile` — pattern file format

```
# Comment
ID:/regex_or_literal/flags

Example:
4:/\x00\x00\x00\x00\x00/H
6:/Host: [a-zA-Z0-9._-]+/iH
```

- `ID`: unsigned integer, maps to the `ids[]` array
- Pattern: between the two `/` delimiters
- Flags: one or more flag characters after the closing `/`

In the DP application, `hs_init_global_scratch()` calls `parseFile()` for both
`patterns.txt` and `patterns2.txt`, then compiles both into `domainsPatternDB`.

### 5. Serialization for fast restart

```c
/* After compilation — save to disk: */
size_t sz;
hs_serialized_database_size(db, &sz);
char *buf = malloc(sz);
hs_serialize_database(db, buf, sz);
fwrite(buf, 1, sz, fp);   /* write to /var/lib/dp_app/global.hsdb */

/* On restart — load instead of recompile: */
fread(buf, 1, sz, fp);
hs_deserialize_database(buf, sz, &db);
/* db is ready to use, same as a freshly compiled one */
```

For the global DB with 100+ patterns, compilation takes ~500ms.
With serialization, restart takes ~5ms (just a memory copy).
In DP production, recompilation happens only when patterns change.

### 6. Compile error handling

```c
hs_compile_error_t *err = NULL;
hs_error_t r = hs_compile_multi(..., &db, &err);

if (r != HS_SUCCESS) {
    if (err) {
        /* err->expression: 0-indexed position of the bad pattern */
        /* err->message: human-readable description                */
        LOG_ERROR("Pattern %d failed: %s", err->expression, err->message);

        /* MANDATORY: free the error struct to avoid memory leak */
        hs_free_compile_error(err);
    }
    return -1;
}
/* err is NULL on success — no need to free */
```

Common compile errors:
- Unclosed bracket: `[0-9` → "missing ]"
- Invalid escape: `\q` → "unrecognised escape sequence"
- Too complex: patterns that require exponential NFA → Hyperscan rejects them

### 7. `HS_MODE_BLOCK` — always use this in the DP application

```
HS_MODE_BLOCK:    Scan a complete buffer at once.
                  Most efficient for fixed-size packet payloads.
                  The DP application uses this for both DNS and TLS payloads.

HS_MODE_STREAM:   Data arrives in chunks (e.g., TCP stream reassembly).
                  Heavier — maintains state between chunks.
                  Would be needed if scanning fragmented DNS over TCP.

HS_MODE_VECTORED: Scan multiple non-contiguous buffers in one call.
                  Not used in the DP application.
```

---

## Next module

**Module 16 — Hyperscan: Scratch + Scan**: Allocate scratch space
(`hs_alloc_scratch`), clone per-lcore scratch (`hs_clone_scratch`), and
call `hs_scan()` with the `onMatch` callback. This is `hs_scan_payload()`
and `hs_scan_domain_group()` from `domain_scan.c`.
