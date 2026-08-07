# 02 — PostgreSQL

**What this chapter is.** A from-zero, career-long guide to PostgreSQL (Postgres): what a relational database is, *why* one exists, how Postgres actually works under the hood, how to run and operate one, and exactly how this repo — the GIBP ledger, a production financial system — uses Postgres through Sequelize and Google Cloud SQL. Read it once to learn the subject; keep it as the reference you return to when a migration fails at 2 a.m.

**Status in this repo.** Postgres is the API's system-of-record relational database.

- Locally it runs as the Docker image **`postgres:16-alpine`** (docker-compose service `db`), with a second **`postgres_test`** instance for the test suite.
- Node talks to it through **`pg` 8.18.0** (the driver) and the **`sequelize` 6.37.7** ORM, wired into NestJS via **`@nestjs/sequelize` 11.0.1** and **`sequelize-typescript` 2.1.6**. Schema is owned by **migrations** run through **`sequelize-cli` 6.6.5**.
- In production the database is a managed **Google Cloud SQL** instance, `gibp-ledger:us-west1:gibp-ledger-postgres`, reached *not directly* but through a **Cloud SQL Auth Proxy** pod inside the cluster.
- Postgres also backs **Formance** (the double-entry ledger engine); Formance stores its data in the same Postgres server locally via `POSTGRES_URI`.

**How to read this.** Sections 1–4 teach the *subject* (relational databases and Postgres internals) — read top to bottom the first time. Sections 5–6 are hands-on: run Postgres yourself and connect from code. Section 7 is *this repo specifically*. Sections 8–11 are the reference half: production concerns, debugging playbooks, gotchas, glossary. Every shell/SQL block is meant to be runnable. When production plumbing (secrets, the proxy, GCP) is involved, this chapter points you to [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md) and [`08-gcp.md`](08-gcp.md) rather than duplicating them.

---

## 1. The problem: why a relational database at all?

Imagine you are asked to store the money GIBP moves — bills, ACH batches, wires, ledger postings — and you reach for the simplest thing: files.

You start with a spreadsheet, `bills.xlsx`. It works for a week. Then reality arrives:

| What you need | Why a file/spreadsheet breaks |
|---|---|
| **Two people editing at once** | Person A opens the file, Person B opens the file, both save. One of them silently overwrites the other. A payment vanishes. |
| **"Only apply this if the whole thing succeeds"** | You need to debit one account *and* credit another. The process crashes between the two writes. Now money exists in neither place (or both). There is no "undo half of it." |
| **"Find every unpaid bill for vendor 4021 in June"** | You scan the entire file by hand, or write a script that reads all rows every time. At 10 million rows it takes minutes. |
| **"This bill must reference a real customer"** | Nothing stops someone typing `CustomerId = 99999` for a customer that doesn't exist. The data rots. |
| **"The power just went out mid-save"** | The file is now half-written and corrupt. Yesterday's numbers are gone. |
| **"Prove the books balanced at close of business"** | You have no consistent snapshot — the file was changing while you read it. |

A **relational database** exists to solve exactly these problems. It is a program whose entire job is to store data so that it is:

- **Durable** — once it tells you "saved," the data survives a crash, a power loss, a process kill.
- **Consistent** — the data always obeys the rules you declared (a bill *must* point at a real customer; an amount *cannot* be null).
- **Queryable** — you ask *what* you want ("unpaid June bills for vendor 4021") and the engine figures out *how* to get it fast, using indexes.
- **Concurrent** — thousands of readers and writers can work at the same time without corrupting each other's data or seeing half-finished changes.
- **Transactional** — you can group several changes into one all-or-nothing unit. Either every step happens, or none does.

**Why ACID matters *especially* for a financial ledger.** ACID is the four-letter promise a serious database makes about transactions: **A**tomicity, **C**onsistency, **I**solation, **D**urability (defined fully in §3). For most apps, a lost "like" or a duplicated comment is annoying. For GIBP, a half-applied transfer is *money created or destroyed out of thin air*. The whole reason a double-entry ledger exists is that every debit has an equal credit and the books always balance. That invariant is only trustworthy if the database can guarantee: *both* sides of a transfer commit together, or neither does; a crash never leaves you half-posted; and a concurrent reader never sees a moment where the books don't balance. That guarantee is ACID. It is not a nice-to-have here — it is the product.

> **Rule of thumb.** If wrong data means wrong money, you want a transactional relational database, and you want to use its guarantees deliberately — not a pile of files, not a cache pretending to be a database, and never floating-point numbers for amounts (see §10).

---

## 2. Mental model: the strict librarian

One analogy, mapped carefully back to the real terms. Use it to build intuition, then drop it.

Think of Postgres as a **strict, meticulous librarian** running a library:

- The **tables** are like different kinds of shelves — one for *Users*, one for *Bills*, one for *Accounts*. Each is a grid: **rows** (one book / one record) and **columns** (the fixed fields every book on that shelf must have: title, author, ISBN).
- The **schema** is the library's rulebook: which shelves exist, what fields each book must have, what's allowed to be blank. The librarian *enforces* the rulebook — you cannot shelve a "book" with no title if the rulebook says a title is required.
- A **primary key** is the book's unique catalog number. No two books share one. Ask for catalog #42 and you get exactly one book.
- A **foreign key** is a cross-reference: "this Bill belongs to the Customer with catalog #17." The librarian refuses to file a cross-reference to a customer that doesn't exist — that's referential integrity.
- An **index** is the card catalog: instead of walking every shelf to find "all books by author = 4021," the librarian keeps a sorted card file that jumps straight there.
- A **transaction** is the librarian's promise: "I will make these five changes as one act. If I'm interrupted, I undo all of them — you'll never see three-of-five done."
- **MVCC** (§4) is how the librarian lets many people read a book while someone else edits a fresh copy of it, so nobody trips over a half-edited page.

The key idea the analogy carries: **the database is not a dumb bucket of bytes; it actively enforces rules and hands out consistent views.** That is exactly why we push correctness (foreign keys, `NOT NULL`, unique constraints, `NUMERIC` types) *into the database* rather than trusting application code to remember every rule everywhere.

Now the terms are real; retire the librarian.

---

## 3. Core concepts & vocabulary

Read this table once, then use it as a lookup. Everything after this section assumes these words.

