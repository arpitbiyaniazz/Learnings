# Chapter 3 — Apache Kafka: The Event Backbone

**What this chapter is.** A from-zero, self-contained guide to Apache Kafka: what it is, the exact problem it solves, every core concept, and how to run it, secure it, produce to it, and consume from it — in general, and specifically the way the GIBP ledger uses it. If you have never touched a "message queue" in your life, start here and read top to bottom. If you already know Kafka and just want *this repo's* wiring, jump to §7.

**Status in this repo.** Kafka is a **load-bearing production dependency**, not an experiment.
- Locally it runs as one Docker container (`confluentinc/cp-kafka:7.8.0`) in **KRaft mode** (no ZooKeeper) with **SASL/PLAIN** authentication.
- In production it is run by the **Strimzi operator** on Kubernetes (a 3-node cluster), installed as an ArgoCD application.
- `apps/api` (NestJS) **produces** audit events onto Kafka. `apps/kafka-worker` (NestJS microservice) **consumes** two families of topics: change-data-capture (CDC) rows streamed from the legacy `gibp04` MySQL database by **Debezium**, and the audit topics produced by the API.
- All Kafka wiring lives in one shared library: `packages/kafka`.

**How to read this.**
- §1–§4 teach Kafka from first principles — read these once and they pay off for your whole career. Every concept opens with *the problem it solves*.
- §5 is a hands-on lab: stand up a real broker in Docker and prove it works from the command line.
- §6 shows the Node/NestJS code patterns generically.
- §7 is the map of *this* repository — the file you'll come back to.
- §8–§11 are production concerns, debugging playbooks, gotchas, and a glossary.

Cross-links: change-data-capture and Debezium have their own chapter — see [`04-debezium-and-cdc.md`](./04-debezium-and-cdc.md). For how the whole pipeline is deployed, see [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md).

---

## 1. The problem: why message streaming exists

Imagine the simplest way for two services to talk. Service A needs Service B to do something, so A calls B directly over HTTP and waits for the answer. This is a **synchronous, point-to-point call**. It is the first thing everyone reaches for, and it is fine — until it isn't.

Here is where it breaks, and every failure below is a real force that shaped Kafka:

**1. Tight coupling.** A must know B's address, B's API shape, and B must be *up right now*. Add a Service C that also cares about the same event ("a bill was paid") and now A must call B *and* C. Add D, E, F and A becomes a switchboard operator. A is now coupled to the entire downstream world. Every new consumer is a code change in the producer.

**2. Fragility / the "up right now" tax.** If B is down, restarting, or slow, A's request fails or hangs. In a synchronous chain `A → B → C`, the slowest and least-reliable link dictates the reliability of the whole thing. A financial system cannot afford "the payment didn't record because the audit service was redeploying."

**3. No buffering for spikes.** If 10,000 bills settle in one burst, A hammers B with 10,000 simultaneous calls. B falls over. Synchronous calls have no shock absorber.

**4. No memory / no replay.** The call happens once, in the moment. If B processed the event but crashed before saving the result, the event is gone. There is no way to say "replay everything from Tuesday" because nothing was kept.

**5. No ordering guarantees across the fan-out.** If A fires events to B and C independently, and you later add a fourth consumer, you have no shared, durable record of *what happened in what order*.

What you actually want is a **durable, replayable, decoupled, ordered stream of events** that:
- the producer writes to **without knowing or caring who reads it**,
- **absorbs spikes** (the reader catches up at its own pace),
- **remembers** events for a configured time so a new or recovering consumer can **replay** them,
- lets **many independent consumers** read the **same** data without stepping on each other,
- preserves **order** for related events.

That is Kafka.

### A queue vs a log — the one distinction that unlocks everything

People say "message queue," but Kafka is not really a queue. This distinction matters enough to teach up front.

A **traditional queue** (think RabbitMQ, SQS in classic mode) is like a **conveyor belt of physical letters**. A worker takes a letter off the belt, and *that letter is now gone from the belt*. The message is **consumed and deleted**. If you have three workers, each letter goes to exactly one of them. Great for distributing work; useless for "let three different teams each read every letter," and useless for "replay last week" — the letters are gone.

A **log** (Kafka) is like a **shared logbook that is only ever appended to**. Every event is written as a new line at the bottom. **Reading a line does not erase it.** Each reader keeps a **bookmark** (their own private page number) and reads forward at their own speed. Ten different readers can each be at a different page in the same book. The book is trimmed only by a **time/size policy** (retention), never by "someone read this line."

```
  QUEUE (consume = delete)              LOG (append-only, read ≠ delete)
  ┌───────────────────────┐            ┌──────────────────────────────────┐
  │ letter → worker (gone) │            │ line 0  line 1  line 2  line 3 …  │
  └───────────────────────┘            │   ▲        ▲              ▲        │
   one message → one worker             │ reader B  reader A     reader C   │
   nothing kept                         │ (each has its own bookmark)       │
                                        └──────────────────────────────────┘
                                         one message → every interested reader
                                         kept until retention expires
```

Everything else in this chapter is a consequence of "Kafka is an append-only log that readers bookmark."

---

## 2. The mental model: an append-only logbook that many readers bookmark

Hold this single picture in your head for the rest of the chapter:

> **A Kafka topic is a logbook. Producers append events to the bottom. Every event gets a permanent line number. Many readers each keep their own bookmark and read forward at their own pace. Nobody can erase anyone else's page; the book is trimmed only by an age/size policy.**

Now map the analogy back to real Kafka vocabulary — you will use these words for the rest of your career:

| Analogy | Real Kafka term | Meaning |
|---|---|---|
| The logbook | **Topic** | A named stream of events (e.g. `audit-ledger-postings`). |
| A line in the book | **Record / message** | One event: an optional **key**, a **value** (the payload bytes), a timestamp, headers. |
| The permanent line number | **Offset** | A monotonically increasing integer identifying a record's position. |
| Who writes lines | **Producer** | Code that appends records to a topic. |
| Who reads lines | **Consumer** | Code that reads records forward from an offset. |
| A reader's bookmark | **Committed offset** | "I have processed up to here." Stored by Kafka per consumer group. |
| The library building | **Broker** | A Kafka server process that stores topics and serves reads/writes. |
| A team sharing one bookmark | **Consumer group** | A set of consumers that cooperate; the group has one bookmark per slice of the topic. |
| Splitting a fat book into volumes so many people write/read at once | **Partition** | A topic is split into partitions; each partition is its own ordered log. |
| Photocopies kept in other buildings | **Replication** | Each partition is copied to several brokers for durability. |
| The age/size trimming rule | **Retention** | How long records are kept before deletion. |

If a sentence in this chapter ever confuses you, translate it back into the logbook. "A consumer group commits its offset for a partition" = "a team writes down which page they've read to, for one volume of the book." That's it.

---

## 3. Core concepts and vocabulary

Read this table once now; refer back forever. Terms are defined in the order you'll meet them.

