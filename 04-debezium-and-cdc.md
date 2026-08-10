# Chapter 04 — Debezium & Change Data Capture (CDC)

> **What this chapter is.** A from-zero, self-contained guide to **Change Data
> Capture (CDC)** with **Debezium** over **Kafka Connect**. It teaches CDC as a
> concept (assuming you have never heard the term), walks you through standing up
> a MySQL → Kafka CDC pipeline on *any* project by hand, and then shows exactly
> how the GIBP ledger uses this pattern to pull data out of the legacy `gibp04`
> billing database in near-real-time without touching that database's code.
>
> **Status in this repo.** Live and load-bearing, but with a foot in two worlds.
> The local dev pipeline runs today via `docker compose` (a MySQL mirror →
> Debezium → Kafka → `apps/kafka-worker`). The production pipeline is defined in
> `infra/k8s/infrastructure/kafka/debezium.yaml` and points at the real remote
> `gibp04` MySQL. Downstream processing runs behind a **shadow-mode feature flag**
> (`GIBP_SYNC_PROCESSING_ENABLED`) — the worker ingests and audits every change
> event but only *acts* on them (posts to Formance) when the flag is on. Several
> correctness gaps are known and flagged (see §8). Treat this chapter as the map;
> treat `gibp-docs/cdc-remote-sync/` as the deeper territory.
>
> **How to read this.** If you are new to CDC, read §§1–4 in order — they build the
> mental model. If you just need to *stand up* a pipeline, jump to §5. If you need
> to understand or debug **this** repo's pipeline specifically, read §7 then keep
> §9 open as a playbook. §§10–11 are the "things that will bite you" and the
> glossary. Cross-references: Kafka fundamentals live in
> [`03-kafka.md`](./03-kafka.md); how any of this actually ships to the cluster
> lives in [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md).

---

## Table of contents