| Term | Plain meaning | Why it matters |
|---|---|---|
| **Table** | A named grid of data (e.g. `Bills`). | The unit of storage; roughly one "kind of thing." |
| **Row** (record/tuple) | One entry in a table — one bill. | The thing you insert, read, update, delete. |
| **Column** (field/attribute) | One typed slot every row has (`billAmount`, `currencyType`). | Defines shape and type; the type is enforced. |
| **Schema** | (a) The overall structure of your tables; (b) also a *namespace* inside a database (default `public`). | The rulebook. In this repo, migrations define it. |
| **Data type** | The kind of value a column holds: `INTEGER`, `TEXT`, `NUMERIC`, `TIMESTAMPTZ`, `BOOLEAN`, `JSONB`. | Wrong type = silent bugs. `NUMERIC` vs `float` is a money-safety issue (§10). |
| **Primary key (PK)** | A column (or set) uniquely identifying each row; implies unique + not-null. | Every row is addressable; joins hang off it. |
| **Foreign key (FK)** | A column that must match a PK in another table. | Referential integrity: no orphan rows. |
| **Constraint** | A declared rule: `NOT NULL`, `UNIQUE`, `CHECK (...)`, `FOREIGN KEY`. | The DB rejects bad data at the source. |
| **Index** | A sorted lookup structure (usually a B-tree) on one or more columns. | Turns "scan everything" into "jump straight there." Speeds reads, costs writes. |
| **Transaction** | A `BEGIN … COMMIT` unit of one or more statements, applied all-or-nothing. | The atom of correctness. |
| **ACID** | Atomicity, Consistency, Isolation, Durability — the four transaction guarantees. | The reason a ledger can trust the DB (see below). |
| **Isolation level** | How much one in-flight transaction can see another's uncommitted/in-progress work. | Controls concurrency anomalies; money code often wants stricter than the default. |
| **MVCC** | Multi-Version Concurrency Control — the DB keeps multiple row versions so readers don't block writers. | How Postgres stays fast and consistent under concurrency. |
| **WAL** | Write-Ahead Log — changes are written to an append-only log *before* the data files. | How Postgres guarantees Durability and can recover after a crash. |
| **VACUUM / autovacuum** | Background cleanup of dead (obsolete) row versions left by MVCC. | Prevents table "bloat" and keeps stats fresh. |
| **Role** | A database login/permission identity (user or group). | Least-privilege access control. |
| **Connection** | One open session (a TCP socket) between a client and Postgres. | Each costs memory; they are finite (see pooling). |
| **Connection pool** | A reused set of open connections shared by an app. | Avoids the cost of opening/closing constantly and caps total connections. |
| **Migration** | A versioned, ordered script that changes the schema (create table, add column, add index). | How schema evolves safely and reproducibly. Migrations own the schema here. |
| **ORM** | Object-Relational Mapper (Sequelize) — maps rows to objects/classes. | Lets you write TypeScript instead of raw SQL for most access. |
| **Query planner** | The part of Postgres that decides *how* to execute your SQL. | `EXPLAIN` shows its plan; the reason indexes matter. |

**ACID, spelled out (with a money example).** Two accounts: `A` has 100, `B` has 0. We transfer 100 from A to B.

```sql
BEGIN;                                   -- start the transaction
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;                                  -- both, or nothing
```

