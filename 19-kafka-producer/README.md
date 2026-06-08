# Module 19 — Kafka Producer (CDR Export)

> Requires **librdkafka**. Needs a running Kafka broker for full demo.

## What you learn

How to implement a Kafka producer in C using `librdkafka` for CDR
(Charging Data Record) export — the pattern used in the DP application to publish
every policy decision to a Kafka topic for downstream billing, analytics,
and SIEM systems.

Covers producer config, delivery report callbacks, per-worker CDR batching,
timer-based flush, back-pressure handling, and graceful shutdown with flush.

---

## Kafka in the DP application architecture

```
the DP application (this module: producer)
  │
  │  rd_kafka_produce() CDRs → Kafka broker
  │  Topic: dp_cdr
  │
  ├── Consumer: Billing microservice (charges subscribers)
  ├── Consumer: Analytics (traffic patterns, blocked categories)
  └── Consumer: SIEM (security incidents from sinkhole events)

the DP application (Module 20: consumer)
  │
  │  rd_kafka_consumer_poll() ← Kafka broker
  │  Topic: policy_updates  (from Provisioning Module)
  │
  └── Update domain_details_table, recompile Hyperscan DB
```

---

## Setup

```bash
# Install librdkafka
dnf install librdkafka librdkafka-devel   # RedHat/Rocky
apt-get install librdkafka-dev            # Ubuntu

# Start a test Kafka broker (Docker)
docker run -d -p 9092:9092 --name kafka apache/kafka:3.7.0

# Build and run
make
./kafka_producer localhost:9092 dp_cdr

# Watch messages arrive (separate terminal)
docker exec kafka /opt/kafka/bin/kafka-console-consumer.sh \
    --topic dp_cdr --from-beginning \
    --bootstrap-server localhost:9092
```

---

## Files

| File | Purpose |
|---|---|
| `kafka_producer.c` | Full producer: config, CDR struct, batch, delivery callback, shutdown |
| `Makefile` | Links with `-lrdkafka` (pkg-config aware) |

---

## Key concepts

### 1. Delivery report callback — the only true confirmation

```c
rd_kafka_conf_set_dr_msg_cb(conf, delivery_report_cb);

static void delivery_report_cb(rd_kafka_t *rk,
                                const rd_kafka_message_t *msg, void *opaque)
{
    if (msg->err) {
        LOG_WARN("CDR delivery failed: %s", rd_kafka_err2str(msg->err));
        /* retry? drop? depends on CDR loss policy */
    } else {
        /* msg->offset: broker confirmed receipt at this offset */
        /* msg->partition: which partition */
    }
}
```

`rd_kafka_produce()` returning 0 means the message entered the **local queue**.
The broker has NOT received it yet. The delivery callback fires (via `rd_kafka_poll()`)
only after the broker ACKs the message. This is the only moment you know it's safe.

### 2. `rd_kafka_poll()` — must be called regularly

```c
/* In the main lcore control loop (every 100ms): */
rd_kafka_poll(g_producer, 0);  /* 0 = non-blocking, process pending callbacks */
```

Without `rd_kafka_poll()`, the delivery callback NEVER fires — even if the
broker received all messages. The local queue fills up, and subsequent
`rd_kafka_produce()` calls return `QUEUE_FULL`. In the DP application, this is called
in the main lcore's while(running) loop alongside Kafka policy consumer polling.

### 3. Batching — reducing per-message overhead

```
Without batching:
  1000 CDRs/sec × rd_kafka_produce() syscall overhead = bottleneck

With CDR_BATCH_MAX=256:
  Accumulate 256 records → burst to Kafka queue in one tight loop
  rd_kafka_poll() in between → delivery callbacks processed
  Net: ~4 bursts/sec of 256 messages each

Timer-based flush (CDR_FLUSH_MS=100):
  Even at low traffic (10 CDRs/sec):
  100ms timer fires → flush 1-9 records → no CDR older than 100ms
```