1. [The problem: getting data out of a database you don't own](#1-the-problem)
2. [Mental model: the court stenographer](#2-mental-model)
3. [Core concepts & vocabulary](#3-core-concepts--vocabulary)
4. [How it actually works (deep)](#4-how-it-actually-works-deep)
5. [Setup from scratch (any project)](#5-setup-from-scratch)
6. [Using the events downstream](#6-using-the-events-downstream)
7. [How THIS repo uses Debezium](#7-how-this-repo-uses-debezium)
8. [Production concerns](#8-production-concerns)
9. [Operations & debugging playbooks](#9-operations--debugging-playbooks)
10. [Gotchas & hard-won lessons](#10-gotchas--hard-won-lessons)
11. [Glossary & further reading](#11-glossary--further-reading)

---

## 1. The problem

You are building a new system, but the **facts you need already live inside
someone else's database**. In GIBP's case, a legacy PHP/MySQL application called
`gibp04` is the system of record for bills, payment batches, vendors, customers,
and ACH transactions. Our job is to take each of those changes and turn them into
double-entry ledger postings in Formance and audit rows in BigQuery — **in
near-real-time**, and **without being allowed to modify `gibp04`**.

That last constraint is the whole ballgame. We cannot add code to `gibp04`. We
cannot ask its developers to publish events for us. We cannot change its schema.
We can, at best, get a read-only replication login. So: how do we learn, within
seconds, that row `id=8842` in `gibp_bills` just changed from status `pending` to
`approved`?

Here are the options a junior engineer usually reaches for first, and why each one
falls short.

### Option A — Polling ("just query it every few seconds")

Run `SELECT * FROM gibp_bills WHERE updated_at > :last_seen` on a timer.

| Problem | Why it hurts |
| --- | --- |
| **You miss deletes.** | A deleted row simply isn't in the result set. You can never tell "was deleted" from "was never there." |
| **You miss intermediate states.** | If a row goes `A → B → A` between two polls, you see nothing changed. CDC would show you all three. |
| **It needs a reliable `updated_at`.** | Many legacy tables don't have one, or don't update it on every write, or update it in a non-monotonic way (clock skew, backfills). |
| **Latency vs. load is a lose-lose knob.** | Poll every 10s → up to 10s stale. Poll every 200ms → you hammer the source DB with full-table scans forever. |
| **It's a full scan or an index you may not have.** | On a big table, "give me everything since T" is expensive and competes with the source app's real traffic. |

### Option B — Dual writes ("have the app write to us too")

Modify `gibp04` so that every time it writes a bill, it also emits an event / calls
our API / writes to a queue.

| Problem | Why it hurts |
| --- | --- |
| **We're not allowed to.** | We don't own `gibp04`. Full stop for our case. |
| **It's not atomic.** | The DB write and the event publish are two separate operations. The app can commit the row and then crash before publishing — now the two systems disagree forever. This is the classic **dual-write problem**. |
| **Every write path must remember.** | Miss one code path (a batch script, an admin tool, a stored procedure) and those changes silently never reach you. |

### Option C — Nightly batch ETL ("dump and diff overnight")

Export the tables each night, diff against yesterday, load the deltas.

| Problem | Why it hurts |
| --- | --- |
| **It's a day stale.** | Useless for a payments system that has to act on a bill within minutes. |
| **Diffing is fragile.** | Same problems as polling (deletes, intermediate states) plus a heavy nightly load spike. |

### The reliable answer: read the database's own transaction log

Every serious database already writes down **every change it makes**, in commit
order, in a durable internal log — because it needs that log for crash recovery
and for **replication** (feeding read-replica copies of itself). In MySQL this log
is the **binary log (binlog)**. In PostgreSQL it's the **Write-Ahead Log (WAL)**.

That log is the **single, authoritative, ordered record of truth** about what
happened to the data. It captures inserts, updates, *and* deletes; it never misses
an intermediate state; it is produced atomically as part of the commit; and it
already exists whether we read it or not.

**Change Data Capture (CDC)** is the technique of *tailing that log* and turning
each committed change into a message on a stream. We don't poll. We don't ask the
app to cooperate. We pretend to be a **replica** — the one thing the source
database is already built to serve — and we read the changes it was always going
to broadcast anyway.

**Debezium** is the open-source tool that does this. It speaks MySQL's and
Postgres's native replication protocols, reads their logs, and publishes each
change as a structured event onto **Kafka**.

```
   The wrong ways                          The right way (CDC)
   ─────────────                           ───────────────────
   app ──(poll)──▶ us      brittle         binlog ──▶ Debezium ──▶ Kafka ──▶ us
   app ──(dual write)──▶ us  atomicity      (the DB's own change log; we tail it)
   nightly dump ──▶ us     stale
```

---

## 2. Mental model

> **Debezium is a court stenographer who sits quietly in the corner of the
> database, reading its private diary out loud onto a public loudspeaker.**

Hold this picture:

- The **private diary** is the MySQL **binlog** (or the Postgres **WAL**). The
  database writes into it, in exact order, *every single thing* it does to its
  data. It was never written for us — it's the DB's own internal bookkeeping.
- The **stenographer** is **Debezium**. It has permission to *read* the diary (a
  replication login) but never to write in it. It reads each new entry the instant
  it appears.
- The **loudspeaker** is a **Kafka topic**. The stenographer reads each diary entry
  aloud — "Row 8842 in `gibp_bills`: status changed from `pending` to `approved` at
  14:03:22" — and that spoken sentence becomes a durable Kafka message that anyone
  subscribed can hear, now or later.
- **We** (`apps/kafka-worker`) are a person in the audience taking notes and acting
  on them — creating ledger postings, writing audit rows.

Two properties of a good stenographer map directly onto why CDC is trustworthy:

1. **They read in order.** The diary is chronological; the stenographer reads it
   top to bottom. So changes arrive **in the same order they were committed** (per
   row, guaranteed — see §4).
2. **They can pick up where they left off.** If the stenographer steps out for
   coffee, they remember the page and line number (the **binlog position** / the
   **offset**) and resume exactly there. They never re-read the whole diary from
   the start (unless they lose their bookmark — see §8 and §10).

Now map every fairy-tale word back to the real term so the analogy doesn't leak:

| Story | Reality |
| --- | --- |
| Private diary | MySQL **binlog** (Postgres **WAL**) |
| Diary entry | A committed row change (insert/update/delete) |
| Stenographer | **Debezium MySQL connector** running inside **Kafka Connect** |
| Reading permission (never writing) | `REPLICATION SLAVE`, `REPLICATION CLIENT`, `SELECT` grants |
| Loudspeaker | A **Kafka topic** (one per source table) |
| The sentence read aloud | A **Debezium change event** (`before`/`after`/`op`/`source`) |
| The page/line bookmark | The connector's stored **offset** (binlog file + position, or GTID) |
| The audience taking notes | `apps/kafka-worker` consumers |

Everything else in this chapter is just detail hanging off this one picture.

---

## 3. Core concepts & vocabulary

Read this table once now, and again after §4 — it will click harder the second
time. Everything here is standard Debezium/Kafka-Connect vocabulary; the
repo-specific values come in §7.

| Term | Plain-English definition |
| --- | --- |
| **CDC (Change Data Capture)** | The practice of capturing every insert/update/delete in a database as a stream of change events, by reading the database's own change log. |
| **binlog (MySQL binary log)** | MySQL's ordered, durable log of every committed change. Must be in **ROW** format for CDC (see below). This is what Debezium tails. |
| **WAL / logical replication (Postgres)** | Postgres's equivalent. The physical WAL is decoded into logical row changes by a **logical decoding** plugin (`pgoutput`). Same idea, different plumbing. |
| **Kafka Connect** | A framework (a cluster of worker JVMs) that runs **connectors** to move data in and out of Kafka. It handles config, offsets, retries, scaling, and a REST API — so a connector author only writes the "read from MySQL" logic. |
| **Connector** | A configured integration. A **source connector** (Debezium) reads from an external system into Kafka; a **sink connector** writes from Kafka into an external system. We only use source connectors. |
| **Task** | The unit of work a connector is split into for parallelism. Debezium's MySQL connector uses exactly **one task** (`tasks.max: 1`) because a single binlog must be read in order by one reader. |
| **Source vs. sink** | Source = into Kafka. Sink = out of Kafka. CDC is always a *source*. |
| **Snapshot** | The initial, one-time read of the *current* full contents of the tables, so consumers start from a complete picture — not just "changes from now on." Emitted as `op: "r"` (read) events. |
| **Streaming** | After the snapshot, the continuous tailing of the binlog for new changes (`op: c/u/d`). |
| **Offset** | The connector's bookmark: how far it has read. For MySQL this is a binlog file name + position (and/or a GTID set). Stored *in Kafka*, not in the source DB. |
| **Tombstone** | A Kafka message with a **key but a `null` value**, emitted right after a delete. It tells log-compacted topics "this key is gone, you may garbage-collect it." |
| **Topic-per-table** | Debezium publishes each source table to its **own** Kafka topic, named `<prefix>.<db>.<table>`. |
| **SMT (Single Message Transform)** | A lightweight, per-message transformation applied inside Connect — e.g. rename a topic, drop a field, flatten the envelope. Configured, not coded. |
| **Converter** | How Connect serializes the key and value onto the wire. Common choices: **JSON** (self-describing, verbose) or **Avro** (compact binary, needs a schema registry). Key and value can use different converters. |
| **Schema Registry** | A service that stores Avro (or Protobuf/JSON) schemas and hands out integer IDs. Avro messages carry only the ID; consumers fetch the full schema from the registry. Keeps messages tiny and enforces schema compatibility. |
| **DLQ (Dead-Letter Queue)** | A side topic where un-processable messages are parked so the main flow isn't blocked. Note: Connect has its own DLQ mechanism, *and* an application can have its own — this repo uses the latter (§7). |
| **at-least-once delivery** | The delivery guarantee CDC pipelines give: every change is delivered **at least once**, possibly **more than once** (on retries/restarts). It is *not* exactly-once. Therefore **consumers must be idempotent** — the single most important downstream discipline. |
| **server-id** | A number that uniquely identifies each participant in MySQL replication. Every Debezium connector must present a **unique** `server-id`, or MySQL will kick readers off (see §10). |
| **GTID (Global Transaction ID)** | An optional MySQL feature giving every transaction a globally unique ID, making offset tracking robust across failovers. Enabled in this repo's mirror. |

---

## 4. How it actually works (deep)

### 4.1 The two phases: snapshot, then stream

When a Debezium connector starts for the very first time, it does **not** begin at
"changes from now on." That would leave consumers with no idea what the *current*
state of the data is. Instead it runs in two phases:

```
   PHASE 1: SNAPSHOT                         PHASE 2: STREAMING
   (one-time, "op": "r")                     (forever, "op": c/u/d)
   ┌──────────────────────────┐             ┌───────────────────────────┐
   │ 1. Note current binlog    │             │ Tail the binlog from the  │
   │    position P0.           │──────────▶  │ remembered position;      │
   │ 2. SELECT * every table   │             │ emit an event per change; │
   │    → emit "read" events.  │             │ advance the offset;       │
   │ 3. Finish at position P0. │             │ commit offset to Kafka.   │
   └──────────────────────────┘             └───────────────────────────┘
```

1. **Snapshot.** Debezium records the *current* binlog position, then reads every
   configured table with `SELECT *` and emits one event per existing row with
   `op: "r"` ("read"). This gives every consumer a complete starting picture. On a
   large table this can be slow and, depending on `snapshot.mode` and isolation
   settings, can briefly take locks — a real operational concern (see §10).
2. **Streaming.** As soon as the snapshot finishes, Debezium switches to tailing
   the binlog *from the position it recorded at the start of the snapshot*, so no
   change committed during the snapshot is lost. From here on it emits `c`
   (create), `u` (update), and `d` (delete) events forever, advancing its offset as
   it goes.

The `snapshot.mode` config decides what phase 1 does. The important values:

| `snapshot.mode` | Behaviour | Used where in this repo |
| --- | --- | --- |
| `initial` | Snapshot the data **once**, then stream. The default and the safe choice. | `gibp-source-local.json`, `gibp-source-remote.json` |
| `schema_only` | Capture only the *table structure*, **skip the data**, then stream changes from now on. Use when you don't need historical rows (or will backfill another way). | `infra/k8s/.../debezium.yaml` (production K8s) |
| `never` | Never snapshot; stream only. Assumes offsets already exist. | — |

> **Heads-up (real drift in this repo):** the local/dev connectors snapshot the
> data (`initial`) while the production K8s connector uses `schema_only`. That is a
> deliberate difference — production gets its historical rows another way — but it
> means "it worked locally" does **not** prove the production snapshot behaviour.
> Flagged again in §7 and §10.

### 4.2 The change-event envelope

Every Debezium event has the **same shape** regardless of table — an *envelope*
wrapping the row data. This is the single most important thing to internalise,
because all your downstream code keys off it. In this repo the envelope is typed in
`packages/kafka/src/cdc.types.ts`:

```ts
export type DebeziumOperation = 'c' | 'u' | 'd' | 'r';

export interface DebeziumSource {
  version: string;   connector: string;  name: string;
  ts_ms: number;     db: string;         table: string;
  server_id: number; file: string;       pos: number;
  row: number;       gtid: string | null;
}

export interface CdcEventPayload<T = Record<string, unknown>> {
  before: T | null;      // row state BEFORE the change — null on insert/snapshot
  after: T | null;       // row state AFTER  the change — null on delete
  source: DebeziumSource; // where/when this came from (binlog coordinates, etc.)
  op: DebeziumOperation;  // c=create, u=update, d=delete, r=read(snapshot)
  ts_ms: number;          // when the connector emitted the event
  transaction: unknown | null; // present only if txn metadata is enabled
}
```

A concrete **UPDATE** event for a bill looks like this on the wire (JSON form;
`gibp04-cdc.gibp04.gibp_bills`):

```json
{
  "before": {
    "id": 8842,
    "bill_status_code": 1,
    "bill_amount": 152000,
    "vendor_id": 41,
    "t_updated": "2026-07-16T09:00:00Z"
  },
  "after": {
    "id": 8842,
    "bill_status_code": 3,
    "bill_amount": 152000,
    "vendor_id": 41,
    "t_updated": "2026-07-16T14:03:22Z"
  },
  "source": {
    "version": "3.0.0.Final",
    "connector": "mysql",
    "name": "gibp04-cdc",
    "ts_ms": 1752674602000,
    "db": "gibp04",
    "table": "gibp_bills",
    "server_id": 184055,
    "file": "mysql-bin.000037",
    "pos": 41984210,
    "row": 0,
    "gtid": "3E11FA47-71CA-11E1-9E33-C80AA9429562:1-100"
  },
  "op": "u",
  "ts_ms": 1752674602123,
  "transaction": null
}
```

How to read it:

- **`op`** tells you what kind of change. `c` = insert (`before` is `null`), `u` =
  update (both present), `d` = delete (`after` is `null`), `r` = a row read during
  the initial snapshot.
- **`before` / `after`** are the full row images. Because this repo sets
  `binlog_row_image = FULL` (see `my.cnf`), the `before` image contains *all*
  columns, not just the ones that changed — so you can diff old vs. new reliably.
- **`source`** is the provenance block. `file` + `pos` are the exact binlog
  coordinates; `gtid` is the global transaction ID; `server_id` identifies the
  source MySQL; `ts_ms` here is *when the change happened at the source*, whereas
  the top-level `ts_ms` is *when Debezium emitted the event* — the gap between them
  is your **end-to-end CDC lag**.

The Kafka **message key** is the row's primary key (e.g. `{"id": 8842}`). This
matters enormously: Kafka guarantees ordering **within a partition**, and messages
with the same key go to the same partition — so **all changes to bill 8842 arrive
in commit order**, even though changes to *different* bills may interleave. (See
[`03-kafka.md`](./03-kafka.md) for partitioning mechanics.)

### 4.3 Where Kafka Connect keeps its own state

Kafka Connect is itself stateless on disk — it stores everything it needs to
survive a restart in **three internal Kafka topics**:

| Internal topic | Holds | This repo (docker-compose) | This repo (K8s) |
| --- | --- | --- | --- |
| **config** | The JSON config of every registered connector | `connect_configs` | `debezium-connect-standalone-configs` |
| **offset** | Each source connector's bookmark (binlog file/pos/GTID) | `connect_offsets` | `debezium-connect-standalone-offsets` |
| **status** | Running/failed state of connectors and tasks | `connect_status` | `debezium-connect-standalone-status` |

Two consequences you must remember:

- Because **offsets live in Kafka** (not in MySQL, not on the Connect worker's
  disk), a Connect worker can crash and restart and pick up exactly where it left
  off. But it also means: **if you delete the offset records, the connector forgets
  where it was and re-snapshots** (§10).
- Deleting the *connector* via the REST API removes its **config**, but its
  **offsets survive** (they're keyed by connector name in the offset topic). So
  re-creating a connector with the *same name* resumes streaming — it does **not**
  re-snapshot. This is exactly why this repo's `register-connector.sh` can safely
  delete-then-recreate on every boot (§7).

Additionally, the Debezium MySQL connector keeps a **schema history topic** — a log
of every `DDL` (table structure change) it has seen, so that when it replays old
binlog entries it knows what the table looked like *at that time*. In this repo
that's `schema-changes.gibp04` (remote) / `schema-changes.local-gibp04` (local).
Losing this topic corrupts the connector's ability to interpret the binlog.

### 4.4 server-id, GTID, and binlog retention

- **server-id.** MySQL replication requires every reader to present a **unique**
  numeric `server-id`. Debezium *is* a replica as far as MySQL is concerned. If two
  connectors (or a connector and a real replica) share a `server-id`, MySQL will
  repeatedly disconnect one of them and replication thrashes. This repo assigns
  distinct IDs on purpose: the local mirror MySQL is `223346`, the remote gibp04
  connector uses `184055`, real MySQL is `223344`. **Keep them unique.**
- **GTID.** With `gtid_mode = ON` (set in the mirror's `my.cnf`), every transaction
  gets a globally unique ID. Offsets tracked by GTID survive source failovers and
  binlog file renames far more gracefully than raw file+position offsets. Strongly
  recommended for production CDC.
- **Binlog retention.** MySQL eventually *purges* old binlog files to reclaim disk.
  If the connector is **down longer than the retention window**, the binlog
  position it remembered may have been deleted — the diary pages it bookmarked are
  gone. When that happens the connector cannot resume and must **re-snapshot** (or
  fail). This repo keeps 7 days of binlog (`binlog_expire_logs_seconds = 604800`),
  which is the connector's maximum safe downtime. See §8.

### 4.5 Converters & Avro + Schema Registry

A **converter** decides how a change event is serialized onto the Kafka wire.

- **JSON converter** — human-readable, self-describing, verbose. Great for local
  debugging; every message repeats the field names.
- **Avro converter** — compact binary. Each message carries only a small integer
  **schema ID** plus the packed values; the full schema lives in the **Schema
  Registry**. Consumers see the ID, fetch the schema once, and cache it. This keeps
  messages small and enforces **schema compatibility** (the registry rejects an
  incompatible schema change, protecting consumers).

This repo's data connectors use the **Avro converter** for both key and value,
pointed at a Confluent **Schema Registry** (`SCHEMA_REGISTRY_URL`, default
`http://schema-registry:8081`). One consequence you'll actually trip over: an Avro
message value begins with a **magic byte `0x00`** followed by the schema ID. If you
try to read it as UTF-8 you get null characters — which is why
`cdc-ingestion.normalizer.ts` special-cases a leading `0x00` and base64-encodes the
key rather than corrupting it. **The producer's converter and the consumer's
deserializer must match**, or you get garbage (§10).

### 4.6 SMTs (Single Message Transforms)

An **SMT** is a small transformation Connect applies to each message *without any
custom code* — you just add config. Common uses: **route/rename** topics
(`RegexRouter`), **unwrap** the envelope so downstream sees a flat row
(`ExtractNewRecordState`), **mask** or **drop** fields. This repo's connectors do
**not** currently use SMTs (they publish the full envelope on the default
`<prefix>.<db>.<table>` topics and unwrap it in application code — see
`cdc-ingestion.normalizer.ts`). Mentioned here so you recognise the tool when you
see it in other projects.

### 4.7 Failure modes (know these cold)

| Failure | What you observe | What's happening | First move |
| --- | --- | --- | --- |
| **Connector FAILED** | `/status` shows `state: FAILED` with a stack trace | Bad config, unreachable DB, permissions, or an unhandled event with `errors.tolerance: none` | Read the trace via REST (§9); fix root cause; `POST .../restart` |
| **Binlog purged** | Connector fails on restart complaining the requested binlog position no longer exists | It was down longer than retention (§4.4) | Re-snapshot (reset offsets) or restore from GTID; then fix the retention/downtime gap |
| **Schema change on source** | Events keep flowing but new/renamed columns appear; or a `DDL` breaks deserialization | Debezium recorded the DDL in its schema-history topic and adapts; Avro registry may reject an incompatible change | Verify registry compatibility mode; ensure consumers tolerate additive changes |
| **Duplicate server-id** | Connector repeatedly disconnects/reconnects; unstable | Another reader shares the `server-id` | Assign a unique `database.server.id` (§10) |
| **Snapshot never finishes** | Only `op: r` events, table lock contention, high source load | A huge table under `snapshot.mode: initial` | Consider `schema_only` + a controlled backfill, or an incremental snapshot |

---

## 5. Setup from scratch

This section stands alone: follow it to build a MySQL → Kafka CDC pipeline on **any
project**, then §7 maps each step onto what this repo already ships. You need a
running Kafka broker (see [`03-kafka.md`](./03-kafka.md)); everything else is below.

### 5.1 Prepare MySQL as a source

Debezium can only tail the binlog if the binlog exists, is in the right format, and
is retained long enough. And it needs a login with just enough privilege — no more.

**(a) Enable the binlog in ROW format.** Add to `my.cnf` (`[mysqld]` section). These
are exactly this repo's mirror settings from
`infra/debezium/mysql-gibp04/my.cnf`:

```ini
[mysqld]
# Unique across ALL replication participants (see §10)
server-id       = 223346

# Turn the binary log on
log_bin         = mysql-bin

# ROW format is MANDATORY for CDC. It records the actual row values that
# changed, not the SQL statement. (STATEMENT/MIXED are useless to Debezium.)
binlog_format   = ROW

# Record the FULL row image on every change, so `before` contains all columns.
binlog_row_image = FULL

# Keep 7 days of binlog → the connector can be down up to 7 days and still resume.
binlog_expire_logs_seconds = 604800
max_binlog_size            = 536870912   # 512 MB per file

# GTIDs: optional but strongly recommended for robust offsets across failover.
gtid_mode                = ON
enforce_gtid_consistency = ON
```

> **Why ROW and not STATEMENT?** STATEMENT format logs the SQL text
> (`UPDATE bills SET status=3 WHERE ...`). Debezium can't reliably know which rows
> that affected or what the values became — especially for non-deterministic SQL
> (`NOW()`, `RAND()`). ROW format logs the actual before/after values of each
> affected row. CDC needs the values, so **ROW is non-negotiable.**

**(b) Create a least-privilege replication user.** Exactly this repo's
`infra/debezium/mysql-gibp04/init/01-debezium-user.sql`:

```sql
CREATE USER IF NOT EXISTS 'debezium'@'%'
  IDENTIFIED WITH mysql_native_password BY 'debezium_password';

GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT
  ON *.* TO 'debezium'@'%';

FLUSH PRIVILEGES;
```

What each grant is for — least privilege means you grant these and *nothing more*:

| Grant | Why Debezium needs it |
| --- | --- |
| `SELECT` | To read table rows during the **snapshot** phase |
| `RELOAD` | To flush tables / take a consistent snapshot |
| `SHOW DATABASES` | To discover the databases it's configured to watch |
| `REPLICATION SLAVE` | To connect to the binlog stream and read change events |
| `REPLICATION CLIENT` | To query binlog status/position (`SHOW MASTER STATUS`) |

Note it has **no `INSERT`/`UPDATE`/`DELETE`** — the stenographer can read the diary
but never writes in it. In production, restrict the host (`'debezium'@'10.%'`) and
consider scoping `SELECT` to the specific database.

### 5.2 Run Kafka Connect with the Debezium MySQL connector (Docker)

Kafka Connect needs the Debezium MySQL connector *plugin* installed. Two ways this
repo does it, both valid:

**Option 1 — the official Debezium image (what production K8s uses):**

```yaml
image: quay.io/debezium/connect:2.6.0.Final   # plugin already bundled
```

**Option 2 — a Confluent Connect base + install the plugin (what local dev uses).**
Exactly `infra/debezium/Dockerfile`:

```dockerfile
FROM confluentinc/cp-kafka-connect:7.8.0
ARG DEBEZIUM_MYSQL_VERSION=3.0.0.Final
USER root
RUN curl -fsSL \
  "https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/${DEBEZIUM_MYSQL_VERSION}/debezium-connector-mysql-${DEBEZIUM_MYSQL_VERSION}-plugin.tar.gz" \
  | tar xz -C /usr/share/confluent-hub-components/
USER appuser
```

Either way, the Connect worker needs a handful of environment variables so it knows
which Kafka to talk to and where to keep its internal topics. This repo's
docker-compose `debezium-connect` service sets (abridged to essentials):

```yaml
environment:
  CONNECT_BOOTSTRAP_SERVERS: kafka:9092
  CONNECT_GROUP_ID: debezium-connect-group
  CONNECT_CONFIG_STORAGE_TOPIC: connect_configs
  CONNECT_OFFSET_STORAGE_TOPIC: connect_offsets
  CONNECT_STATUS_STORAGE_TOPIC: connect_status
  CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter    # internal
  CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter  # internal
  CONNECT_PLUGIN_PATH: /usr/share/java,/usr/share/confluent-hub-components
  CONNECT_SECURITY_PROTOCOL: SASL_PLAINTEXT
  CONNECT_SASL_MECHANISM: PLAIN
  # ... SASL JAAS config with user "connect" ...
ports:
  - "${DEBEZIUM_PORT:-8083}:8083"   # the Connect REST API
```

> The `CONNECT_*_CONVERTER` values above are for Connect's own *internal* topics.
> The **data** topics' converters are set **per connector** in the connector JSON
> (§5.3), where this repo chooses Avro. Don't confuse the two.

Bring it up and wait for the REST API on `:8083` to answer:

```bash
docker compose up -d kafka schema-registry debezium-connect
curl -s http://localhost:8083/ | jq .        # version info once ready
curl -s http://localhost:8083/connector-plugins | jq '.[].class'  # confirm the MySQL plugin is present
```

### 5.3 Register a connector via the REST API

Kafka Connect is driven entirely by a **REST API**. You register a connector by
`POST`-ing a JSON config to `/connectors`. Here is a **full, annotated** MySQL
source config — this is `infra/debezium/connectors/gibp-source-remote.json` with
comments added (real JSON can't have comments; these are for you):

```jsonc
{
  "name": "gibp_source_remote",              // connector name → also the offset key
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",                         // ALWAYS 1 for a single binlog reader

    // ── Where the source MySQL is ────────────────────────────────────────────
    "database.hostname": "${MYSQL_HOST}",
    "database.port":     "${MYSQL_PORT}",
    "database.user":     "${MYSQL_USER}",     // the least-privilege user from 5.1(b)
    "database.password": "${MYSQL_PASSWORD}",
    "database.server.id":"${MYSQL_SERVER_ID}",// UNIQUE across all readers (§10)
    "database.ssl.mode": "${MYSQL_SSL_MODE}", // e.g. "preferred" / "disabled"

    // ── What to publish, and where ───────────────────────────────────────────
    "topic.prefix": "gibp04-cdc",             // topics become gibp04-cdc.<db>.<table>
    "database.include.list": "${MYSQL_DATABASE}",
    "table.include.list":
      "${MYSQL_DATABASE}.gibp_bills,${MYSQL_DATABASE}.gibp_payment_batches, ...",  // explicit allow-list of tables

    // ── Schema-history topic (DDL log — see §4.3) ────────────────────────────
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.gibp04",
    // ... plus SASL creds for that topic's producer/consumer ...

    // ── How to serialize onto the wire (§4.5) ────────────────────────────────
    "key.converter":   "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url":   "${SCHEMA_REGISTRY_URL}",
    "key.converter.schemas.enable": "true",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "${SCHEMA_REGISTRY_URL}",
    "value.converter.schemas.enable": "true",
    "schema.name.adjustment.mode": "avro",    // sanitize names to be Avro-legal

    // ── Behaviour ────────────────────────────────────────────────────────────
    "snapshot.mode": "initial",               // snapshot once, then stream (§4.1)
    "include.schema.changes": "true",
    "heartbeat.interval.ms": "10000",         // emit a heartbeat every 10s (see §8)
    "decimal.handling.mode": "double",        // how NUMERIC/DECIMAL columns map
    "time.precision.mode": "connect",
    "database.connectionTimeZone": "UTC",
    "tombstones.on.delete": "true",           // emit a tombstone after each delete

    // ── Error handling (fail-fast — NOT a Connect DLQ) ───────────────────────
    "errors.max.retries": "10",
    "errors.retry.delay.max.ms": "60000",
    "errors.tolerance": "none",               // stop the task on an un-handled error
    "errors.log.enable": "true",
    "errors.log.include.messages": "true",

    // ── Throughput tuning ────────────────────────────────────────────────────
    "max.batch.size": "2048",
    "max.queue.size": "8192",
    "poll.interval.ms": "500"
  }
}
```

Register it (the `${...}` placeholders must already be substituted — see §7 for how
this repo does that):

```bash
curl -sS -X POST http://localhost:8083/connectors \
  -H 'Content-Type: application/json' \
  --data @gibp-source-remote.json | jq .
```

A `201 Created` means success. An idempotent alternative (create *or* update) is a
`PUT` to `/connectors/<name>/config` with just the inner `config` object — this is
what the production K8s hook uses.

### 5.4 Verify it works

Prove data is actually flowing — don't assume:

```bash
# 1. Connector exists and is RUNNING (task state too)
curl -s http://localhost:8083/connectors | jq .
curl -s http://localhost:8083/connectors/gibp_source_remote/status | jq .

# 2. The per-table topics were auto-created
docker compose exec kafka \
  kafka-topics --bootstrap-server kafka:9092 --list | grep gibp04-cdc

# 3. Actually watch change events land (JSON example; Avro needs a schema-aware consumer)
docker compose exec kafka \
  kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic gibp04-cdc.gibp04.gibp_bills --from-beginning --max-messages 5

# 4. Make a change at the source and watch it appear
#    UPDATE gibp_bills SET bill_status_code = 3 WHERE id = 8842;
```

For Avro topics, use the schema-registry-aware console consumer, or just open the
**Kafka UI** (this repo ships `provectuslabs/kafka-ui` on port `8090`, wired to both
the broker and the Connect REST API), which decodes Avro for you and shows connector
status in a browser.

### 5.5 The Postgres-source variant (brief)

The concepts are identical; the plumbing differs. For a Postgres source you:

- use `connector.class: io.debezium.connector.postgresql.PostgresConnector`;
- set `wal_level = logical` in `postgresql.conf` (the equivalent of MySQL's ROW
  binlog) and use a **logical decoding** output plugin — the modern default is
  **`pgoutput`** (built in; no extra install);
- grant the user `REPLICATION` and create a **replication slot** (Postgres's
  equivalent of the binlog bookmark — and a sharp edge: a slot for a *dead*
  connector will pin WAL and fill the source disk, so drop stale slots);
- there is no `server-id`; instead each connector needs a unique **`slot.name`** and
  **`publication.name`**.

Everything downstream — the envelope, `op` codes, topic-per-table, converters,
offsets in Kafka — is the same. This repo's sources are all MySQL, so we won't go
deeper here.

---

## 6. Using the events downstream

CDC only pays off when something *consumes* the events. This section is the
consumer's playbook.

### 6.1 Topic naming

Debezium names each topic `<topic.prefix>.<database>.<table>`. This repo centralises
those names in `packages/kafka/src/kafka.topics.ts` so no consumer hard-codes a
string:

```ts
const CDC_REMOTE_DATABASE = process.env['CDC_MYSQL_DATABASE'] ?? 'gibp04';
export const KAFKA_TOPICS = {
  CDC_REMOTE_BILLS:           `gibp04-cdc.${CDC_REMOTE_DATABASE}.gibp_bills`,
  CDC_REMOTE_PAYMENT_BATCHES: `gibp04-cdc.${CDC_REMOTE_DATABASE}.gibp_payment_batches`,
  // ... one per captured table ...
  CDC_REMOTE_DLQ: 'gibp04-cdc.dlq',
} as const;
```

### 6.2 Consuming change events in a worker

A consumer subscribes to a topic, receives the envelope, and switches on `op`. This
repo does exactly that with NestJS microservice `@MessagePattern` handlers (see
`apps/kafka-worker/src/modules/cdc-ingestion/cdc-ingestion.controller.ts`):

```ts
@MessagePattern(KAFKA_TOPICS.CDC_REMOTE_ENTITIES)
async handleEntities(@Payload() raw: unknown, @Ctx() ctx: KafkaContext) {
  await this.ingestionService.ingest(raw, ctx);  // normalise, dedup, (maybe) act
}
```

### 6.3 Handling the op types

| `op` | Meaning | `before` | `after` | Typical handling |
| --- | --- | --- | --- | --- |
| `r` | Snapshot read | `null` | full row | Upsert into your model as "current state" |
| `c` | Insert | `null` | new row | Create the corresponding downstream record |
| `u` | Update | old row | new row | Diff and apply; the `before` image lets you detect *what* changed |
| `d` | Delete | last row | `null` | Soft-delete / tombstone downstream; then expect a Kafka tombstone message |

A subtle but important edge, handled in `cdc-ingestion.normalizer.ts`: a **tombstone
message has a `null` value**, and a raw event may arrive either as the bare payload
or wrapped in a `{ "payload": {...} }` envelope depending on converter settings. The
normalizer defends against both, and if `op` is missing it *infers* it from the
presence of `before`/`after` (`before && !after → 'd'`, etc.). Robust consumers
never assume a perfectly-shaped message.

### 6.4 Idempotency & ordering per key — the non-negotiable discipline

Because CDC is **at-least-once** (§3), the same change *will* be delivered more than
once eventually — on a connector restart, a consumer rebalance, or a redeploy. If
your handler is not **idempotent**, duplicates corrupt your data: a payment posted
twice, an audit row duplicated.

This repo's ingestion layer dedups on the Kafka **coordinates** of the message —
topic + partition + offset are unique per delivery — via
`findOrCreateByKafkaPosition` (`cdc-ingestion.service.ts`):

```ts
const { created } = await this.auditLog.findOrCreateByKafkaPosition(attrs);
if (!created) {
  this.logger.debug(`Duplicate event skipped ...`);  // seen before → no-op
}
```

Two rules follow from at-least-once + per-key ordering:

1. **Idempotency:** design every downstream write so that applying the same event
   twice equals applying it once (upserts keyed by source `id`, dedup tables, or a
   natural idempotency key on the target — e.g. a deterministic Formance transaction
   reference). Kafka-offset dedup handles *exact* replays; a *business* idempotency
   key is what protects you when the same logical change arrives via a different
   offset (e.g. after a re-snapshot).
2. **Ordering:** rely on Kafka's per-key ordering (all changes to `id=8842` land on
   one partition in commit order). Do **not** parallelise processing of the same key
   across partitions/consumers, or you can apply an older state after a newer one.

### 6.5 The DLQ for un-processable events

When a message can't be processed after retries, you don't want it to block the
partition forever. The pattern is a **dead-letter queue**: publish the poison
message to a side topic and move on. This repo has an application-level DLQ
(`gibp-sync/retry/dlq.service.ts`) that publishes to `gibp04-cdc.dlq`:

```ts
await this.kafkaProducer.produce({
  topic: KAFKA_TOPICS.CDC_REMOTE_DLQ,
  messages: [{ value: JSON.stringify({ originalTopic, error: error.message, payload: raw, failedAt }) }],
});
```

> **Honest caveat (see §8):** this DLQ is currently **write-only** — events land in
> it but nothing consumes or re-drives it, and if the *DLQ publish itself* fails the
> error is only logged. So a poison event can be **silently dropped**. Treat the DLQ
> as an alerting signal, not a safety net, until a re-drive path exists.

### 6.6 Translating source rows into this system's model

The raw CDC row is *the legacy schema* — snake_case MySQL columns with legacy
quirks. Downstream **handlers** (`gibp-sync/handlers/*.ts`:
`bill.handler.ts`, `payment-batch.handler.ts`, `customer.handler.ts`,
`vendor.handler.ts`, `ach-batch.handler.ts`) translate each row into this system's
model and post the resulting double-entry transaction to Formance. This is where
domain meaning is assigned — and where the legacy-data quirks in §8 (inverted
currency codes, negative fee rows) must be handled explicitly, because Debezium
faithfully reproduces whatever the source stored, warts and all.

---

## 7. How THIS repo uses Debezium

Now the concrete picture. The GIBP ledger runs **two** CDC source setups that share
one topic namespace so the downstream worker never has to know which one is live.

### 7.1 The pipeline end to end

```
  ┌──────────────────────┐   binlog    ┌─────────────────────┐   Avro over    ┌──────────┐
  │  SOURCE MySQL        │─(ROW fmt)──▶ │  Debezium (Kafka    │──Kafka topics─▶│  Kafka   │
  │                      │             │  Connect, :8083)    │  gibp04-cdc.*  │          │
  │  prod: remote gibp04 │             │  MySqlConnector     │                └────┬─────┘
  │  dev:  gibp04-local- │             │  topic.prefix =     │                     │
  │        mysql (8.0)   │             │  "gibp04-cdc"       │                     ▼
  └──────────────────────┘             └─────────────────────┘        ┌────────────────────────┐
                                                                       │  apps/kafka-worker      │
   register-connector.sh (curl init) ──POST──▶ Connect REST :8083      │  • dedup + audit (always)│
                                                                       │  • if flag on: post to   │
   Schema Registry :8081 ◀── Avro schemas ──▶ producers/consumers      │    Formance + BigQuery   │
                                                                       └────────────────────────┘
```

### 7.2 The two source setups

| | **Local dev** | **Production** |
| --- | --- | --- |
| Source DB | `mysql:8.0` container `gibp04-local-mysql`, host port **3308**→3306 | Remote real `gibp04` MySQL |
| Schema | `infra/debezium/mysql-gibp04/init/02-create-gibp04-tables.sql` (the 11–12 CDC tables only) | The real production schema |
| binlog config | `infra/debezium/mysql-gibp04/my.cnf` (ROW, FULL, 7-day retention, GTID on) | Managed on the source side |
| Connector JSON | `gibp-source-local.json` | `gibp-source-remote.json` |
| Connect runtime | docker-compose `debezium-connect` (custom `Dockerfile`, `cp-kafka-connect:7.8.0` + Debezium MySQL **3.0.0.Final** plugin) | K8s `debezium.yaml`, image `quay.io/debezium/connect:2.6.0.Final` |
| Registered by | `register-connector.sh` (a `curlimages/curl` **init container**) | a Python **`postStart` lifecycle hook** in the pod |
| Snapshot mode | `initial` (snapshot the data) | `schema_only` (skip data) |
| Converter | **Avro** + Schema Registry | JSON (image defaults in the hook payload) |
| server-id | `223346` (mirror) via `GIBP04_LOCAL_MYSQL_SERVER_ID` | `184055` via `CDC_MYSQL_SERVER_ID` (hook hard-codes `"1"`) |
| `tombstones.on.delete` | `true` | `false` |
| `decimal.handling.mode` | `double` | `string` |

**Both use the same `topic.prefix` `gibp04-cdc`.** That is the key design choice:
whether the events come from the local mirror or the real remote DB, they land on
identically-named topics (`gibp04-cdc.gibp04.<table>`), so **`apps/kafka-worker`
needs zero code changes to switch between dev and prod sources.** The local mirror
exists precisely so a developer can exercise the whole pipeline without a
connection to production.

> **Be honest about the drift.** The two setups differ in more than just the
> hostname: different base image and Debezium version (3.0.0.Final vs 2.6.0.Final),
> different snapshot mode (`initial` vs `schema_only`), different converter (Avro vs
> JSON), different decimal handling (`double` vs `string`), different registration
> mechanism, and different `server-id`. This is real and worth knowing: a change
> verified against the local Avro pipeline is **not** automatically proven against
> the production JSON pipeline. The deeper `gibp-docs/cdc-remote-sync/` design docs
> are where this convergence is being worked through.

There is also a **third, unrelated** connector, `local-mysql-source.json` — a toy
`users`/`orders` example on `topic.prefix: mysql-server` (topics
`mysql-server.<db>.users|orders`, JSON converter, and note it demonstrates
`column.exclude.list` to drop `password`/`password_hash`). It's a learning/reference
connector, not part of the gibp04 flow.

### 7.3 The captured tables

The connector's `table.include.list` is an explicit **allow-list** — Debezium
captures only these, nothing else in `gibp04`. Each maps to a topic constant in
`packages/kafka/src/kafka.topics.ts`:

| Source table | Topic (`gibp04-cdc.gibp04.<table>`) | Domain |
| --- | --- | --- |
| `gibp_transaction_schedules` | `...gibp_transaction_schedules` | ACH batches |
| `gibp_transactions` | `...gibp_transactions` | ACH transactions |
| `gibp_payment_batches` | `...gibp_payment_batches` | Payment batches |
| `gibp_payment_batch_currency_totals` | `...gibp_payment_batch_currency_totals` | Batch currency totals |
| `gibp_payment_batch_currency_entries` | `...gibp_payment_batch_currency_entries` | Batch currency entries |
| `gibp_bills` | `...gibp_bills` | Bills |
| `gibp_accounts` | `...gibp_accounts` | Customers |
| `gibp_vendors` | `...gibp_vendors` | Vendors |
| `gibp_vendor_services` | `...gibp_vendor_services` | Vendor services |
| `gibp_account_service_signups` | `...gibp_account_service_signups` | Signups |
| `gibp_entities_general` | `...gibp_entities_general` | Entity reference |
| `gibp_account_transactions` | `...gibp_account_transactions` | Account transactions |

Plus the DLQ topic `gibp04-cdc.dlq` (§6.5).

### 7.4 How registration actually happens locally

The connector JSON files use `${MYSQL_HOST}`-style placeholders. They are **not**
Kafka Connect config providers — they are substituted by shell before the POST.
`infra/debezium/register-connector.sh` (run as the `connector-init-*` service):

1. Waits for the Connect REST API at `CONNECT_URL` (`http://debezium-connect:8083`),
   retrying up to `MAX_RETRIES` (30) every `RETRY_INTERVAL` (5s).
2. `sed`-substitutes every `${VAR}` in the chosen `CONNECTOR_CONFIG_PATH` from env
   (`MYSQL_HOST/PORT/USER/PASSWORD/DATABASE/SERVER_ID/SSL_MODE`,
   `SCHEMA_REGISTRY_URL`, `KAFKA_CONNECT_PASSWORD`).
3. If a connector of that name already exists, **DELETE**s it and polls until the
   REST API returns `404` (avoids a create-before-delete race).
4. **POST**s the substituted JSON to `/connectors`, expecting `201`.
5. Prints `/status`.

> **Why delete-then-recreate is safe:** deleting a connector removes its *config*
> but **not its offsets** (offsets are keyed by connector name in the offset topic,
> §4.3). Recreating with the same name resumes from the stored offset — it does
> **not** re-snapshot. If you *rename* the connector, or wipe the offset topic, you
> *will* trigger a fresh snapshot.

The docker-compose wiring: `connector-init-gibp-source-local` (default) points at
the local mirror; `connector-init-gibp-source-remote` (under the `remote-cdc`
compose profile) points at the real DB. Both `depends_on` a healthy
`debezium-connect` (healthcheck: `curl -sf http://localhost:8083/`).

### 7.5 Environment variables

| Variable | Purpose | Default / example |
| --- | --- | --- |
| `DEBEZIUM_PORT` | Host port for the Connect REST API | `8083` |
| `SCHEMA_REGISTRY_URL` | Confluent Schema Registry for Avro | `http://schema-registry:8081` |
| `KAFKA_CONNECT_PASSWORD` | SASL/PLAIN password for user `connect` | (secret) |
| **Remote source** | | |
| `CDC_MYSQL_HOST` | Remote gibp04 host | (secret) |
| `CDC_MYSQL_PORT` | Remote gibp04 port | `6033` |
| `CDC_MYSQL_USER` / `CDC_MYSQL_PASSWORD` | Replication login | (secret) |
| `CDC_MYSQL_DATABASE` | Source DB name (also drives topic names) | `gibp04` |
| `CDC_MYSQL_SERVER_ID` | Unique replication id | `184055` |
| `CDC_MYSQL_SSL_MODE` | TLS mode to the source | `preferred` |
| **Local mirror** | | |
| `GIBP04_LOCAL_MYSQL_*` | Host/port/user/password/database for the dev mirror | host `gibp04-local-mysql`, port `3306` (exposed `3308`) |
| `GIBP04_LOCAL_MYSQL_SERVER_ID` | Unique replication id for the mirror | `223346` |
| `GIBP04_LOCAL_MYSQL_SSL_MODE` | TLS mode | `disabled` |
| **Downstream** | | |
| `GIBP_SYNC_PROCESSING_ENABLED` | Shadow-mode switch: `false` → audit-only (ingest+dedup+record but **don't** post to Formance) | `false` by default |

In production K8s, the source DB creds come from an **`ExternalSecret` →
`debezium-secrets`** (`debezium-db-host/port/user/password`), never from a checked-in
file. Locally they come from `.env` (see `.env.example`).

### 7.6 Downstream: what the worker does with the events

`apps/kafka-worker` consumes the `gibp04-cdc.*` topics and, for each event:

1. **Normalises** the envelope (`cdc-ingestion.normalizer.ts`) into the
   `CdcEventPayload` shape (§4.2), inferring `op`/`source` defensively.
2. **Dedups + audits** it via `findOrCreateByKafkaPosition` — every event, even in
   shadow mode, is recorded as a `RemoteCdcEvent` audit row.
3. **If `GIBP_SYNC_PROCESSING_ENABLED` is true**, hands the normalised payload to the
   domain handlers, which post the corresponding double-entry transaction to
   **Formance** and stream audit rows to **BigQuery**. If the flag is false, it stops
   after step 2 ("audit-only mode" — logged loudly on startup).

For the deeper design — normalization rules, intermediate services, the full
migration and production rollout strategy — read, in order:
`gibp-docs/cdc-remote-sync/01-general-plan.md`, `02-technical-plan.md`,
`03-sync-general-plan.md`, `04-sync-technical-plan.md`,
`05-normalization-formance-plan.md`, `06`/`07-intermediate-services-*`,
`08-gibp-sync-runtime-guide.md`, and `11-migration-and-production-rollout-strategy.md`.

---

## 8. Production concerns

CDC is easy to demo and hard to run. Here is what actually matters in production.

- **Connector HA & restarts.** A single Connect worker is a single point of failure;
  in K8s it runs as a `Deployment` (1 replica here) with liveness/readiness probes on
  `/connectors`. On restart, the connector resumes from its Kafka-stored offset — no
  data loss *provided the binlog is still retained* (next point). Run more than one
  Connect worker in the same `group.id` for real HA so a failed worker's task is
  rebalanced onto a survivor.

- **Binlog retention vs. downtime — the cardinal rule.** The connector can only
  resume if the binlog position it remembers still exists on the source. This repo
  keeps **7 days** (`binlog_expire_logs_seconds = 604800`). **If the connector is
  down longer than the retention window, its bookmark points at a purged file → it
  cannot resume → you must re-snapshot** (and, with `schema_only` in prod, you'd have
  a gap to backfill). Retention is therefore your **maximum tolerable outage**. Alarm
  on connector downtime well before it approaches the window.

- **Unique server-id per connector.** Restated because it *will* bite you: every
  reader (each Debezium connector, plus any real MySQL replica) needs its own
  `database.server.id`. Duplicates cause both readers to be repeatedly disconnected.

- **Schema changes on the source.** When someone `ALTER`s a `gibp04` table, Debezium
  records the DDL in its schema-history topic and keeps going for *additive* changes.
  But a **dropped/renamed column** an Avro consumer depends on, or an
  incompatible-type change, can break deserialization or be rejected by the Schema
  Registry. Coordinate source schema changes; prefer additive changes; watch the
  registry's compatibility mode.

- **Exactly-once vs. at-least-once.** Debezium is **at-least-once**. There is no
  exactly-once across this whole chain. The **only** defense is downstream idempotency
  (§6.4). This is flagged as a live gap here — see below.

- **Secrets for source DB creds.** The replication login is a credential to a
  production database. Locally it's in `.env`; in K8s it's an `ExternalSecret`
  (`debezium-secrets`). Never commit it; rotate it; scope it (least privilege, §5.1).

- **Monitoring connector status & lag.** Poll `/connectors/<name>/status` for
  `state: RUNNING` on both connector and tasks. Watch **lag** — the gap between the
  source `source.ts_ms` and the emit `ts_ms` in events, and the consumer group lag on
  the `gibp04-cdc.*` topics. The connectors emit a **heartbeat every 10s**
  (`heartbeat.interval.ms`), which both advances offsets on low-traffic tables (so the
  bookmark doesn't fall behind a purging binlog) and gives you a liveness signal.

### 8.1 Known/flagged issues in *this* pipeline (be honest)

These are documented gaps, not hypotheticals. Do not assume the pipeline is
correct end-to-end today:

| Issue | Impact | Mitigation / status |
| --- | --- | --- |
| **Formance postings not fully idempotent** | A replayed event (at-least-once) can double-post to the ledger | Kafka-offset dedup catches exact replays; a *business* idempotency key on Formance transactions is the real fix and is not fully in place |
| **ACH total accumulator bug** | Totals can be mis-accumulated across events | Flagged; validate against source totals before trusting |
| **Inverted currency-code quirk** | Source encodes `1 = JPY, 2 = USD` — the *opposite* of an intuitive reading; a naive mapping flips currencies | Handlers must map currency codes explicitly and correctly |
| **Fee/adjustment rows as negative amounts** | Some rows carry negative `amount` meaning a fee/adjustment, not a normal bill | Handlers must special-case negatives rather than treat them as bills |
| **Write-only DLQ, silent drops** | Poison events go to `gibp04-cdc.dlq` but nothing re-drives them; a failed DLQ publish is only logged | Treat DLQ as an alert; a re-drive consumer is needed |
| **No automated reconciliation checker** | Nothing continuously proves "source rows == ledger postings" | Manual verification only today; an automated reconciler is a known TODO |
| **~5× reprocessing from multiple consumer groups** | Multiple consumer groups over the same topics multiply processing; without idempotency this amplifies double-posting risk | Reinforces that **idempotency/dedup is the key downstream discipline** |

The through-line: **idempotency and reconciliation are the load-bearing disciplines
here, and both are incomplete.** Until they're solid, production processing stays
behind `GIBP_SYNC_PROCESSING_ENABLED` (shadow/audit-only mode).

---

## 9. Operations & debugging playbooks

### 9.1 Kafka Connect REST cheat sheet

All against `http://localhost:8083` (in-cluster: `http://debezium-connect:8083`).

```bash
# List all connectors
curl -s :8083/connectors | jq .

# Full status: connector state + each task's state (look for FAILED)
curl -s :8083/connectors/gibp_source_remote/status | jq .

# View the live config
curl -s :8083/connectors/gibp_source_remote/config | jq .

# Restart the connector, and/or a specific failed task
curl -s -X POST :8083/connectors/gibp_source_remote/restart
curl -s -X POST :8083/connectors/gibp_source_remote/tasks/0/restart

# Pause / resume (stops streaming without deleting offsets)
curl -s -X PUT :8083/connectors/gibp_source_remote/pause
curl -s -X PUT :8083/connectors/gibp_source_remote/resume

# Update config in place (idempotent create-or-update)
curl -s -X PUT :8083/connectors/gibp_source_remote/config \
  -H 'Content-Type: application/json' --data @config-only.json

# Delete the connector (KEEPS offsets → recreating same name resumes, no re-snapshot)
curl -s -X DELETE :8083/connectors/gibp_source_remote

# What plugins are installed?
curl -s :8083/connector-plugins | jq '.[].class'
```

### 9.2 "Connector is FAILED"

1. `GET /connectors/<name>/status` and read the `trace` on the connector **and each
   task** (a connector can be RUNNING while a task is FAILED).
2. Common causes: DB unreachable (host/port/SSL), wrong creds, missing grants
   (§5.1), duplicate `server-id`, purged binlog, or — because `errors.tolerance:
   none` — a single un-handled event.
3. Fix the root cause, then `POST /connectors/<name>/restart` (add
   `?includeTasks=true`).

### 9.3 "No events are flowing"

Work down the pipe:

1. **Is the connector RUNNING?** `/status`. If FAILED, go to §9.2.
2. **Is the binlog on?** On the source: `SHOW VARIABLES LIKE 'log_bin';` (must be
   `ON`) and `SHOW VARIABLES LIKE 'binlog_format';` (must be `ROW`).
3. **Does the user have grants?** `SHOW GRANTS FOR 'debezium'@'%';` — needs
   `REPLICATION SLAVE`, `REPLICATION CLIENT`, `SELECT`.
4. **Is the snapshot still running?** A large `initial` snapshot emits only `op: r`
   and no `c/u/d` until it finishes. Check task logs / `snapshot` metrics.
5. **Are the topics even there?** `kafka-topics --list | grep <prefix>`. If the
   topics exist but are empty, no changes have been made to those tables since the
   connector started (make a test write).
6. **Is the consumer subscribed to the right topic name?** Compare against
   `packages/kafka/src/kafka.topics.ts` — a `CDC_MYSQL_DATABASE` mismatch changes the
   topic name.

### 9.4 "Duplicate / replayed events"

Expected under at-least-once — the question is whether your consumer is idempotent
(§6.4). Verify the dedup layer (`findOrCreateByKafkaPosition`) is logging
`Duplicate event skipped`. If duplicates are causing double **effects** (double
ledger posts), that's the idempotency gap in §8.1, not a bug in Debezium.

### 9.5 Reset a connector (force a fresh snapshot)

Sometimes you *want* to re-read everything (offsets corrupted, binlog purged, schema
reset). Because offsets survive a plain DELETE, you must clear the **offset**:

- **Preferred (modern Connect):** `DELETE /connectors/<name>/offsets` while the
  connector is stopped, then restart — Connect re-runs the snapshot.
- **Nuclear (docker-compose local):** stop everything, delete the `connect_offsets`
  topic (and, if schema history is corrupt, `schema-changes.*`), then re-register.
  Only do this in dev; in prod it re-snapshots a production table.

### 9.6 Inspect the raw change-event JSON

```bash
# JSON topics (e.g. the local users/orders demo, or Connect internals):
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic gibp04-cdc.gibp04.gibp_bills --from-beginning --max-messages 3 \
  --property print.key=true

# Avro topics: use the schema-registry-aware consumer, or the Kafka UI (:8090)
kafka-avro-console-consumer --bootstrap-server kafka:9092 \
  --property schema.registry.url=http://localhost:8081 \
  --topic gibp04-cdc.gibp04.gibp_bills --from-beginning --max-messages 3
```

### 9.7 Check the DLQ

```bash
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic gibp04-cdc.dlq --from-beginning
```

Each DLQ message carries `{ originalTopic, error, payload, failedAt }`. Remember it's
**write-only** today (§6.5/§8.1) — anything here needs manual attention.

---

## 10. Gotchas & hard-won lessons

- **binlog MUST be ROW format, and MUST be retained.** `STATEMENT`/`MIXED` are
  useless to Debezium. Retention (`binlog_expire_logs_seconds`) is your maximum
  tolerable connector downtime — exceed it and you re-snapshot.
- **Duplicate `server-id` silently wrecks replication.** Two readers with the same id
  keep disconnecting each other. Every connector and every real replica needs a
  unique one. (Here: mirror `223346`, remote `184055`, mysql `223344`.)
- **Snapshots can be huge and locking.** An `initial` snapshot of a big table is slow
  and may take locks. Prefer `schema_only` + a controlled backfill, or an incremental
  snapshot, for large production tables. (This is exactly why prod here uses
  `schema_only`.)
- **`op: 'd'` then a tombstone.** A delete is *two* messages: the delete event
  (`after: null`) and then a `null`-value **tombstone** on the same key. Consumers
  must not choke on the `null` value. This repo's normalizer handles it; naive
  `JSON.parse(value)` on a tombstone will throw.
- **Schema drift breaks the wire silently.** If the source schema changes and your
  Avro consumer isn't compatible, you get failed deserialization or registry
  rejections — not obviously "a Debezium problem." Coordinate DDL; prefer additive.
- **Converters must match on both ends.** Avro producer ⇄ Avro-aware consumer; JSON
  ⇄ JSON. Mismatch yields "garbage" — remember the leading `0x00` magic byte on Avro
  messages that the normalizer base64-encodes to avoid null-char corruption in
  Postgres text columns.
- **Offsets live in Kafka; deleting *them* re-snapshots, deleting the *connector*
  does not.** The single most misunderstood operational fact. `DELETE /connectors/x`
  keeps offsets (recreate resumes); `DELETE /connectors/x/offsets` (or wiping the
  offset topic) forces a fresh snapshot.
- **`errors.tolerance: none` means fail-fast, not "DLQ everything."** These
  connectors stop the task on an un-handled error (after `errors.max.retries`). The
  `gibp04-cdc.dlq` topic is an *application-level* DLQ produced by the worker, not
  the Connect framework's DLQ. Don't conflate them.
- **Two runtimes, one namespace — verify against the right one.** Local (Avro,
  `initial`, Debezium 3.0.0.Final) and prod (JSON, `schema_only`, 2.6.0.Final)
  diverge. "Works locally" is necessary, not sufficient (§7.2).
- **The source data is quirky and Debezium is faithful.** Inverted currency codes
  (`1=JPY, 2=USD`), negative amounts meaning fees — Debezium reproduces the legacy
  data exactly, quirks and all. The *handlers* must know the quirks (§8.1).

---

## 11. Glossary & further reading

### Quick glossary

| Term | One-liner |
| --- | --- |
| **CDC** | Capturing every DB change as a stream by reading the DB's own change log. |
| **binlog** | MySQL's ordered log of committed changes; must be ROW format for CDC. |
| **WAL / `pgoutput`** | Postgres's equivalent log + its logical-decoding output plugin. |
| **Kafka Connect** | The framework that runs connectors, stores their config/offsets/status in Kafka, and exposes a REST API on `:8083`. |
| **Connector / task** | A configured integration / its unit of parallel work (Debezium MySQL = 1 task). |
| **Snapshot (`op: r`)** | One-time initial read of current table contents. |
| **Streaming (`op: c/u/d`)** | Continuous tailing of the binlog after the snapshot. |
| **Envelope** | The `before`/`after`/`op`/`source`/`ts_ms` shape wrapping every event. |
| **Offset** | The connector's bookmark (binlog file+pos / GTID), stored in Kafka. |
| **Tombstone** | A `null`-value message after a delete, for log compaction. |
| **Converter** | On-wire serialization (JSON vs. Avro). |
| **Schema Registry** | Stores Avro schemas; messages carry only a schema ID. |
| **server-id** | Unique MySQL-replication identity each reader must present. |
| **GTID** | Globally-unique transaction id for robust, failover-safe offsets. |
| **DLQ** | Side topic for un-processable messages (`gibp04-cdc.dlq` here). |
| **at-least-once** | Delivery guarantee → consumers must be idempotent. |
| **Shadow mode** | `GIBP_SYNC_PROCESSING_ENABLED=false` → ingest+audit but don't act. |

### Further reading

**Official docs**
- Debezium documentation (home): https://debezium.io/documentation/
- Debezium MySQL connector: https://debezium.io/documentation/reference/stable/connectors/mysql.html
- Debezium PostgreSQL connector: https://debezium.io/documentation/reference/stable/connectors/postgresql.html
- Debezium event envelope / data types: https://debezium.io/documentation/reference/stable/connectors/mysql.html#mysql-events
- Kafka Connect (Apache): https://kafka.apache.org/documentation/#connect
- Kafka Connect REST API: https://kafka.apache.org/documentation/#connect_rest
- Confluent Schema Registry & Avro: https://docs.confluent.io/platform/current/schema-registry/index.html
- MySQL binary log & ROW format: https://dev.mysql.com/doc/refman/8.0/en/binary-log.html

**In this repo**
- [`03-kafka.md`](./03-kafka.md) — Kafka fundamentals: brokers, topics, partitions, consumer groups, ordering.
- [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md) — how Connect and the worker actually ship to the cluster.
- `packages/kafka/src/cdc.types.ts` — the typed change-event envelope.
- `packages/kafka/src/kafka.topics.ts` — the canonical topic names.
- `apps/kafka-worker/src/modules/cdc-ingestion/` — normalization, dedup, audit.
- `apps/kafka-worker/src/modules/gibp-sync/` — the domain handlers and DLQ.
- `infra/debezium/` — Dockerfile, connector JSON, `register-connector.sh`, the MySQL mirror (`my.cnf` + `init/*.sql`).
- `infra/k8s/infrastructure/kafka/debezium.yaml` — the production Connect deployment.
- `gibp-docs/cdc-remote-sync/` — the deep design docs (general/technical plans, normalization, intermediate services, runtime guide, and the migration & production rollout strategy).
```