- **Atomicity** — if the process dies after the first `UPDATE`, the DB rolls the whole thing back on restart. A never loses 100 into the void.
- **Consistency** — declared rules (a `CHECK (balance >= 0)`, a foreign key) hold before and after. A transaction that would break a rule is rejected, not half-applied.
- **Isolation** — another transaction reading balances *while* this one runs never sees the moment where A is −100 and B is still 0 (a state where the books don't balance).
- **Durability** — once `COMMIT` returns, the change survives a crash, because it was written to the **WAL** first (§4).

That is the entire reason we use Postgres for the ledger instead of anything looser.

---

## 4. How it actually works (deep, but accessible)

You do not need to be able to reimplement Postgres. You *do* need a working mental model of these six mechanisms, because every production surprise traces back to one of them.

### 4.1 MVCC — how concurrent writes don't corrupt each other

Naïve databases lock a row while someone edits it: everyone else waits. That serializes traffic and kills throughput.

Postgres instead uses **MVCC (Multi-Version Concurrency Control)**. When you `UPDATE` a row, Postgres does *not* overwrite it in place. It writes a **new version** of the row and marks the old version as valid only up to your transaction. Every transaction effectively sees a **snapshot** of the database as of the moment it started (or the moment each statement started — see isolation).

Consequences worth internalizing:

- **Readers never block writers, and writers never block readers.** A long report query sees a consistent snapshot even as thousands of writes happen; it just doesn't see the new versions.
- **An `UPDATE` is really an insert-plus-mark-old-dead.** So updates and deletes leave behind **dead tuples** (obsolete row versions) that must be cleaned up later — that's what VACUUM is for (§4.5).
- **Two writers editing the *same* row** *do* conflict: the second one waits for the first to commit or roll back. Different rows: no conflict.

```
Time ──►
Txn 1 (report):  [ snapshot taken ]───reads v1 of row 42────────────► sees v1
Txn 2 (update):        UPDATE row 42 → writes v2, marks v1 dead ─ COMMIT
Txn 1 still sees v1 (its snapshot). New transactions see v2.
```

### 4.2 The WAL — durability and crash recovery

How can `COMMIT` be both *fast* and *crash-proof*? If Postgres had to flush every changed data page to disk in random locations on every commit, it would be slow.

The trick is the **Write-Ahead Log (WAL)**: before a change touches the real data files, Postgres appends a compact record of "here is what changed" to a sequential log and flushes *that* to disk. Sequential appends are cheap. Once the WAL record is safely on disk, the commit is durable — even if the machine loses power one millisecond later. On restart, Postgres **replays** the WAL to reconstruct any changes that hadn't yet made it into the data files.

The WAL is also the foundation of two production features you'll meet in §8:

- **Point-in-time recovery (PITR)** — replay the WAL up to a chosen moment to "rewind" the database.
- **Replication** — stream the WAL to a standby server so a read replica or hot standby stays current.

Mental model: the WAL is the database's *black-box flight recorder*. The data files are the plane; the WAL is the tape that lets you reconstruct exactly what happened.

### 4.3 Transactions & isolation levels — with money

Isolation controls what one transaction can see of another's concurrent work. Postgres offers three effective levels:

| Level | Prevents | Typical use |
|---|---|---|
| **Read Committed** (default) | Dirty reads (never sees uncommitted data). Each *statement* sees a fresh snapshot. | Most app queries. |
| **Repeatable Read** | Also: non-repeatable reads. The whole transaction sees one snapshot; rows you read won't change under you. | Multi-step reads that must be self-consistent. |
| **Serializable** | Also: all serialization anomalies. Behaves *as if* transactions ran one-at-a-time. | Correctness-critical invariants across rows. |

Why this matters for money — a concrete hazard. Two clerks each read that a fund has $100 available and each approve a $70 withdrawal at the same time:

```
Txn A: SELECT balance FROM fund;   -- sees 100
Txn B: SELECT balance FROM fund;   -- also sees 100
Txn A: UPDATE fund SET balance = 100 - 70;   -- 30
Txn B: UPDATE fund SET balance = 100 - 70;   -- 30  ← overdraft! total withdrawn = 140
```

Under **Read Committed** both can proceed and you overdraw. Fixes, in order of preference:

1. **Do the math in the database, atomically**, so you never trust a stale read:
   ```sql
   UPDATE fund SET balance = balance - 70 WHERE id = 1 AND balance >= 70;
   -- affected-rows = 0 means "insufficient funds"; the row lock serializes the two.
   ```
2. **Lock the row you're about to spend against**: `SELECT balance FROM fund WHERE id = 1 FOR UPDATE;` — the second transaction blocks until the first commits, then reads the true balance.
3. **Raise the isolation level to Serializable** for the transaction; Postgres will detect the conflict and abort one transaction with a serialization error, and you retry it.

The lesson: the *default* isolation level does not protect you from lost-update/double-spend on its own. Financial writes must use `FOR UPDATE`, atomic conditional updates, or Serializable — deliberately.

### 4.4 Indexes (B-tree): when they help and when they hurt

An **index** is a separate, sorted data structure that lets Postgres find rows without scanning the whole table. The default and by far most common type is the **B-tree** — a balanced tree that keeps keys in sorted order, giving `O(log n)` lookups and efficient range scans (`>`, `<`, `BETWEEN`, `ORDER BY`).

Indexes help when:

- You filter or join on the column a lot (`WHERE CustomerId = ?`, `JOIN ... ON ...`).
- You sort or range-scan on it (`WHERE createdAt BETWEEN ...`, `ORDER BY createdAt`).
- You enforce uniqueness (a `UNIQUE` index).

Indexes **hurt** when:

- **Writes** — every `INSERT`/`UPDATE`/`DELETE` must also update every index on the table. Ten indexes = ten extra writes per row change.
- **Storage** — indexes take disk and memory (they compete for cache).
- **Low selectivity** — indexing a column with 2 values (e.g. a boolean) rarely helps; Postgres may just scan.

Rule of thumb: **index the columns you filter/join/sort on, especially foreign keys** (see the unindexed-FK gotcha in §10), and no more. This repo does exactly this — e.g. `apps/api/src/database/migrations/20260414121000-add-accounts-performance-indexes.js` adds targeted indexes on `Accounts(ParentAccountId)` and `Accounts(OrganizationId, deletedAt)` because those are the columns the account queries filter on.

### 4.5 `EXPLAIN` — reading the planner's mind

Postgres is *declarative*: you say what rows you want; the **query planner** picks how to get them (scan the whole table? use an index? which join order?). `EXPLAIN` shows that plan; `EXPLAIN ANALYZE` actually runs it and shows real timings.

```sql
EXPLAIN ANALYZE
SELECT * FROM "Bills" WHERE "CustomerId" = 4021 AND "deletedAt" IS NULL;
```

What to look for:

- **`Seq Scan`** on a big table you filter heavily = usually a missing index (Postgres is reading every row).
- **`Index Scan` / `Index Only Scan`** = the index is being used. Good.
- **`rows=` estimate vs actual** wildly off = stale statistics; run `ANALYZE <table>;` (autovacuum normally does this).
- A **`Nested Loop`** over a huge outer set can be a sign a join isn't using an index.

You don't optimize by guessing — you `EXPLAIN ANALYZE`, find the `Seq Scan` or the bad estimate, and fix *that*.

### 4.6 VACUUM / autovacuum & bloat

Because MVCC leaves dead tuples behind (§4.1), tables and indexes accumulate garbage. **VACUUM** reclaims that space and updates the planner's statistics; **autovacuum** is the background daemon that does it automatically. Normally you leave autovacuum on and forget it. You care when:

- A table takes heavy `UPDATE`/`DELETE` churn and starts to **bloat** (grows far larger than its live data), slowing scans.
- **Transaction ID wraparound** looms — Postgres *must* vacuum periodically to reclaim transaction IDs; a database that outruns autovacuum will start warning and, if ignored, refuse writes to protect itself.

Symptoms and the fix live in §9. The one-liner: dead rows are normal; unbounded dead rows mean autovacuum can't keep up (often blocked by a long-running or idle-in-transaction session — see §10).

### 4.7 Connections & why pooling matters

Each client connection to Postgres is a **separate OS process** with its own memory. That design is robust but means connections are **expensive and finite**. A server configured for, say, `max_connections = 100` will *reject* the 101st connection with `FATAL: too many connections`.

Now picture an app deployed as many pods, each opening its own connections, each spiking under load. It's easy to exhaust the limit and take the whole system down — even though most of those connections sit idle most of the time.

The fix is a **connection pool**: the app keeps a small, reused set of open connections and hands them out to requests as needed. Two layers exist:

- **In-process pool** — the driver/ORM (here, `pg` under Sequelize) keeps a pool per process.
- **External pooler** — a dedicated process (e.g. PgBouncer, or the pooling built into managed Postgres) that fronts the database so *hundreds* of app connections share a *small* number of real Postgres connections.

The governing idea: **total real connections must stay well under `max_connections`, no matter how many pods or requests you have.** More on tuning this in §8.

---

## 5. Setup from scratch

Two paths: run it yourself (Docker), and the managed path (Cloud SQL). Do the Docker path at least once — it demystifies everything.

### 5.1 Run Postgres locally with Docker

You need Docker installed. The fastest correct setup, matching this repo's image `postgres:16-alpine`:

```bash
docker run --name pg-playground \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=playground \
  -p 5432:5432 \
  -v pg_playground_data:/var/lib/postgresql/data \
  -d postgres:16-alpine
```

What each part does:

- `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` — the official image reads these on first boot to create the superuser and an initial database.
- `-p 5432:5432` — publish the container's Postgres port (`5432`) to your machine.
- `-v pg_playground_data:/var/lib/postgresql/data` — a **named volume** so your data *survives* `docker rm`. Without this, deleting the container deletes your database. (In this repo the equivalent volume is `db_data`.)
- `-d` — run in the background.

**A production-shaped setup uses docker-compose with a healthcheck** (so dependents wait until Postgres is actually accepting connections). This is exactly what the repo does — see §7. The essential shape:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: playground
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d playground"]
      interval: 5s
      timeout: 5s
      retries: 20
volumes:
  db_data:
```

`pg_isready` is a tiny Postgres tool that returns success only when the server can accept connections — the right thing to gate startup on.

### 5.2 Connect with `psql` and verify it works

`psql` is the official interactive terminal. If it's not installed on your host, exec into the container:

```bash
docker exec -it pg-playground psql -U postgres -d playground
```

You should get a `playground=#` prompt. Verify:

```sql
SELECT version();          -- prints the PostgreSQL 16.x banner
SELECT current_database(), current_user;
\conninfo                  -- who/where am I connected as
\l                         -- list databases
\q                         -- quit
```

If `SELECT version()` returns, Postgres is up and you are querying it. That's zero-to-querying.

### 5.3 Create a database, users, and roles (least privilege)

In real systems you do **not** hand the application the superuser. You create a database and a scoped role:

```sql
-- as the superuser (postgres):
CREATE DATABASE app_db;
CREATE ROLE app_user WITH LOGIN PASSWORD 'a-strong-password';
GRANT CONNECT ON DATABASE app_db TO app_user;

\connect app_db
GRANT USAGE, CREATE ON SCHEMA public TO app_user;
-- app_user can now create tables and read/write its own data,
-- but is not a superuser and can't touch other databases/roles.
```

Note the distinction: a **role** is any permission identity; a role `WITH LOGIN` is what people call a "user." Groups are just roles you `GRANT` to other roles. Keep application roles least-privilege (§8).

### 5.4 Basic DDL and DML

**DDL** (Data Definition Language) shapes the schema; **DML** (Data Manipulation Language) moves data.

```sql
-- DDL: define a table
CREATE TABLE bills (
  id            BIGSERIAL PRIMARY KEY,      -- auto-incrementing PK
  customer_id   BIGINT NOT NULL REFERENCES customers(id),  -- FK
  bill_amount   NUMERIC(19,4) NOT NULL,     -- money: exact, never float (see §10)
  currency_type CHAR(3),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bills_customer_id ON bills (customer_id);  -- index the FK

-- DML: insert, read, update, delete
INSERT INTO bills (customer_id, bill_amount, currency_type)
VALUES (17, 1234.5600, 'JPY');

SELECT id, bill_amount FROM bills WHERE customer_id = 17;

UPDATE bills SET bill_amount = 1300.0000 WHERE id = 1;

DELETE FROM bills WHERE id = 1;
```

### 5.5 Backups: `pg_dump` and `pg_restore`

A database with no backup is a liability. Two levels:

- **Logical backup** — `pg_dump` writes the data (and/or schema) as SQL or an archive. Great for a single database, moving data between servers, and taking snapshots before risky changes.
- **Physical backup + WAL archiving** — file-level copies plus the WAL, enabling **point-in-time recovery**. This is what managed Postgres does for you (§8).

```bash
# Dump one database to a compressed custom-format archive
pg_dump -U postgres -Fc -d app_db -f app_db.dump

# Restore it into a fresh database
createdb -U postgres app_db_restored
pg_restore -U postgres -d app_db_restored app_db.dump

# Plain SQL dump (human-readable, restore with psql)
pg_dump -U postgres -d app_db > app_db.sql
psql -U postgres -d app_db_restored < app_db.sql
```

> **Always take a `pg_dump` before running an irreversible migration in a non-managed environment.** In production this repo relies on Cloud SQL's automated backups/PITR instead (§8), but the habit is universal.

### 5.6 The managed path: Google Cloud SQL

Running Postgres yourself means you own patching, failover, backups, disk growth, monitoring, and 2 a.m. pages. **Google Cloud SQL** is Google's managed Postgres: you get a Postgres instance and Google operates the machine — automated backups, point-in-time recovery, high-availability failover, minor-version patching, and metrics — while you keep using ordinary Postgres and the ordinary `pg` driver.

You'd choose managed Postgres when uptime and durability of *money data* matter more than saving a few dollars a month — which is exactly the GIBP case. This repo uses Cloud SQL in production (§7); the local Docker `db` service just stands in for it. How you connect *securely* to Cloud SQL (the Auth Proxy, Workload Identity, Secret Manager) is the subject of §7 and, in depth, [`08-gcp.md`](08-gcp.md) and [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md).

---

## 6. Using it in code

### 6.1 The raw driver: `pg`

At the bottom of everything is the **`pg`** driver (`node-postgres`) — this repo pins **`pg` 8.18.0**. It speaks the Postgres wire protocol and gives you a client and a pool. You rarely use it directly here (the ORM sits on top), but you must understand it's the thing actually holding sockets:

```js
import { Pool } from 'pg';

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'app_db',
  user: 'app_user',
  password: process.env.DB_PASSWORD,
  max: 10,                    // pool size: at most 10 real connections
  idleTimeoutMillis: 30_000,
});

// Parameterized query — NEVER string-concatenate user input (SQL injection).
const { rows } = await pool.query(
  'SELECT id, bill_amount FROM bills WHERE customer_id = $1',
  [17],
);
```

Two non-negotiables even at this layer: **always parameterize** (`$1`, `$2` …) — never build SQL by concatenating user input — and **always use a pool**, never open a fresh connection per request.

### 6.2 The ORM: Sequelize + `sequelize-typescript`

Writing raw SQL for every read and write is repetitive and error-prone. An **ORM** (Object-Relational Mapper) maps table rows to TypeScript objects so you work with `Bill` instances instead of hand-written SQL. This repo uses **`sequelize` 6.37.7** with **`sequelize-typescript` 2.1.6** (decorator-based models) and integrates it into NestJS via **`@nestjs/sequelize` 11.0.1**.

A model is a class annotated with decorators. Real example — `apps/api/src/database/models/role.model.ts`:

```ts
@Table({ tableName: TABLE_NAME.ROLES, modelName: TABLE_NAME.ROLES, paranoid: true })
export class Role extends BaseModel<RoleModel, RoleCreationAttributes> implements RoleModel {
  @Unique
  @Index(ACCESS_CONTROL_INDEXES.ROLES_NAME)
  @Column({ type: DataType.STRING(ACCESS_CONTROL_SCHEMA.ROLE_NAME_MAX_LENGTH), allowNull: false, field: ROLE_COLUMNS.NAME })
  declare name: string;

  @Column({ type: DataType.TEXT, allowNull: true, field: ROLE_COLUMNS.DESCRIPTION })
  declare description: string | null;
}
```

Two conventions worth copying from this repo:

- **`paranoid: true`** means **soft delete**: `destroy()` sets a `deletedAt` timestamp instead of physically removing the row, and queries automatically exclude "deleted" rows. Every model extends `BaseModel` (`apps/api/src/database/models/base/base.model.ts`), which centralizes the auto-increment `id`, `createdAt`, `updatedAt`, and `deletedAt`. In an audited financial system, not truly deleting rows is a feature.
- Column names come from **constants**, not string literals — consistent with the repo's "single source, no magic strings" standard.

Wiring into NestJS is done once in `apps/api/src/database/database.module.ts`, which calls `SequelizeModule.forRootAsync(...)` with options built in `apps/api/src/database/config/sequelize.options.ts` (host/port/db/user/password from config, the model list, logging toggle, and SSL only when enabled).

### 6.3 Migrations, and why *not* `sync()` in production

Sequelize can build tables directly from your models via **`sequelize.sync()`** — it inspects the models and issues `CREATE TABLE`. That is convenient and **dangerous in production**, because:

- It infers changes from model diffs, with no reviewable record of *what* it will do.
- `sync({ alter: true })` can silently drop or rewrite columns to "match" the models — data loss with no audit trail.
- There is no ordered history you can replay on a fresh database or roll back.

The professional pattern is **migrations**: small, ordered, version-controlled scripts, each with an `up` (apply) and `down` (revert). They are the *single source of truth for the schema*, reviewed in PRs, run in the same order everywhere, and reproducible from empty. This repo runs on **`sequelize-cli` 6.6.5** with migrations in `apps/api/src/database/migrations` (69 of them at the time of writing).

The commands (from `package.json` scripts; each loads env with `--env-file=.env` and points at the repo's CLI config):

```bash
# Apply all pending migrations
npm run db:migrate

# Roll back the most recent migration
npm run db:migrate:undo

# Scaffold a new, empty migration file (timestamped)
npm run db:migration:generate -- --name add-something-to-bills
```

Under the hood these invoke `sequelize-cli … db:migrate --config apps/api/src/database/config/sequelize-cli.config.js --migrations-path apps/api/src/database/migrations`. Sequelize records which migrations have run in a bookkeeping table (`SequelizeMeta`), so re-running `db:migrate` only applies the new ones.

A migration in this repo (see `apps/api/src/database/migrations/20260318000006-create-bills-table.js`) looks like:

```js
module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.sequelize.transaction(async (transaction) => {  // wrap DDL in a txn
      await queryInterface.createTable('Bills', {
        id:         { type: Sequelize.INTEGER, primaryKey: true, autoIncrement: true, allowNull: false },
        CustomerId: { type: Sequelize.INTEGER, allowNull: true,
                      references: { model: 'Customers', key: 'id' },      // foreign key
                      onUpdate: 'CASCADE', onDelete: 'SET NULL' },
        billAmount: { type: Sequelize.DECIMAL(19, 4), allowNull: false }, // money: exact decimal
        currencyType: { type: Sequelize.STRING(3), allowNull: true },
        createdAt:  { type: Sequelize.DATE, allowNull: false, defaultValue: Sequelize.literal('CURRENT_TIMESTAMP') },
        updatedAt:  { type: Sequelize.DATE, allowNull: false, defaultValue: Sequelize.literal('CURRENT_TIMESTAMP') },
        deletedAt:  { type: Sequelize.DATE, allowNull: true },            // paranoid soft-delete column
      }, { transaction });
    });
  },
  async down(queryInterface) {
    await queryInterface.sequelize.transaction(async (t) => {
      await queryInterface.dropTable('Bills', { transaction: t });
    });
  },
};
```

Note three habits the repo follows: **wrap DDL in a transaction** (so a failed migration doesn't leave a half-created schema, since Postgres supports transactional DDL), **declare foreign keys** with explicit `onDelete`/`onUpdate` behavior, and **use `DECIMAL` for money** (§10).

---

## 7. How *this* repo uses Postgres

Put it all together. Here is the full picture, local and production.

### 7.1 Local: `db` + `postgres_test`

From `docker-compose.yml`:

- **`db`** — the app's Postgres, image `${DB_IMAGE:-postgres:16-alpine}`. Its `POSTGRES_DB` / `POSTGRES_USER` / `POSTGRES_PASSWORD` come from `DB_NAME` / `DB_USER` / `DB_PASSWORD`; it publishes `${DB_PORT}:5432`, persists to the named volume `db_data`, and has a `pg_isready` healthcheck so dependents wait for readiness.
- **`postgres_test`** — a *separate* Postgres for the test suite (`postgres:16-alpine`, its own `pg_test_data` volume, host port default **5435**, default DB `gibp_db_test`). Keeping tests on their own instance means the suite can create/drop/truncate freely without ever touching your dev data.
- **`formance-ledger`** — Formance also runs on Postgres: it connects to the same `db` server via `POSTGRES_URI: postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}`. So both the API's relational data *and* the double-entry ledger sit on Postgres.

The database env vars (`.env.example`) the app and CLI read:

| Var | Default (example) | Meaning |
|---|---|---|
| `DB_IMAGE` | `postgres:16-alpine` | Which Postgres image compose runs. |
| `DB_DIALECT` | `postgres` | Sequelize dialect (validated; only `postgres` is allowed). |
| `DB_HOST` | `localhost` | Host to connect to. |
| `DB_PORT` | `5432` | Port. |
| `DB_NAME` | `gibp-db` | Database name. |
| `DB_USER` | `postgres` | Login role. |
| `DB_PASSWORD` | `postgres` | Password (real secret outside local dev). |
| `DB_LOGGING` | `false` | Log every SQL statement (noisy; dev-only). |
| `DB_SSL` | `false` | Require TLS to the DB. |
| `DB_SYNC` | `false` | **Must stay `false`** — migrations own the schema, not `sync()`. |

`DB_SYNC=false` is load-bearing: `getSequelizeModuleOptions` passes it as `synchronize`, so the running app **never** auto-alters tables. Schema changes only ever happen through migrations. Likewise the test suite provisions its schema by running migrations (`db:migrate:test`), not `sequelize.sync()` — tests exercise the *real* schema, the same DDL production gets.

### 7.2 Migrations at startup: the prod entrypoint

The production container's entrypoint is `apps/api/docker-entrypoint-prod.sh`. Its logic:

1. **Fail fast** if any of `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` is missing — the container refuses to start with an incomplete DB config.
2. **Run migrations** (`sequelize-cli db:migrate`) against the compiled `dist/database/...` config and migrations. If migrations fail, the script `exit 1`s — **the app does not start on a broken schema**.
3. Only then `exec node dist/main.js`.

```
┌──────────────────────────── API pod boot ────────────────────────────┐
│ 1. check DB_* env present ──► missing? exit 1                         │
│ 2. sequelize-cli db:migrate ──► fails? print error, exit 1           │
│ 3. exec node dist/main.js  (app starts only after schema is current) │
└───────────────────────────────────────────────────────────────────────┘
```

This is why a deploy that ships a bad migration fails *loudly at startup* rather than serving traffic against a wrong schema.

### 7.3 Production: Cloud SQL behind the Auth Proxy

In production the database is **Google Cloud SQL**, instance `gibp-ledger:us-west1:gibp-ledger-postgres`. Crucially, the app pods do **not** connect to it over the public internet, and there is no Cloud SQL password floating around GCP. Instead:

```
                 GKE cluster (namespace: gibp)
  ┌──────────────┐      DB_HOST=cloudsql-proxy...       ┌────────────────────┐
  │  API pod(s)  │ ───► cloudsql-proxy.gibp.svc         │  cloudsql-proxy    │
  │  (Sequelize/ │      .cluster.local:5432   ────────► │  Deployment (pod)  │
  │   pg)        │                                      │  image 2.8.0       │
  └──────────────┘                                      └─────────┬──────────┘
                                                                  │ Workload Identity
                                                                  │ (no DB password on the wire to GCP)
                                                                  ▼
                                             Google Cloud SQL: gibp-ledger:us-west1:
                                                        gibp-ledger-postgres
```

- The **Cloud SQL Auth Proxy** runs as its own Deployment (`infra/k8s/infrastructure/cloudsql-proxy.yaml`, image `gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0`). It listens on `5432` inside the cluster and is exposed as the ClusterIP Service **`cloudsql-proxy.gibp.svc.cluster.local`**.
- The API just sets `DB_HOST` to that service name and talks plain Postgres to it. The proxy handles the encrypted, authenticated tunnel to Cloud SQL, authenticating via **Workload Identity** (`serviceAccountName: cloudsql-proxy-ksa`) — so no long-lived DB credential is presented to GCP over the wire.
- The application's own DB credentials (`DB_PASSWORD`, etc.) come from **GCP Secret Manager**, delivered into the pods by the **External Secrets Operator** — never committed, never in the image.

Why this shape: the managed database is **never publicly exposed**; the only path to it is an authenticated in-cluster tunnel. The security and delivery mechanics (Workload Identity, Secret Manager, ESO) are documented in depth in [`08-gcp.md`](08-gcp.md) and [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md) — this chapter deliberately doesn't duplicate them.

---

## 8. Production concerns

### 8.1 Connection pooling under many pods / serverless

Recall §4.7: connections are finite. In Kubernetes you may run several API replicas, each with its own `pg` pool, plus the Cloud SQL Auth Proxy, plus migration jobs, plus Formance — all consuming connections on the same Cloud SQL instance. Do the arithmetic: **(replicas × per-pod pool size) + overhead must stay comfortably below the instance's `max_connections`.** Serverless/autoscaling makes this worse because pod count spikes with traffic. Mitigations, in order:

1. **Cap the per-process pool** small and explicit.
2. **Right-size `max_connections`** on the Cloud SQL instance for the real workload.
3. **Introduce an external pooler** (PgBouncer / Cloud SQL's built-in pooling) so many app connections multiplex onto few DB connections when replica counts grow.

### 8.2 Indexes & slow queries

Slow endpoints are usually slow queries, and slow queries are usually a missing or wrong index. Workflow: reproduce the query, `EXPLAIN ANALYZE` it (§4.5), and if you see a `Seq Scan` on a large table filtered by a column, add an index on that column — ideally a composite index matching the actual `WHERE`/`ORDER BY`. This repo's `20260414121000-add-accounts-performance-indexes.js` is the template: add exactly the indexes the hot queries need, in a migration.

### 8.3 Migration safety (backwards-compatible, no long locks)

The dangerous truth: **some schema changes take a lock that blocks all reads/writes to a table for the duration of the change.** On a big table under load, a careless migration can freeze the app. Safe practice:

- **Add columns as nullable (or with a cheap default), never `NOT NULL` with a table rewrite in one shot** on a large table. Add nullable → backfill in batches → then add the `NOT NULL` constraint.
- **Build indexes concurrently**: `CREATE INDEX CONCURRENTLY ...` avoids the long write-lock a plain `CREATE INDEX` takes (at the cost of not running inside a transaction — plan the migration accordingly).
- **Make changes backwards-compatible with the currently-running code**, because during a rolling deploy the *old* app and the *new* schema coexist. Expand first (add the new thing), deploy code that uses it, then contract (remove the old thing) in a later migration.
- **Never rename/drop a column the running code still reads** in the same deploy.

See the safe-migration checklist in §9.7.

### 8.4 Backups & PITR

Cloud SQL provides automated daily backups and **point-in-time recovery** (replay WAL to any moment inside the retention window) — this is your undo button for "someone ran a bad `DELETE`." Verify the retention window is set to your recovery needs, and periodically test that a restore actually works (an untested backup is a rumor). For non-managed environments, `pg_dump` before risky changes (§5.5).

### 8.5 High availability & read replicas

- **HA / failover** — a standby that takes over if the primary fails, kept current by streaming the WAL. Cloud SQL offers this as a configuration.
- **Read replicas** — additional read-only copies to offload heavy read traffic (reports, analytics) from the primary. Beware **replication lag**: a replica may be milliseconds-to-seconds behind, so never read a value from a replica and expect it to reflect a write you just committed to the primary. For a ledger, keep *money-critical* reads on the primary.

### 8.6 Security (least privilege, SSL, no public exposure)

- **Least-privilege roles** — the app role should be able to do its job and nothing more; no superuser for the application.
- **SSL/TLS** — the `DB_SSL` flag controls whether Sequelize requires TLS; both the runtime config and the CLI config enable `ssl: { require: true, rejectUnauthorized: true }` when it's on. (With the Cloud SQL Auth Proxy, the proxy already provides an encrypted, authenticated tunnel to Cloud SQL.)
- **No public exposure** — the production DB has no public IP path from the app; the *only* route is the in-cluster proxy (§7.3). This is the single most important database-security decision in the deployment.

### 8.7 Monitoring

Watch, at minimum: connection count vs `max_connections`, replication lag, slowest queries (`pg_stat_statements`), transaction age / autovacuum health (wraparound risk), disk usage, and cache hit ratio. Cloud SQL surfaces most of these as metrics; wire the important ones to alerts.

---

## 9. Operations & debugging playbooks

### 9.1 `psql` cheat sheet

```
\l                 list databases
\c dbname          connect to a database
\dt                list tables
\d "Bills"         describe a table (columns, indexes, FKs)
\di                list indexes
\du                list roles/users
\dn                list schemas
\x                 toggle expanded (row-per-line) output — great for wide rows
\timing            show query durations
\e                 open the last query in $EDITOR
\watch 5           re-run the last query every 5 seconds
\conninfo          show current connection details
\q                 quit
```

Connect to the local repo DB (adjust to your `.env`):

```bash
docker compose exec db psql -U "$DB_USER" -d "$DB_NAME"
```

### 9.2 Find slow / expensive queries

```sql
-- Requires the pg_stat_statements extension (enable it in Cloud SQL flags).
SELECT round(total_exec_time::numeric, 1) AS total_ms,
       calls,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       query
FROM   pg_stat_statements
ORDER  BY total_exec_time DESC
LIMIT  20;
```

Then `EXPLAIN ANALYZE` the worst offenders (§4.5) and fix the plan (usually an index).

### 9.3 Check locks & blocking

When "the app is hung," a transaction is often holding a lock others are waiting on:

```sql
-- Who is blocking whom
SELECT blocked.pid          AS blocked_pid,
       blocked.query        AS blocked_query,
       blocking.pid         AS blocking_pid,
       blocking.query       AS blocking_query
FROM   pg_stat_activity blocked
JOIN   pg_stat_activity blocking
       ON blocking.pid = ANY (pg_blocking_pids(blocked.pid));
```

If a rogue session must go: `SELECT pg_cancel_backend(<pid>);` (cancel its current query) or, harder, `SELECT pg_terminate_backend(<pid>);` (kill the session).

### 9.4 Connection-limit errors (`too many connections`)

```sql
-- How many connections, by state?
SELECT state, count(*) FROM pg_stat_activity GROUP BY state ORDER BY count DESC;

-- The configured ceiling
SHOW max_connections;
```

If you see many `idle` (fine, pooled) but also many `idle in transaction` — that's a bug (§10): a transaction was left open. Trace it to the code path that forgot to commit/rollback. Structural fix: cap pools and add an external pooler (§8.1).

### 9.5 Inspect a table, its indexes and constraints

```sql
\d+ "Bills"        -- full description incl. indexes, FKs, storage
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'Bills';
SELECT relname AS table, n_live_tup AS live_rows, n_dead_tup AS dead_rows
FROM   pg_stat_user_tables ORDER BY n_dead_tup DESC;   -- bloat / vacuum health
```

### 9.6 "Migration failed mid-deploy"

The prod entrypoint (`docker-entrypoint-prod.sh`) runs `db:migrate` and **exits non-zero if it fails, so the app never starts** on a half-migrated schema. Recovery steps:

1. **Read the actual error** in the pod logs — most failures are a bad SQL statement, a constraint violation on existing data, or a lock timeout.
2. Because the repo's migrations **wrap their DDL in a transaction**, a failed migration generally rolls itself back — the schema is left as it was *before* that migration, and `SequelizeMeta` does not record it as applied. Confirm with `\dt` / `\d` and by checking `SequelizeMeta`.
3. **Fix the migration** (or the data it choked on), redeploy. The migration runner re-attempts only the un-recorded (pending) migrations.
4. If a migration is *not* fully transactional (e.g. it used `CREATE INDEX CONCURRENTLY`, which cannot run in a transaction), you may need to manually clean up a partially-created object before retrying. This is why concurrent/index-heavy steps deserve their own small migration.
5. Never hand-edit `SequelizeMeta` to "skip" a migration in production without understanding the schema state — you'll desync code and DB.

### 9.7 Safe-migration checklist

Before merging a migration, confirm:

- [ ] It has a working `down` (or a documented reason it's irreversible).
- [ ] DDL is wrapped in a transaction where possible (repo convention).
- [ ] New columns are nullable or cheaply-defaulted on large tables; `NOT NULL` is added *after* a backfill.
- [ ] Indexes on large tables use `CREATE INDEX CONCURRENTLY`.
- [ ] The change is **backwards-compatible** with the currently-deployed app (expand → migrate code → contract later).
- [ ] Foreign-key columns are indexed (§10).
- [ ] Money columns are `NUMERIC/DECIMAL`, never `float` (§10).
- [ ] You've considered the lock it takes and the table size it takes it on.
- [ ] There's a backup / PITR window covering the change.

---

## 10. Gotchas & hard-won lessons

**1. Never store money as a floating-point number.** JavaScript `number` and SQL `float`/`double`/`real` are binary floating point: `0.1 + 0.2 !== 0.3`. For money that means cents that don't add up and a ledger that won't balance — the one thing this system must never do. Two correct options:

- **Exact decimal columns** — Postgres `NUMERIC(precision, scale)` (Sequelize `DECIMAL`). This repo uses `DECIMAL(19,4)` for `billAmount` and, elsewhere, `DECIMAL(19,6)` / `DECIMAL(14,2)` for higher-precision FX/rate figures. `NUMERIC` is exact base-10; no rounding surprises.
- **Integer minor units** — store amounts as integers of the smallest currency unit (e.g. cents), so `$12.34` is the integer `1234`. No fractions exist to round.

Either is fine; **binary float is not**. And beware the driver edge: `pg` returns `NUMERIC` as a *string* (to preserve exactness) — do arithmetic with a decimal library, not by `Number()`-ing it.

**2. `sync()` / `synchronize: true` in production is dangerous.** It can silently drop or rewrite columns to match models — data loss with no audit trail. This repo hard-guards it: `DB_SYNC=false`, migrations own the schema. Keep it that way.

**3. Unindexed foreign keys bite twice.** A foreign key does *not* automatically create an index on the referencing column. Result: (a) queries filtering/joining on that FK do full scans, and (b) deleting or updating a row in the *parent* table can take a slow lock while Postgres scans the child table to enforce the constraint. Always index FK columns (the repo does this deliberately, e.g. the `Accounts` performance-index migration).

**4. `idle in transaction` sessions are silent poison.** A code path that runs `BEGIN` and then does slow work (or errors) without `COMMIT`/`ROLLBACK` holds locks and pins MVCC snapshots — which **blocks autovacuum** and causes bloat and wraparound risk (§4.6), plus it burns a connection. Keep transactions short; always commit or roll back; set a sane `idle_in_transaction_session_timeout`.

**5. Case sensitivity & quoting.** Unquoted identifiers in Postgres are folded to lowercase: `SELECT * FROM Bills` actually looks for `bills`. But Sequelize creates tables with quoted PascalCase names like `"Bills"`, `"Accounts"`, `"CustomerId"` — so in raw `psql` you must **quote them exactly**: `SELECT * FROM "Bills";` works, `SELECT * FROM Bills;` fails with "relation does not exist." When in doubt, `\dt` shows the real names.

**6. Timestamps & time zones.** Prefer **`TIMESTAMPTZ`** (`timestamp with time zone`) over `TIMESTAMP` — it stores an unambiguous instant and handles conversions, whereas naked `TIMESTAMP` is a wall-clock reading with no zone and invites off-by-hours bugs. Sequelize's `DataType.DATE` maps to `timestamp with time zone` on Postgres, which is the right default; store instants in UTC and convert at the edges. Never store local times without a zone for financial events.

**7. `NULL` is not zero and not equal to itself.** `NULL = NULL` is `NULL` (not `true`); use `IS NULL`. Aggregates skip `NULL`. A `NOT IN (subquery)` that returns any `NULL` yields no rows. Decide explicitly whether a column is nullable, and query accordingly.

**8. Soft deletes change every query's meaning.** With `paranoid: true`, "deleted" rows still physically exist with a `deletedAt`. Sequelize hides them by default, but raw SQL and reports do *not* — filter `deletedAt IS NULL` yourself when writing SQL, or you'll double-count. (The repo indexes on `(OrganizationId, deletedAt)` precisely because this filter is ever-present.)

---

## 11. Glossary + further reading

### Glossary (quick recall)

- **ACID** — Atomicity, Consistency, Isolation, Durability: the transaction guarantees.
- **B-tree** — the default balanced-tree index; fast equality and range lookups.
- **Bloat** — wasted space from dead row versions not yet vacuumed.
- **Cloud SQL** — Google's managed Postgres service; used in this repo's production.
- **Cloud SQL Auth Proxy** — sidecar/pod that provides an authenticated encrypted tunnel to Cloud SQL; here it's `cloudsql-proxy.gibp.svc.cluster.local:5432`.
- **DDL / DML** — Data Definition (schema) / Data Manipulation (rows) SQL.
- **Foreign key (FK)** — a column constrained to match a primary key elsewhere.
- **Isolation level** — how much concurrent transactions see of each other (Read Committed → Serializable).
- **Migration** — an ordered, versioned schema-change script; the schema's source of truth here.
- **MVCC** — Multi-Version Concurrency Control; readers and writers don't block via row versioning.
- **NUMERIC / DECIMAL** — exact base-10 numeric type; the correct type for money.
- **ORM** — Object-Relational Mapper (Sequelize) mapping rows ↔ objects.
- **Paranoid / soft delete** — mark rows deleted via `deletedAt` instead of removing them.
- **PITR** — Point-In-Time Recovery via WAL replay.
- **Pool** — a reused set of open connections shared by an app.
- **Role** — a database permission identity; a `LOGIN` role is a "user."
- **Sequelize** — the Node ORM used here (`6.37.7`), with `sequelize-typescript` decorators and `sequelize-cli` migrations.
- **VACUUM / autovacuum** — reclaims dead tuples and refreshes planner statistics.
- **WAL** — Write-Ahead Log; the basis of durability, PITR, and replication.

### Further reading

**PostgreSQL (official):**
- Docs home — https://www.postgresql.org/docs/current/
- Tutorial — https://www.postgresql.org/docs/current/tutorial.html
- Transactions & concurrency control (MVCC, isolation) — https://www.postgresql.org/docs/current/mvcc.html
- Write-Ahead Logging (WAL) — https://www.postgresql.org/docs/current/wal-intro.html
- Indexes — https://www.postgresql.org/docs/current/indexes.html
- `EXPLAIN` / using EXPLAIN — https://www.postgresql.org/docs/current/using-explain.html
- Routine vacuuming — https://www.postgresql.org/docs/current/routine-vacuuming.html
- Numeric types (`NUMERIC`) — https://www.postgresql.org/docs/current/datatype-numeric.html
- Date/time types (`timestamptz`) — https://www.postgresql.org/docs/current/datatype-datetime.html
- `psql` reference — https://www.postgresql.org/docs/current/app-psql.html
- `pg_dump` / `pg_restore` — https://www.postgresql.org/docs/current/app-pgdump.html

**Drivers / ORM (as used here):**
- `node-postgres` (`pg`) — https://node-postgres.com/
- Sequelize (v6) — https://sequelize.org/docs/v6/
- `sequelize-typescript` — https://github.com/sequelize/sequelize-typescript
- Sequelize CLI & migrations — https://sequelize.org/docs/v6/other-topics/migrations/
- `@nestjs/sequelize` (NestJS integration) — https://docs.nestjs.com/recipes/sql-sequelize

**Google Cloud SQL (managed path):**
- Cloud SQL for PostgreSQL — https://cloud.google.com/sql/docs/postgres
- Cloud SQL Auth Proxy — https://cloud.google.com/sql/docs/postgres/sql-proxy

**In this handbook / repo:**
- Deployment, secrets, and the proxy in practice — [`../deployment-lifecycle-guide.md`](../deployment-lifecycle-guide.md)
- GCP, Secret Manager, External Secrets, Workload Identity — [`08-gcp.md`](08-gcp.md)
- Repo coding standards (single source, no magic strings, DTO validation) — [`../engineering-guidelines.md`](../engineering-guidelines.md)