The Kafka-side `linger.ms=5` config further batches messages in the
local queue before sending to the broker — amortizing TCP round trips.

### 4. Message key — per-subscriber ordering

```c
uint32_t key_ip = htonl(rec->subscriber_ip);
rd_kafka_produce(topic, RD_KAFKA_PARTITION_UA,
                  RD_KAFKA_MSG_F_COPY,
                  payload, len,
                  &key_ip, sizeof(key_ip),  /* KEY */
                  NULL);
```

Kafka partitions messages by hash of the key. Using subscriber IP as key
means all CDRs for one subscriber land in one partition — in order.
A billing consumer can process per-subscriber CDRs sequentially without
needing to sort by timestamp across partitions.

### 5. `RD_KAFKA_MSG_F_COPY` vs `RD_KAFKA_MSG_F_FREE`

```c
RD_KAFKA_MSG_F_COPY:  librdkafka copies payload — you can reuse buffer immediately
RD_KAFKA_MSG_F_FREE:  librdkafka frees payload via free() after delivery
RD_KAFKA_MSG_F_ZERO:  neither — you manage lifetime (dangerous, fast)
```

In the DP application, `MSG_F_COPY` is used because the JSON buffer is stack-allocated
and reused for each CDR. `MSG_F_FREE` would require heap-allocating each
CDR buffer — more overhead per message.

### 6. Back-pressure: `QUEUE_FULL`

```c
if (err == -1 && rd_kafka_last_error() == RD_KAFKA_RESP_ERR__QUEUE_FULL) {
    /* Broker unreachable or too slow */
    rd_kafka_poll(producer, 100);  /* wait 100ms, process callbacks, free slots */
    /* retry once */
}
```

`QUEUE_FULL` means `queue.buffering.max.messages` was exceeded. This happens
when:
- Broker is down / unreachable
- Network congestion
- Consumer lag causing Kafka backpressure

In the DP application, CDR loss is acceptable (< 0.01% target). If the queue stays
full, CDRs are dropped rather than blocking the packet processing pipeline.

### 7. Graceful shutdown

```c
/* WRONG: destroy immediately — queued CDRs lost */
rd_kafka_destroy(producer);

/* CORRECT: flush first */
rd_kafka_flush(producer, 5000);  /* wait up to 5s for delivery */
rd_kafka_topic_destroy(topic);
rd_kafka_destroy(producer);
```

`rd_kafka_flush()` blocks until all queued messages are acknowledged or
the timeout expires. In the DP application, this is called after the main lcore
exits its control loop, before `rte_eal_cleanup()`.

---

## CDR JSON format

```json
{
  "ts": 1717430400123,
  "ip": "198.51.100.42",
  "subscriber_id": "subscriber-42",
  "domain": "malware.ru",
  "qtype": 1,
  "action": "sinkhole",
  "group": 3,
  "category": "0x00000001"
}
```

In production, Protobuf or Avro would be more efficient (smaller messages,
schema evolution). JSON is used here for inspectability with
`kafka-console-consumer.sh`.

---

## Kafka config reference

| Property | DP value | Effect |
|---|---|---|
| `bootstrap.servers` | from config | Broker address(es) |
| `acks` | `"1"` | Leader ACK only (speed over durability) |
| `retries` | `3` | Retry transient errors |
| `batch.size` | `65536` | Max bytes batched per partition |
| `linger.ms` | `5` | Wait up to 5ms to fill a batch |
| `compression.type` | `snappy` | Reduce network + disk by ~3-5× |
| `queue.buffering.max.messages` | `100000` | Drop CDRs beyond this |

---

## Next module

**Module 20 — Kafka Consumer (Policy Sync)**: The Kafka consumer that
receives policy updates from the PM (Provisioning Module) and applies them
to the in-memory data plane — updating `domain_details_table`, recompiling
Hyperscan databases, and handling the `SYNC_COMPLETE` protocol.
