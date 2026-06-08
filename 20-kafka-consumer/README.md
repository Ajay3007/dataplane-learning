# Module 20 — Kafka Consumer (Policy Sync)

> Requires **librdkafka**. Standalone demo runs without a broker.

## What you learn

How the DP application Kafka consumer receives policy updates from the
Provisioning Module and applies them to the in-memory domain tables —
the SYNC_COMPLETE atomic swap protocol, manual offset commit, partition
rebalance callbacks, and the RCU QSBR write-side pattern.

---

## Build and run

```bash
make

# Standalone demo (no Kafka broker needed):
./kafka_consumer --demo

# With a real Kafka broker:
./kafka_consumer localhost:9092 policy_updates
```

Standalone output:
```
=== Module 20: Kafka Consumer (Policy Sync) ===

Processing 14 policy messages:

  [BEGIN_SYNC]    group=enterprise_a — building pending table
  [ADD_DOMAIN]    group=enterprise_a  domain=google.com        action=whitelist
  [ADD_DOMAIN]    group=enterprise_a  domain=facebook.com      action=blacklist
  ...
  [SYNC_COMPLETE] group=enterprise_a — 5 domains now active

  [BEGIN_SYNC]    group=enterprise_b — building pending table
  ...
  [SYNC_COMPLETE] group=enterprise_b — 1 domains now active

  [ADD_MALICIOUS] domain=c2-botnet.badactor.net  conf=98%

Final policy state:
  Group [enterprise_a]  domains=5
    google.com         whitelist=1  blacklist=0
    facebook.com       whitelist=0  blacklist=1
    ...

Policy lookup verification:
  [enterprise_a] google.com           → WHITELIST
  [enterprise_a] facebook.com         → BLACKLIST
  [enterprise_a] youtube.com          → NOT FOUND (removed)
  [enterprise_a] new-approved-site.com → WHITELIST
  [enterprise_b] phishing-bank.tk     → BLACKLIST
```

---

## The SYNC_COMPLETE protocol — why it matters

```
WITHOUT SYNC_COMPLETE (naive immediate apply):
  
  Main lcore applies ADD_DOMAIN(google.com=whitelist)
  Worker lcore reads domain_details_table — sees google.com=whitelist ✓
  
  Main lcore applies ADD_DOMAIN(tracker.com=blacklist)
  Worker lcore processes request for tracker.com — MISS (not yet in table)
  → tracker.com is ALLOWED when it should be blocked
  
  This is a race condition: partial policy state visible to workers.

WITH SYNC_COMPLETE (buffered apply):
  
  BEGIN_SYNC: all updates go to pending table (invisible to workers)
  ADD_DOMAIN(google.com=whitelist)  → pending
  ADD_DOMAIN(tracker.com=blacklist) → pending
  ...
  SYNC_COMPLETE:
    1. Build new Hyperscan DB from pending domains
    2. atomic_swap(active_table ← pending_table)  ← workers see complete state
    3. RCU synchronize (wait for all workers to quiesce)
    4. free(old_active_table)
  
  Workers only ever see either the full old policy or the full new policy.
  Never a partial intermediate state.
```

---

## Where this fits in the real application

```
Provisioning Module → publishes to Kafka

DP Application main lcore control loop:
  while (running) {
      rd_kafka_poll(cdr_producer, 0);         ← Module 19
      
      msg = rd_kafka_consumer_poll(policy_consumer, 100);
      if (msg) {
          apply_policy_message(parse(msg));   ← THIS MODULE
          if (SYNC_COMPLETE)
              rd_kafka_commit_message(consumer, msg, 0);
      }
      
      cdr_batch_flush_if_timeout(&batches);   ← Module 19
      print_stats();
  }
```

---

## Key concepts

### 1. `auto.offset.reset = "earliest"` for policy topics

```c
{ "auto.offset.reset", "earliest" }
```

On DP restart (crash or maintenance), the consumer re-reads all policy
messages from the beginning of the topic. This rebuilds the domain tables
from scratch — the same state as if the PM had just synced everything.

`"latest"` would miss all policies published before the restart, leaving
domain tables empty until the next full sync.

### 2. `enable.auto.commit = false` — manual commit

```c
{ "enable.auto.commit", "false" }
```