| Term | One-line definition | Why it exists |
|---|---|---|
| **Broker** | A single Kafka server. Stores partitions, serves produce/fetch requests. | The unit you scale horizontally; a **cluster** is many brokers. |
| **Cluster** | A group of brokers working together. | Durability and throughput beyond one machine. |
| **Topic** | A named, append-only stream of records. | The logical channel producers write to and consumers read from. |
| **Partition** | One ordered, immutable log; a topic is split into N of them. | **The unit of parallelism and of ordering.** More partitions = more parallel readers. |
| **Offset** | A record's integer position *within a partition*. | A stable address; how consumers bookmark their progress. |
| **Record / message** | key + value + timestamp + headers. | The event itself. |
| **Key** | Optional bytes attached to a record. | Determines *which partition* a record lands in, which controls ordering (§4). |
| **Producer** | Client that appends records. | Writes events. |
| **Consumer** | Client that reads records forward. | Reads events. |
| **Consumer group** | Consumers sharing a `group.id`; Kafka splits partitions among them. | Lets you scale reading horizontally *and* lets independent teams each read the whole topic (different group.id = independent bookmark). |
| **Rebalancing** | Kafka reassigning partitions when group membership changes. | Automatic failover/scaling within a group — but it pauses consumption briefly. |
| **Committed offset** | The offset a group has durably recorded as "done." | Where a consumer resumes after a restart. |
| **Replication factor (RF)** | How many brokers hold a copy of each partition. | Durability: lose a broker, keep the data. |
| **Leader / follower** | One replica is the **leader** (handles all reads/writes for that partition); the rest are **followers** that copy it. | Followers are hot standbys; a follower is promoted if the leader dies. |
| **ISR (in-sync replicas)** | The set of replicas currently caught up to the leader. | `min.insync.replicas` uses this to define "durably written." |
| **Retention** | Time- or size-based deletion policy for a topic's records. | Bounds disk usage; enables replay within the window. |
| **Log compaction** | An alternative to time-retention: keep only the *latest* record per key. | Turns a topic into a "current state per key" table (e.g. latest row per primary key). |
| **Controller** | The broker coordinating cluster metadata (leaders, partition assignments). | Cluster brain. In KRaft, a **quorum** of controllers runs Raft. |
| **KRaft** | Kafka Raft — Kafka managing its own metadata via a Raft quorum. | Replaces **ZooKeeper**. Fewer moving parts. This repo uses KRaft. |
| **ZooKeeper** | The old external service Kafka used for metadata (pre-KRaft). | You may still see it in old tutorials; **this repo does not use it.** |
| **Schema Registry** | A service storing versioned message schemas (Avro/Protobuf/JSON) and handing out schema IDs. | Producers and consumers agree on message shape *without shipping the schema in every message*; prevents silent breakage. |
| **Serialization** | Turning an object into bytes on the way out and back on the way in. | Kafka moves bytes; **Avro** (compact, schema'd) and **JSON** (human-readable) are the two used here. |
| **SASL** | An authentication framework; **SASL/PLAIN** = username+password over the wire. | Only authorized clients may connect. This repo uses SASL/PLAIN. |
| **TLS** | Transport encryption. | Confidentiality in transit; pair with SASL in real prod. |
| **DLQ (dead-letter queue)** | A separate topic where un-processable ("poison") messages are parked. | Keeps one bad message from blocking the whole partition forever. |

---

## 4. How it actually works (the deep dive)

This section is the mechanical heart. Once these six ideas click, Kafka has no more mysteries.

### 4.1 Partitions: the unit of both parallelism and ordering

A topic is not one log — it is **N logs** called partitions. When you create `payment.processed` with `partitions: 3`, Kafka creates three independent append-only logs: partition 0, 1, 2. Each has its own offset counter starting at 0.

```
TOPIC "payment.processed"  (3 partitions)

 partition 0:  [0][1][2][3][4] →   (append here)
 partition 1:  [0][1][2] →
 partition 2:  [0][1][2][3][4][5][6] →
                └ each box is a record; number is its offset
```

Two rules that follow directly, and that you must tattoo on your brain:

1. **Order is guaranteed *only within a single partition*.** Records in partition 0 are read strictly in offset order: 0, 1, 2, 3… But there is **no global order across partitions.** Partition 1's offset 2 and partition 0's offset 5 have no defined relative order. If two events *must* be processed in order relative to each other, they must be in the **same partition**.

2. **Partitions are the parallelism unit.** Within one consumer group, **each partition is handled by at most one consumer.** So a topic with 3 partitions can be read by at most 3 consumers *in the same group* working in parallel. A 4th consumer in that group sits idle. Want more parallelism? Add partitions.

### 4.2 How a key routes a record to a partition

When you send a record you may attach a **key**. Kafka decides the partition like this:

- **Key present:** `partition = hash(key) mod numPartitions`. Same key → **always the same partition** → those records are **strictly ordered relative to each other.**
- **Key absent (`null`):** Kafka spreads records across partitions (sticky/round-robin). Max throughput, **no ordering guarantee.**

This is *the* lever for correctness. In this repo the audit producer keys messages deliberately:
- `audit-business-events` is keyed by **`correlation_id`** — so every message from one user action lands on one partition and stays ordered.
- `audit-ledger-postings` is keyed by **`formance_transaction_id`** — so all postings of one ledger transaction stay together and in order.

Analogy: the key is the **"which volume of the logbook"** decision. Everything about entity X goes in volume `hash(X)`, so entity X's history is always read in order. But it also means the moment you *change the number of volumes* (add partitions), the `mod N` math changes and existing keys may route to a different volume — see §10.

### 4.3 Consumer groups and rebalancing

A **consumer group** is identified by a `group.id` string. Kafka guarantees: **within one group, every partition is assigned to exactly one consumer.** Two different groups are completely independent — each has its own set of bookmarks over the whole topic.

```
TOPIC with 3 partitions, TWO groups reading the SAME data:

                 ┌── group "billing-sync" (2 consumers) ──┐
   partition 0 ──┤→ consumer B1                            │
   partition 1 ──┤→ consumer B1                            │  B1 gets p0+p1, B2 gets p2
   partition 2 ──┴→ consumer B2                            │
                 └────────────────────────────────────────┘

                 ┌── group "bq-audit-sink" (1 consumer) ──┐
   partition 0 ──┤→ consumer S1                           │  independent bookmark:
   partition 1 ──┤→ consumer S1                           │  reads ALL 3 partitions,
   partition 2 ──┴→ consumer S1                           │  its own offsets
                 └────────────────────────────────────────┘
```

That right there is the whole "many independent consumers of the same data" superpower from §1: give each independent consumer a **different `group.id`** and they each get the full stream with their own progress. Give consumers the **same `group.id`** and they *share* the work (horizontal scaling).

**Rebalancing:** when a consumer joins or leaves a group (deploy, crash, scale-up), Kafka **reassigns partitions** across the surviving members. During a rebalance, consumption briefly pauses ("stop-the-world" in the classic protocol). Frequent rebalances = "rebalance storms" and are a real production pain (§8).

This repo runs **eight consumer groups** in the worker (see `kafka.constants.ts` → `KAFKA_CONSUMER_GROUPS`), one per concern, so a failure or slowdown in one does not stall the others. They are: `cdc-ingestion-group`, `accounts-sync-group`, `vendors-sync-group`, `bills-sync-group`, `batches-sync-group`, and three audit → BigQuery sinks.

### 4.4 Offsets and commit semantics

An **offset commit** is a consumer telling Kafka "I've successfully processed up to offset N in partition P for my group." On restart, the consumer resumes from the committed offset. This is the bookmark.

The subtle, important choices:

- **Auto-commit** (kafkajs / the NestJS transport default behavior): the client periodically commits offsets in the background. Simple, but dangerous — it can commit offset N *before* your handler actually finished processing N. If you crash after the commit but before the work, that record is **skipped** (data loss). If you crash after processing but before commit, it is **re-delivered** (duplicate).
- **Manual commit after processing:** commit only once your handler has durably done its work. Safer, gives **at-least-once** semantics (you may see duplicates, never silent loss).

The unavoidable takeaway, which we repeat because it is the #1 correctness rule: **you will get duplicates. Every consumer must be idempotent** (processing the same record twice must be harmless). More in §4.5 and §6.

### 4.5 Delivery guarantees

There are exactly three, and you should be able to explain them cold:

| Guarantee | What it means | How you get it | Cost |
|---|---|---|---|
| **At-most-once** | Each record delivered 0 or 1 times. May be **lost**, never duplicated. | Commit offset *before* processing. | Data loss on crash. Almost never what a financial system wants. |
| **At-least-once** | Each record delivered 1+ times. May be **duplicated**, never lost. | Commit offset *after* processing. | You must dedupe / be idempotent. **This is the sane default and what this repo relies on.** |
| **Exactly-once** | Each record's effect happens exactly once. | Kafka transactions + idempotent producer, *and* the consumer's side effects must be transactional with the offset commit. | Complex; only works cleanly for Kafka-to-Kafka. For "consume Kafka, write to Postgres/Formance/BigQuery," true exactly-once is usually **at-least-once + dedup**. |

**Idempotent producer.** kafkajs and Kafka support an idempotent producer (`enable.idempotence`) so that producer retries don't create duplicate records on the broker even if a network hiccup causes a resend. It solves duplicates *on the write side*. It does **not** solve consumer-side duplicates from rebalances/redeliveries — that's on you.

### 4.6 Replication and durability (the `acks` dial)

Each partition has a **replication factor**: how many brokers keep a copy. One replica is the **leader** (serves all reads/writes); the others are **followers** copying it. If the leader broker dies, a follower is promoted. The set of replicas currently caught up is the **ISR (in-sync replicas)**.

Two dials control how durable a write is:

- **Producer `acks`:**
  - `acks=0` — fire-and-forget, don't wait. Fastest, can lose data.
  - `acks=1` — wait for the leader to write. Loses data if the leader dies before followers copy it.
  - `acks=all` (a.k.a. `-1`) — wait until **all in-sync replicas** have the record. Safest.
- **Broker/topic `min.insync.replicas`:** with `acks=all`, the write only succeeds if at least this many replicas are in-sync. If fewer are available, the producer gets an error rather than silently under-replicating.

**This repo's production settings** (`infra/k8s/infrastructure/kafka/kafka.yaml` and `topics.yaml`): `default.replication.factor: 3`, `min.insync.replicas: 2` on a 3-broker cluster. Translation: *a record is only acknowledged when at least 2 of 3 brokers have it; the cluster tolerates losing 1 broker with zero data loss and continued writes.* Locally everything is replication factor **1** (single broker) — fine for dev, never for prod.

### 4.7 Retention and compaction

Records don't live forever. Two policies:

- **Time/size retention (default):** keep records for e.g. 7 days or until the partition hits a size cap, then delete the oldest. Replay is possible **within the window.**
- **Log compaction:** instead of deleting by age, keep the **latest record for each key** and garbage-collect older records with the same key. The topic becomes a compacted "table" of current state per key — perfect for "latest state of row 12345." (Debezium CDC topics are frequently compaction-friendly because each record is keyed by primary key.)

### 4.8 Why schemas (Avro) prevent breakage — and the Schema Registry

Kafka moves **raw bytes.** It has no idea whether your value is JSON, Avro, a protobuf, or a cat photo. That's flexible and terrifying: a producer can change the shape of its messages and every consumer silently breaks at 3 a.m.

**Avro** fixes the shape problem: messages are serialized against an explicit **schema** (field names, types, defaults). But you don't want to ship the whole schema in every message (huge waste). Enter the **Confluent Schema Registry**:

1. The producer registers its Avro schema with the Registry and gets back a small integer **schema ID**.
2. Each message on the wire is: `[magic byte 0x00][4-byte schema ID][Avro-encoded payload]` — the **Confluent wire format**.
3. The consumer reads the schema ID, fetches that schema from the Registry (cached), and decodes.

The Registry also **enforces compatibility rules** (§8): a new schema version must be, say, *backward compatible* with the old one, or the Registry rejects it. That is how you evolve message shapes without breaking live consumers.

In this repo, Debezium serializes CDC rows as **Avro** into the Registry, and the worker decodes them with `AvroKafkaDeserializer` (`packages/kafka/src/avro-deserializer.ts`), which checks for exactly that wire format:

```ts
// Confluent wire format requires ≥5 bytes: 1 magic + 4 schema-id.
if (Buffer.isBuffer(data) && data.length >= 5 && data.readUInt8(0) === 0) {
  data = await this.registry.decode(data);   // fetch schema by id, decode Avro
}
```

The audit topics, by contrast, are **plain JSON** (produced by the API), so they are *not* run through the Avro deserializer — you'll see this split in `apps/kafka-worker/src/main.ts` (§7).

> **One real gotcha baked into the code:** `avro-deserializer.ts` monkey-patches avsc's `LongType._read` so large MySQL `BIGINT` values decode as JS numbers without a "precision loss" error. It's a deliberate workaround; don't "clean it up."

---

## 5. Setup from scratch: a real broker in Docker (KRaft + SASL + Schema Registry + UI)

Time to make it real. We'll build up the exact shape this repo uses, explaining each line. Everything here matches `docker-compose.yml` in the repo root — you can read along there.

### 5.1 The single most confusing thing in Kafka: advertised listeners

Before any YAML, understand **listeners** or you *will* lose an afternoon. This is the #1 Kafka support question on earth.

When a client connects to Kafka, it first hits a **bootstrap** address. The broker then replies with **metadata**: "here is the address to actually reach the leader of each partition." **The client then connects to *that* address.** The address the broker advertises is `advertised.listeners` — and it must be reachable *from wherever the client runs.*

The trap: inside Docker, other containers reach the broker at hostname `kafka` (Docker's internal DNS). But your app running on your **laptop** (outside Docker) reaches it at `localhost`. One advertised address cannot be right for both. The solution is **multiple listeners**, one per network vantage point:

```
                          ┌─────────────────── Docker network ───────────────────┐
  your laptop             │                                                       │
  (host machine)          │   kafka container                                     │
      │                   │   ├─ BROKER   listener  :9092  advertised kafka:9092  │◄── other containers
      │  localhost:9094   │   ├─ CONTROLLER listener :9093 (KRaft, internal only) │    use THIS
      └──────────────────►│   └─ EXTERNAL listener  :9094  advertised localhost:9094 ◄─ your laptop uses THIS
                          └───────────────────────────────────────────────────────┘
```

- **BROKER (`:9092`)** advertised as `kafka:9092` — used by *other containers* (schema-registry, kafka-worker, debezium) and for inter-broker traffic.
- **CONTROLLER (`:9093`)** — KRaft metadata quorum; internal, plaintext, never for clients.
- **EXTERNAL (`:9094`)** advertised as `localhost:9094` — the port published to your host, used by tools on your laptop and by the API in local dev (`KAFKA_BROKER=localhost:9094`).

If a client "connects but then times out / can't find the topic-partition," 90% of the time it's an advertised-listener mismatch: the client reached bootstrap, got told to go to an address it can't route to, and quietly failed.

### 5.2 The Kafka service (KRaft combined mode, SASL/PLAIN)

Here is the repo's `kafka` service, annotated. **KRaft combined mode** means one process plays **both** `broker` and `controller` roles — no ZooKeeper, no second container.

```yaml
kafka:
  image: confluentinc/cp-kafka:7.8.0
  hostname: kafka
  ports:
    - "9094:9094"                       # publish only the EXTERNAL listener to the host
  environment:
    CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk  # REQUIRED in KRaft; a stable base64 UUID for this cluster
    KAFKA_NODE_ID: 1
    KAFKA_PROCESS_ROLES: broker,controller           # combined mode
    KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093      # the (single-node) Raft quorum

    # ── the three listeners from §5.1 ──
    KAFKA_LISTENERS: BROKER://:9092,CONTROLLER://:9093,EXTERNAL://:9094
    KAFKA_ADVERTISED_LISTENERS: BROKER://kafka:9092,EXTERNAL://localhost:9094
    KAFKA_INTER_BROKER_LISTENER_NAME: BROKER
    KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER

    # ── security per listener ──
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,BROKER:SASL_PLAINTEXT,EXTERNAL:SASL_PLAINTEXT
    KAFKA_SASL_ENABLED_MECHANISMS: PLAIN
    KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL: PLAIN
    # admin identity the broker uses for inter-broker auth:
    KAFKA_SASL_JAAS_CONFIG: >-
      org.apache.kafka.common.security.plain.PlainLoginModule required
      username="admin" password="${KAFKA_ADMIN_PASSWORD}";
    # the user database for the BROKER listener (user_<name>="<password>"):
    KAFKA_LISTENER_NAME_BROKER_PLAIN_SASL_JAAS_CONFIG: >-
      org.apache.kafka.common.security.plain.PlainLoginModule required
      username="admin" password="${KAFKA_ADMIN_PASSWORD}"
      user_admin="${KAFKA_ADMIN_PASSWORD}"
      user_connect="${KAFKA_CONNECT_PASSWORD}"
      user_schema="${KAFKA_SCHEMA_PASSWORD}"
      user_worker="${KAFKA_WORKER_PASSWORD}";
    # same block again for the EXTERNAL listener (KAFKA_LISTENER_NAME_EXTERNAL_PLAIN_SASL_JAAS_CONFIG)

    # ── single-node durability + defaults ──
    KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
    KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0     # don't wait to rebalance in dev
    KAFKA_NUM_PARTITIONS: 3                        # default partitions for auto-created topics
  volumes:
    - kafka_data:/var/lib/kafka/data
```

Understanding the **JAAS** (Java Authentication and Authorization Service) config is worth 30 seconds: `user_worker="secret"` declares a login named `worker` with password `secret`. The `PlainLoginModule required username="admin" ...` line at the top sets the *broker's own* identity for talking to other brokers. Four users exist in this repo: **admin** (broker + admin tools), **connect** (Debezium), **schema** (Schema Registry), **worker** (the kafka-worker). Least privilege by identity.

> The passwords come from your `.env` (`KAFKA_ADMIN_PASSWORD`, `KAFKA_CONNECT_PASSWORD`, `KAFKA_SCHEMA_PASSWORD`, `KAFKA_WORKER_PASSWORD`). Never commit them. See `.env.example` for the non-secret Kafka vars (`KAFKA_BROKER`, `KAFKA_PORT=9094`, `KAFKA_GROUP_ID=kafka-worker-group`).

### 5.3 Add the Schema Registry

The Registry is itself a Kafka client (it stores schemas *in* a Kafka topic), so it authenticates with the `schema` user and talks to the broker over the internal `kafka:9092` listener:

```yaml
schema-registry:
  image: confluentinc/cp-schema-registry:7.6.0
  ports: [ "8081:8081" ]
  depends_on: { kafka: { condition: service_healthy } }
  environment:
    SCHEMA_REGISTRY_HOST_NAME: schema-registry
    SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092   # internal listener
    SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    SCHEMA_REGISTRY_KAFKASTORE_SECURITY_PROTOCOL: SASL_PLAINTEXT
    SCHEMA_REGISTRY_KAFKASTORE_SASL_MECHANISM: PLAIN
    SCHEMA_REGISTRY_KAFKASTORE_SASL_JAAS_CONFIG: >-
      org.apache.kafka.common.security.plain.PlainLoginModule required
      username="schema" password="${KAFKA_SCHEMA_PASSWORD}";
```

The Registry's HTTP API is now on `http://localhost:8081` from your host (and `http://schema-registry:8081` from other containers — which is what `SCHEMA_REGISTRY_URL` is set to for the worker).

### 5.4 Add a UI

`provectuslabs/kafka-ui` is a browser dashboard for topics, messages, consumer-group lag, and schemas. Invaluable for debugging.

```yaml
kafka-ui:
  image: provectuslabs/kafka-ui:latest
  ports: [ "8090:8080" ]              # → http://localhost:8090
  environment:
    KAFKA_CLUSTERS_0_NAME: local
    KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
    KAFKA_CLUSTERS_0_PROPERTIES_SECURITY_PROTOCOL: SASL_PLAINTEXT
    KAFKA_CLUSTERS_0_PROPERTIES_SASL_MECHANISM: PLAIN
    KAFKA_CLUSTERS_0_PROPERTIES_SASL_JAAS_CONFIG: >-
      org.apache.kafka.common.security.plain.PlainLoginModule required
      username="admin" password="${KAFKA_ADMIN_PASSWORD}";
```

### 5.5 Bring it up and verify

```bash
# from repo root, with a populated .env
docker compose up -d kafka schema-registry kafka-ui

# is the broker healthy?
docker compose ps kafka
docker compose logs -f kafka        # look for "Kafka Server started"
```

Open `http://localhost:8090` — you should see the `local` cluster. Open `http://localhost:8081/subjects` — the Registry answers (`[]` if empty). If both respond, the stack is alive.

### 5.6 Talk to it from the CLI (with SASL)

The CLI tools live *inside* the `kafka` container. Because the broker requires SASL, you must hand the tools a small client config file. Create it once inside the container:

```bash
# open a shell in the broker container
docker compose exec kafka bash

# write a client props file (uses the admin user)
cat > /tmp/client.properties <<'EOF'
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="REPLACE_WITH_KAFKA_ADMIN_PASSWORD";
EOF
```

Now the essentials — **create, list, describe, produce, consume**:

```bash
BS=localhost:9092   # inside the container, use the BROKER listener

# create a topic with 3 partitions
kafka-topics --bootstrap-server $BS --command-config /tmp/client.properties \
  --create --topic demo.hello --partitions 3 --replication-factor 1

# list all topics
kafka-topics --bootstrap-server $BS --command-config /tmp/client.properties --list

# describe a topic (partitions, leader, replicas, ISR)
kafka-topics --bootstrap-server $BS --command-config /tmp/client.properties \
  --describe --topic demo.hello

# PRODUCE — type lines, each becomes a record; Ctrl-D to stop
kafka-console-producer --bootstrap-server $BS --producer.config /tmp/client.properties \
  --topic demo.hello

# CONSUME — read from the very beginning
kafka-console-consumer --bootstrap-server $BS --consumer.config /tmp/client.properties \
  --topic demo.hello --from-beginning
```

Produce a few lines in one terminal, run the consumer in another, and watch them appear. To prove keys route to partitions, produce with keys:

```bash
kafka-console-producer --bootstrap-server $BS --producer.config /tmp/client.properties \
  --topic demo.hello --property parse.key=true --property key.separator=:
# then type:  vendor-42:hello
#             vendor-42:world     ← same key → same partition → ordered
```

You've now stood up an authenticated, KRaft-mode Kafka with a Schema Registry and a UI, created topics, and moved real messages. That is the whole hard part of "adding Kafka to any project."

---

## 6. Using it in code (Node / NestJS)

Now the application layer. Everything below uses **kafkajs 2.2.4** (the raw client), **@nestjs/microservices 11.1.16** (the NestJS consumer transport), and **@kafkajs/confluent-schema-registry 4.0.8** + **avsc 5.7.9** (Avro). These are the exact versions in this repo.

### 6.1 A producer with kafkajs (connect, key, error handling, best-effort)

```ts
import { Kafka, logLevel } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'my-service',           // shows up in broker logs; helps debugging
  brokers: ['localhost:9094'],      // EXTERNAL listener from the host
  logLevel: logLevel.INFO,
  sasl: {                           // required because the broker enforces SASL
    mechanism: 'plain',
    username: 'worker',
    password: process.env.KAFKA_SASL_PASSWORD!,
  },
  // ssl: true,                     // add in real prod (SASL_SSL)
});

const producer = kafka.producer();  // add { idempotent: true } for producer-side dedupe

async function start() {
  await producer.connect();
}

async function publishBillPaid(billId: string, payload: object) {
  await producer.send({
    topic: 'bill.paid',
    // KEY = billId ⇒ every event for one bill lands on the same partition ⇒ ordered
    messages: [{ key: billId, value: JSON.stringify(payload) }],
    acks: -1,                        // wait for all in-sync replicas (durable)
  });
}
```

**Best-effort / optional publishing.** Sometimes an event is *nice to have*, and you'd rather your request succeed than fail because Kafka blinked. In this repo the API's audit publishing is exactly this: it must never break the user's actual operation. The pattern is to catch, log once, and move on — which is precisely what `KafkaProducerService` does (§7).

### 6.2 A consumer / consumer group with kafkajs

```ts
const consumer = kafka.consumer({ groupId: 'bill-projector' });

await consumer.connect();
await consumer.subscribe({ topic: 'bill.paid', fromBeginning: true });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const key = message.key?.toString();
    const value = JSON.parse(message.value!.toString());
    await handleBillPaid(key, value);   // ← MUST be idempotent (see 6.5)
    // kafkajs auto-commits offsets after eachMessage returns successfully.
  },
});
```

Change the `groupId` and you get a *second, independent* reader of the same topic (the §4.3 superpower). Reuse the same `groupId` on more instances and they *share* the partitions (scaling).

### 6.3 Avro serialize / deserialize with the Schema Registry

```ts
import { SchemaRegistry } from '@kafkajs/confluent-schema-registry';

const registry = new SchemaRegistry({ host: 'http://localhost:8081' });

// PRODUCE Avro: register schema once, encode by id
const schema = `{
  "type": "record", "name": "BillPaid",
  "fields": [{ "name": "billId", "type": "string" }, { "name": "amount", "type": "long" }]
}`;
const { id } = await registry.register({ type: 'AVRO', schema });
const value = await registry.encode(id, { billId: 'B-1', amount: 12345 });
await producer.send({ topic: 'bill.paid.avro', messages: [{ value }] });

// CONSUME Avro: decode looks up the schema id embedded in the bytes
const decoded = await registry.decode(message.value!);  // → { billId, amount }
```

That `registry.decode(...)` call is the core of this repo's `AvroKafkaDeserializer` (§4.8).

### 6.4 The NestJS microservice consumer pattern

NestJS wraps kafkajs so a class method becomes a message handler. You attach a Kafka transport per consumer group and decorate handlers with the topic name:

```ts
// bootstrap: attach ONE transport per consumer group
app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.KAFKA,
  options: {
    client:   { clientId: 'worker-bills', brokers: ['kafka:9092'], sasl: {/*…*/} },
    consumer: { groupId: 'bills-sync-group' },
    subscribe: { fromBeginning: true },
    // deserializer: new AvroKafkaDeserializer('http://schema-registry:8081'),
  },
});
await app.startAllMicroservices();
```

```ts
// a handler: @MessagePattern(topic) turns this method into a consumer for that topic
@Controller()
export class BillsConsumer {
  @MessagePattern(KAFKA_TOPICS.CDC_REMOTE_BILLS)
  async handle(@Payload() raw: CdcEventPayload<GibpBillRow>, @Ctx() ctx: KafkaContext) {
    // ctx.getTopic(), ctx.getPartition(), ctx.getMessage(), ctx.getHeartbeat()
    // …process idempotently…
  }
}
```

### 6.5 Idempotent consumers — the rule you cannot skip

Because delivery is **at-least-once**, the *same record can arrive more than once* (rebalance, redelivery, retry). Your handler must make reprocessing a no-op. Common strategies:

- **Natural idempotency:** `UPSERT` by primary key instead of `INSERT`. Re-processing just overwrites with the same values.
- **Dedup table / idempotency key:** record a unique message identifier (e.g. `topic+partition+offset`, or a business `idempotency_key`) in a `processed_events` table inside the same DB transaction as your side effect; skip if already present.
- **Conditional writes:** "only apply if version > current."

> This repo leans on natural idempotency in the sync handlers (upserts keyed by the source row's primary key), and on `fromBeginning: true` being safe *because* the handlers tolerate re-seeing Debezium's initial snapshot on first boot.

### 6.6 The dead-letter-queue (DLQ) pattern for poison messages

A **poison message** is one that will *never* process successfully — malformed payload, a violated invariant, a bug. If you keep retrying it, it blocks its entire partition forever (head-of-line blocking): every later record behind it is stuck. The fix: after bounded retries, **move it aside** to a DLQ topic and keep going.

Generic shape:

```ts
try {
  await process(record);
} catch (err) {
  if (isUnrecoverable(err) || attempts >= MAX) {
    await producer.send({
      topic: 'my-app.dlq',
      messages: [{ value: JSON.stringify({ originalTopic, payload: record, error: err.message, failedAt: new Date().toISOString() }) }],
    });
    // commit/advance past it so the partition keeps moving
  } else {
    await backoffAndRetry();
  }
}
```

This is exactly the repo's `DlqService` (§7). The DLQ is then a triage queue: an operator inspects it, fixes the root cause, and can replay the parked messages.

---

## 7. How THIS repo uses Kafka

This is the section you'll return to. Everything Kafka-related is centralized in one shared library and consumed by two apps.

### 7.1 The shared library: `packages/kafka`

| File | What it provides |
|---|---|
| `kafka.config.ts` | The `KafkaModuleOptions` interface: `{ brokers: string[]; clientId: string; optional?: boolean; sasl?: { mechanism: 'plain'; username; password } }`. |
| `kafka.module.ts` | `KafkaModule` — a NestJS `DynamicModule` with `register()` and `registerAsync()`. Builds the kafkajs `Kafka` client (`createKafkaClient`) and exports `KafkaProducerService`. |
| `kafka-producer.service.ts` | `KafkaProducerService` — a Nest `@Injectable` that connects on init, disconnects on destroy, and exposes `produce({ topic, messages })`. Honors `optional`. |
| `kafka.constants.ts` | DI tokens `KAFKA_CLIENT` / `KAFKA_MODULE_OPTIONS`, and `KAFKA_CONSUMER_GROUPS` (the eight group IDs). |
| `kafka.topics.ts` | `KAFKA_TOPICS` — the single source of truth for every topic name, and the `KafkaTopic` type. |
| `avro-deserializer.ts` | `AvroKafkaDeserializer` — Nest `Deserializer` that decodes Confluent-wire Avro via the Schema Registry (with the avsc `LongType` patch). |
| `cdc.types.ts` | `CdcEventPayload<Row>`, `DebeziumOperation`, `DebeziumSource` — the shape of a Debezium CDC message. |

**`KafkaProducerService` — the two behaviors that matter.** It centralizes connect/disconnect and, critically, the **best-effort mode**. When `optional: true`, a broker that's down or a failed publish is caught, logged **once**, and swallowed (returns `[]`) instead of throwing:

```ts
async produce(options: ProduceOptions): Promise<RecordMetadata[]> {
  if (!this.isConnected) {
    if (this.isOptional) { this.logUnavailable(/* … */); return []; }
    throw new Error('Kafka producer is not connected');
  }
  try {
    return await this.producer.send({ topic: options.topic, messages: options.messages });
  } catch (error) {
    this.isConnected = false;
    if (!this.isOptional) throw error;
    this.logUnavailable(/* … */); return [];
  }
}
```

The kafkajs client also gets a small bounded retry (`retries: 3`) in optional mode — deliberately *not* `retries: 0` — to survive the common "This server does not host this topic-partition" race on first publish to an auto-created topic (see the comment in `kafka.module.ts`).

### 7.2 The topics

All names come from `kafka.topics.ts`. The CDC topic prefix is `gibp04-cdc.<database>.<table>` where `<database>` is `CDC_MYSQL_DATABASE` (default `gibp04`).

```
CDC (Debezium → worker)                          consumed by
────────────────────────────────────────────    ─────────────────────────────
gibp04-cdc.gibp04.gibp_transaction_schedules  ┐
gibp04-cdc.gibp04.gibp_transactions           ├─ ACH batches
gibp04-cdc.gibp04.gibp_payment_batches        ┐
gibp04-cdc.gibp04.gibp_payment_batch_currency_totals    ├─ batches-sync-group
gibp04-cdc.gibp04.gibp_payment_batch_currency_entries   ┘
gibp04-cdc.gibp04.gibp_bills                  ┐
gibp04-cdc.gibp04.gibp_account_service_signups├─ bills-sync-group
gibp04-cdc.gibp04.gibp_vendor_services        ┘
gibp04-cdc.gibp04.gibp_accounts               ─ accounts-sync-group
gibp04-cdc.gibp04.gibp_vendors                ─ vendors-sync-group
gibp04-cdc.gibp04.gibp_vendor_services
gibp04-cdc.gibp04.gibp_entities_general
gibp04-cdc.gibp04.gibp_account_transactions
gibp04-cdc.dlq                                ─ dead-letter queue (poison messages)

Audit (api → worker → BigQuery)
────────────────────────────────────────────
audit-business-events    keyed by correlation_id          ─ audit-bq-sink-business-events
audit-ledger-postings    keyed by formance_transaction_id ─ audit-bq-sink-ledger-postings
audit-activity-events    keyed by actor/subject           ─ audit-bq-sink-activity-events

Local-dev / test only
────────────────────────────────────────────
mysql-server.gibp_cdc.users, .orders   (local Debezium demo)
test.events
```

### 7.3 The producer side — `apps/api`

The API imports `KafkaModule.registerAsync(...)` in its **audit module** (`apps/api/src/modules/audit/audit.module.ts`) with `clientId: 'api-audit'`, `brokers: [ KAFKA_BROKER ]`, `optional: true`, and SASL when configured. Two services publish:

- `AuditEmitterService` → `audit-business-events` (keyed by `correlation_id`) and `audit-ledger-postings` (keyed by `formance_transaction_id`).
- `ActivityAuditService` → `audit-activity-events`.

Because the module is `optional: true`, if Kafka is unreachable the *business operation still succeeds* — audit publishing is best-effort by design.

> **The one hard requirement:** even though publishing is best-effort, the API's config reads `KAFKA_BROKER` via `getOrThrow`, so the **`gibp-kafka-broker` secret must exist or the api pod won't start.** "Optional to publish" ≠ "optional to configure."

### 7.4 The consumer side — `apps/kafka-worker`

`apps/kafka-worker/src/main.ts` builds a **NestJS hybrid app** and attaches **one Kafka microservice transport per consumer group** (§4.3), so each group tracks its offset independently and a stall in one never blocks another:

- SASL is assembled by `buildSasl()` from `KAFKA_SASL_MECHANISM` / `_USERNAME` / `_PASSWORD` (username defaults to `worker`).
- `subscribe: { fromBeginning: true }` — on a *fresh* group with no committed offset, read from offset 0 so Debezium's initial snapshot is fully processed. On later restarts the committed offset wins, so there's no double-processing.
- The **Avro deserializer** is attached only to the CDC/sync groups (Debezium publishes Avro). The three **audit sink** groups are attached *without* it, because audit messages are **plain JSON** produced by the API.

The consumers live under `apps/kafka-worker/src/modules/gibp-sync/consumers/*` (e.g. `vendors.consumer.ts`, `bills.consumer.ts`, `accounts.consumer.ts`, `payment-batches.consumer.ts`, `ach-batches.consumer.ts`). Each uses `@MessagePattern(KAFKA_TOPICS.*)` and a `withRetry(...)` loop:

- `DependencyNotReadyError` → bounded exponential-backoff retry (a related row hasn't synced yet); after max attempts, **skip** (the cascade will reprocess) — deliberately *not* DLQ'd, so it isn't lost.
- `UnrecoverableEventError` (or unexpected error after max retries) → `DlqService.publish(topic, raw, error)` → parks the message on `gibp04-cdc.dlq` with `{ originalTopic, error, payload, failedAt }`.

`DlqService` (`.../gibp-sync/retry/dlq.service.ts`) uses the shared `KafkaProducerService` and even wraps *its own* publish in try/catch so a DLQ failure can't crash the consumer.

### 7.5 Local vs production topology

| | Local (`docker-compose.yml`) | Production |
|---|---|---|
| Broker | one `confluentinc/cp-kafka:7.8.0`, KRaft combined | **Strimzi**-operated 3-node cluster (`kafka.yaml`) |
| Auth | SASL_PLAINTEXT / PLAIN, 4 users | SASL (+ TLS) managed by Strimzi |
| Replication | factor **1** | factor **3**, `min.insync.replicas: 2` |
| Broker address | `localhost:9094` (host) / `kafka:9092` (containers) | injected from Secret Manager (`gibp-kafka-broker`) |
| Schema Registry | `confluentinc/cp-schema-registry:7.6.0` on 8081 | provisioned by infra |
| UI | `provectuslabs/kafka-ui` on 8090 | — |
| Topic creation | auto-create (`KAFKA_NUM_PARTITIONS: 3`) | declared as `KafkaTopic` CRDs (`topics.yaml`) |

### 7.6 Production: Strimzi on Kubernetes

In production Kafka is **not** a container you run — it's managed by the **Strimzi operator**, a Kubernetes operator that turns Kafka concepts into Kubernetes custom resources (CRDs). Manifests live under `infra/k8s/infrastructure/kafka/`:

- `strimzi-operator-app.yaml` (in `infra/k8s/argocd/`) installs the Strimzi Helm chart (`0.51.0`) via **ArgoCD**, watching the `kafka-system` namespace.
- `kafka.yaml` declares a `Kafka` + `KafkaNodePool`: **3 replicas**, each node both `controller` and `broker` (**KRaft**, `strimzi.io/kraft: enabled`), pod anti-affinity so brokers land on different nodes, `default.replication.factor: 3`, `min.insync.replicas: 2`. The `entityOperator` enables the **topic** and **user** operators.
- `topics.yaml` declares topics as `KafkaTopic` CRDs (`user.created`, `ledger.updated`, `payment.processed` — each `partitions: 3, replicas: 3, min.insync.replicas: 2`). You create/change a topic by editing YAML and letting ArgoCD sync it — GitOps, not `kafka-topics --create`.
- `storage-class.yaml` — a `pd-ssd` GKE storage class with `reclaimPolicy: Retain` (so a deleted claim does **not** delete the disk — data safety).
- `external-secret.yaml` — pulls Debezium DB credentials from GCP Secret Manager into the cluster via the External Secrets operator.

The main **producer of the CDC topics in production is Debezium** (streaming `gibp04` MySQL changes). That machinery is its own chapter → [`04-debezium-and-cdc.md`](./04-debezium-and-cdc.md). For the end-to-end deploy story see [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md).

---

## 8. Production concerns

- **Partition count = your throughput/parallelism ceiling.** A topic's max in-group concurrency equals its partition count. Size for peak: estimate target messages/sec ÷ per-consumer throughput, round up, add headroom. **You can add partitions but never remove them**, and adding them **breaks key→partition stability** (§10) — so slightly over-provision at creation rather than reshard later.
- **Replication factor & `min.insync.replicas`.** RF=3 + `min.insync.replicas=2` + producer `acks=all` is the standard durable triad: survive one broker loss with zero data loss and continued writes. This repo's prod uses exactly that. RF=1 (local) tolerates no failures — dev only.
- **Consumer lag monitoring.** *Lag* = (latest offset produced) − (consumer's committed offset) per partition, i.e. how far behind a consumer is. Growing lag = consumers can't keep up (add consumers up to the partition count, or make handlers faster). Watch it in kafka-ui, via `kafka-consumer-groups --describe`, or export to your metrics stack. Lag is the single most important Kafka health metric.
- **Rebalance storms.** If consumers repeatedly join/leave (crash loops, too-short `session.timeout.ms`, long handlers that miss heartbeats), the group thrashes and throughput collapses. Mitigations: keep handlers fast (or call `heartbeat()` during long work — the repo's consumers thread `ctx.getHeartbeat()` through retries), tune session/heartbeat timeouts, use cooperative-sticky rebalancing, and don't over-scale past the partition count (extra consumers just sit idle and add churn).
- **Exactly-once vs idempotent+dedup.** True EOS only holds cleanly Kafka-to-Kafka. Since this pipeline writes to Postgres/Formance/BigQuery, the honest posture is **at-least-once + idempotent consumers** (§6.5). Don't promise exactly-once you can't deliver.
- **Schema evolution / compatibility.** The Registry enforces a compatibility mode per subject: **BACKWARD** (new consumers read old data — the common default; you may add optional fields / remove fields), **FORWARD** (old consumers read new data), **FULL** (both). Break the rule and the Registry rejects the new schema at register time — which is the point: you find out at deploy, not at 3 a.m. Adding a field with a **default** is the safe, backward-compatible move.
- **Security.** Local is SASL_**PLAINTEXT** (auth but no encryption — fine on a laptop). Real prod must add **TLS** (SASL_**SSL**) so credentials and payloads aren't on the wire in the clear. Use per-service identities (this repo already does: admin/connect/schema/worker) and least-privilege ACLs.
- **Sizing / retention / cost.** Disk = Σ over topics of (throughput × retention × replication factor). Longer retention and higher RF cost linearly more disk. Set retention to the largest replay window you actually need, not "forever." Prod storage here is 20Gi `pd-ssd` per broker with `Retain` reclaim.
- **Strimzi operations.** Broker config, topics, users, and storage are all **declarative CRDs** synced by ArgoCD. Scaling, rolling upgrades, and cert rotation are the operator's job — you change YAML, not brokers. Because the storage class is `Retain`, deleting a `KafkaNodePool` claim keeps the underlying disk; reclaiming it is a deliberate manual step.

---

## 9. Operations & debugging playbooks

### 9.1 CLI cheat sheet

Run inside the broker container with a SASL client-props file (see §5.6). Locally you can also point kafka-ui at everything (`http://localhost:8090`).

```bash
BS=localhost:9092; CFG=/tmp/client.properties

# topics
kafka-topics --bootstrap-server $BS --command-config $CFG --list
kafka-topics --bootstrap-server $BS --command-config $CFG --describe --topic <t>
kafka-topics --bootstrap-server $BS --command-config $CFG --create --topic <t> --partitions 3 --replication-factor 1
kafka-topics --bootstrap-server $BS --command-config $CFG --alter  --topic <t> --partitions 6   # ⚠ breaks key routing

# consumer groups & LAG (the money command)
kafka-consumer-groups --bootstrap-server $BS --command-config $CFG --list
kafka-consumer-groups --bootstrap-server $BS --command-config $CFG --describe --group bills-sync-group
#   → per-partition CURRENT-OFFSET, LOG-END-OFFSET, LAG

# console produce / consume (add --property print.key=true to see keys)
kafka-console-producer --bootstrap-server $BS --producer.config $CFG --topic <t>
kafka-console-consumer --bootstrap-server $BS --consumer.config $CFG --topic <t> --from-beginning --property print.key=true

# schema registry (HTTP)
curl -s http://localhost:8081/subjects
curl -s http://localhost:8081/subjects/<subject>-value/versions/latest | jq .
```

### 9.2 "My consumer isn't receiving messages"

Work down this list — the cause is almost always one of these:
1. **Advertised listeners (§5.1).** Is the consumer inside Docker (use `kafka:9092`) or on the host (use `localhost:9094`)? A mismatch = connects to bootstrap, then silently can't route. #1 cause.
2. **Wrong topic name.** Typos, or the CDC prefix/database differs (`CDC_MYSQL_DATABASE`). Compare against `kafka-topics --list`.
3. **Offset already past the data.** A group with a committed offset and `fromBeginning: false` won't see old messages. Check with `kafka-consumer-groups --describe`; reset if needed (§9.5).
4. **Group has more consumers than partitions.** Extras sit idle by design (§4.3). Not a bug.
5. **Stuck in a rebalance** (crash loop / handler timeouts) — the group never stabilizes. Check worker logs for repeated "Rebalancing."
6. **Producer never actually produced.** In this repo, `optional: true` means a failed publish is swallowed with a single warn log — check for "Kafka unavailable" / "publish failed" lines.

### 9.3 "Messages are processed out of order"

Order is only guaranteed **within a partition** (§4.1). If related events are landing on different partitions, they were produced with **different keys or a null key**. Fix: key the related events by the same value (this repo keys by `correlation_id` / `formance_transaction_id`). And remember: **adding partitions later** re-routes existing keys (§10) and can interleave old vs new.

### 9.4 "A poison message is stuck / a partition stopped advancing"

One record throws forever → head-of-line blocking for its whole partition. Confirm via lag on one partition frozen while others move, and consumer logs looping on the same offset. The system should DLQ it (§7.4). If a message got stuck *before* DLQ logic existed, inspect it (`kafka-console-consumer --partition P --offset N --max-messages 1`), fix the handler, and either let retries pass it or manually advance the offset past it.

### 9.5 Reset a consumer group's offsets (replay or skip)

The group must have **no active members** (stop the worker first).

```bash
G=bills-sync-group; T=gibp04-cdc.gibp04.gibp_bills
# replay everything:
kafka-consumer-groups --bootstrap-server $BS --command-config $CFG \
  --group $G --topic $T --reset-offsets --to-earliest --execute
# skip to newest (drop the backlog):
kafka-consumer-groups --bootstrap-server $BS --command-config $CFG \
  --group $G --topic $T --reset-offsets --to-latest --execute
# to a point in time:
#   --reset-offsets --to-datetime 2026-07-01T00:00:00.000
```

Replaying is only safe because the consumers are **idempotent** (§6.5) — reprocessing must not double-apply effects.

### 9.6 Inspect the Schema Registry

```bash
curl -s http://localhost:8081/subjects                                   # all subjects
curl -s http://localhost:8081/subjects/<subject>-value/versions          # versions
curl -s http://localhost:8081/subjects/<subject>-value/versions/latest    # latest schema + id
curl -s http://localhost:8081/config                                      # global compatibility mode
```

If Avro decode fails in the worker, `AvroKafkaDeserializer` logs `Avro decode failed for topic=…` and rethrows — usually a schema-id the Registry can't resolve (Registry down / wrong `SCHEMA_REGISTRY_URL`) or a non-Avro message reaching an Avro consumer.

---

## 10. Gotchas & hard-won lessons

- **Advertised listeners inside vs outside Docker.** The single biggest time-sink. Containers use `kafka:9092`; your laptop uses `localhost:9094`. The broker advertises *both* via separate listeners. A client that "connects then hangs/can't find partitions" is almost always talking to an address it can't route to. (§5.1)
- **Ordering is per-partition only.** There is no global order. Related events must share a key to share a partition. "Why did the delete get processed before the insert?" → they had different keys.
- **Adding partitions breaks key→partition stability.** Routing is `hash(key) mod N`. Change `N` and existing keys move to different partitions, so a key's history can split across old and new partitions and briefly reorder. Size partitions generously **up front**; treat repartitioning as a migration, not a config tweak.
- **Consumers MUST be idempotent.** At-least-once means duplicates are normal (rebalances, retries, replays, `fromBeginning` snapshots). Upsert, dedupe, or use idempotency keys. This is non-negotiable in a financial system.
- **Auto-commit pitfalls.** Background auto-commit can mark a record "done" before your handler finished — crash in between and you skip it (loss) or reprocess it (duplicate). Know exactly when your framework commits; prefer committing after processing.
- **Schema-registry backward compatibility is a feature, not an obstacle.** When a schema change is rejected, that's the Registry stopping you from breaking live consumers. Add fields with defaults; don't repurpose or retype existing fields. Pick a compatibility mode deliberately (§8).
- **SASL vs plaintext confusion.** This repo's broker enforces **SASL** on both client listeners. A client without `sasl` config connects, completes the metadata handshake, then gets its **connection closed** with cryptic errors. If a brand-new client "can't stay connected," check SASL first. (Comment to this effect lives in `kafka.module.ts`.)
- **"Optional to publish" ≠ "optional to configure."** The API tolerates Kafka being *down* at publish time (best-effort), but still `getOrThrow`s `KAFKA_BROKER` at boot — the `gibp-kafka-broker` secret must exist or the pod won't start.
- **`fromBeginning: true` is safe only because handlers are idempotent.** On a fresh consumer group it replays the entire Debezium snapshot. If your handlers weren't idempotent, first boot would double-apply everything.
- **The avsc `LongType` patch is load-bearing.** Large MySQL `BIGINT`s would otherwise throw "precision loss." The monkey-patch in `avro-deserializer.ts` is intentional — don't remove it.

---

## 11. Glossary & further reading

### Quick glossary

- **Broker** — one Kafka server. **Cluster** — many brokers.
- **Topic** — a named append-only stream. **Partition** — one ordered log inside a topic; unit of parallelism and ordering. **Offset** — a record's position in a partition.
- **Record** — key + value + timestamp + headers. **Key** — routes a record to a partition (`hash(key) mod N`).
- **Producer / Consumer** — write / read clients. **Consumer group** — consumers sharing a `group.id`; partitions split among them; different group.id = independent bookmark.
- **Rebalance** — reassigning partitions when group membership changes. **Committed offset** — where a group resumes.
- **Replication factor** — copies per partition. **Leader/follower** — the active replica vs its hot standbys. **ISR** — replicas currently caught up. **`acks` / `min.insync.replicas`** — the durability dials.
- **Retention** — age/size deletion policy. **Compaction** — keep only the latest record per key.
- **Controller** — cluster metadata coordinator. **KRaft** — Kafka's built-in Raft metadata (replaces **ZooKeeper**).
- **Schema Registry** — versioned schema store handing out schema IDs. **Avro** — compact schema'd serialization. **Confluent wire format** — `magic byte + 4-byte schema id + payload`.
- **SASL/PLAIN** — username+password auth. **TLS / SASL_SSL** — encryption in transit.
- **DLQ** — dead-letter topic for poison messages. **Lag** — how far a consumer is behind the log end. **Idempotent** — reprocessing is harmless.
- **Strimzi** — the Kubernetes operator that runs Kafka via CRDs. **Debezium** — the CDC tool that produces this repo's `gibp04-cdc.*` topics.

### Further reading

- Apache Kafka documentation — https://kafka.apache.org/documentation/
- Kafka design & the log — https://kafka.apache.org/documentation/#design
- KRaft (no ZooKeeper) — https://kafka.apache.org/documentation/#kraft
- kafkajs (the client this repo uses) — https://kafka.js.org/
- NestJS Kafka microservice transport — https://docs.nestjs.com/microservices/kafka
- Confluent Schema Registry — https://docs.confluent.io/platform/current/schema-registry/index.html
- `@kafkajs/confluent-schema-registry` — https://github.com/kafkajs/confluent-schema-registry
- Avro specification — https://avro.apache.org/docs/current/specification/
- Strimzi (Kafka on Kubernetes) — https://strimzi.io/docs/operators/latest/overview
- Debezium (change data capture) — https://debezium.io/documentation/ — and this handbook's [`04-debezium-and-cdc.md`](./04-debezium-and-cdc.md)

---

*In-repo cross-references:* `packages/kafka/src/*` (shared library), `apps/api/src/modules/audit/*` (producers), `apps/kafka-worker/src/main.ts` + `apps/kafka-worker/src/modules/gibp-sync/*` (consumers, retry, DLQ), `docker-compose.yml` (local stack), `infra/k8s/infrastructure/kafka/*` and `infra/k8s/argocd/strimzi-operator-app.yaml` (production Strimzi), `.env.example` (Kafka env vars).