With `auto.commit = true`: Kafka commits offsets periodically (e.g., every 5s).
If the DP crashes AFTER consuming BEGIN_SYNC + ADD_DOMAINs but BEFORE
SYNC_COMPLETE, the auto-committed offset means those messages won't be
re-delivered. The pending table is lost, but Kafka thinks they were processed.

With `auto.commit = false`, we commit ONLY at SYNC_COMPLETE:
```c
if (pmsg.type == MSG_SYNC_COMPLETE) {
    rd_kafka_commit_message(rk, msg, 0);  /* sync commit */
}
```

A crash before SYNC_COMPLETE replays from BEGIN_SYNC — ensuring the
entire sync unit is re-applied atomically.

### 3. Atomic swap + RCU QSBR (the real write-side pattern)

```c
/* What happens at SYNC_COMPLETE in the real DP application: */

/* Step 1: build pending Hyperscan DB from pending domain table */
hs_db_compile_for_groups(group);   /* slow, done in main lcore */

/* Step 2: atomic pointer swap */
rte_atomic64_set(
    (rte_atomic64_t *)&group->domain_details_table,
    (int64_t)pending_rte_hash_table
);
/* Worker lcores now use the new table for all subsequent lookups */

/* Step 3: wait for all readers to quiesce */
rte_rcu_qsbr_synchronize(qsbr_var, RTE_QSBR_THRID_INVALID);
/* Returns when every worker lcore has called rte_rcu_qsbr_quiescent() */
/* Safe to free old table: no lcore holds a pointer to it */

/* Step 4: free old table */
rte_hash_free(old_table);
```

Workers must call `rte_rcu_qsbr_quiescent()` regularly in their poll loop
(between bursts). This is the RCU QSBR design — implemented in the DP core library, but
the protocol shown here is what the writer side does at SYNC_COMPLETE.

### 4. Partition rebalance callback — must be registered

```c
rd_kafka_conf_set_rebalance_cb(conf, rebalance_cb);

static void rebalance_cb(rd_kafka_t *rk, rd_kafka_resp_err_t err,
                          rd_kafka_topic_partition_list_t *partitions, void *opaque)
{
    if (err == RD_KAFKA_RESP_ERR__ASSIGN_PARTITIONS)
        rd_kafka_assign(rk, partitions);
    else
        rd_kafka_assign(rk, NULL);
}
```

Without this callback, partition assignment is handled automatically but
`rd_kafka_assign()` is not called — the consumer never actually starts
receiving messages. This silent failure is a common bug when first using
librdkafka.

### 5. `rd_kafka_consumer_close()` — graceful shutdown

```c
/* WRONG: destroys immediately, may lose messages mid-delivery */
rd_kafka_destroy(rk);

/* CORRECT: close consumer (sends final offset commit, leaves group) */
rd_kafka_consumer_close(rk);
rd_kafka_destroy(rk);
```

`rd_kafka_consumer_close()` sends a leave-group request to the broker
and commits pending offsets. Without it, the broker waits for
`session.timeout.ms` (30s) before reassigning partitions to another
consumer — blocking policy updates to other DP instances.

---

## Message format (JSON from PM)

```json
{"type":"BEGIN_SYNC",   "group_id":"enterprise_a"}
{"type":"ADD_DOMAIN",   "group_id":"enterprise_a",
 "domain":"blocked.com","action":"blacklist","category":2}
{"type":"SYNC_COMPLETE","group_id":"enterprise_a"}
{"type":"ADD_MALICIOUS","domain":"c2.evil.net","category":4,"confidence":98}
```

| type | Effect |
|---|---|
| `BEGIN_SYNC` | Start buffering into pending table |
| `ADD_DOMAIN` | Add domain to pending (or active if no sync) |
| `REMOVE_DOMAIN` | Remove from active table immediately |
| `SYNC_COMPLETE` | Atomic swap pending→active + commit offset |
| `ADD_MALICIOUS` | Add to global malicious table (from IDPS feed) |
| `ADD_GROUP` | Create a new enterprise group |
| `REMOVE_GROUP` | Delete a group and all its policies |

---

## Next module

**Module 21 — Full Pipeline (Annotated Assembly)**: A single annotated
file showing how all 20 modules connect into a complete dataplane
application — from `main()` startup through packet processing to
graceful shutdown.
