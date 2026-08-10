# 05 — Redis: the shared in-memory store

**What this chapter is.** A complete, from-zero book on Redis — the in-memory data
store used across our industry for caching, sessions, distributed locks, rate
limiting, pub/sub, and as the substrate under job queues. By the end you will
understand *what* Redis is, *why* it exists, *how* it works internally, how to run
and operate it, how to use it safely from Node.js, and exactly where it would fit
into this repository. You will be able to add Redis to *any* project from a blank
slate.

> **Status in this repo: NOT used (see §7).**
> As of this writing the GIBP ledger codebase does **not** run, connect to, or
> depend on Redis anywhere. There is no `redis` or `ioredis` package, no Redis
> service in our infrastructure, and no connection code. The *only* trace of Redis
> is a single `// TODO` comment at
> `apps/api/src/modules/accounts/accounts.service.ts:1757` that notes the current
> per-process balance cache is not shared across API pods and muses that "a
> distributed cache like Redis or Memcached" would help. Sections 1–6 and 8–11 are
> a standard, standalone Redis education. Section 7 is the honest, grounded story of
> where Redis *would* slot in — written as a proposal, not a description of reality.
> Do not read this chapter as evidence that we use Redis. We do not, yet.

**How to read this.** If you have never touched Redis, read straight through §1→§6
once — each section opens with the *problem* before the *solution*, so the tools
earn their place. §7 is our repo-specific proposal. §8–§10 are what you reach for
when you actually run Redis in production, and §9/§10 are the pages you will come
back to at 2 a.m. §11 is the glossary and links. Every command block is runnable;
every code block is real.

---

## 1. The problem: fast data, slow disks, and caches that don't share

Start with the uncomfortable truth about databases: **they are durable but
relatively slow.** A PostgreSQL query, or a call out to an external ledger service
like Formance, has to touch disk, cross a network, parse SQL or JSON, check
constraints, and coordinate transactions. That might take 5 ms, or 50 ms, or 500 ms
under load. That is completely fine for the *authoritative* copy of your data — the
place where correctness lives. But it is a terrible fit for three other shapes of
data that show up in every real system:

1. **Hot data read constantly.** The same account balance, the same product page,
   the same config flag, asked for thousands of times a second. Recomputing or
   re-fetching it every time is wasteful — the answer barely changed.

2. **Ephemeral data that doesn't belong in your main DB.** A user's login session.
   A "how many requests has this IP made in the last minute" counter. A short-lived
   lock. Writing these to Postgres pollutes your schema, hammers your disk with
   throwaway writes, and clutters your backups with data you don't care about
   surviving.

3. **Coordination that must be fast and shared across many app instances.** "Only
   one worker should process this job." "Only send this email once." When you run
   more than one copy of your application — as we do; the GIBP `api` runs on
   **2+ pods** behind a load balancer — you cannot coordinate them using a variable
   in one process's memory. The other pods can't see it.

### The trap this repo is standing in right now

Point 1 and point 3 combine into the exact situation described by the TODO in our
codebase. Our `AccountsService` caches account balances **in the memory of a single
Node.js process** — a plain JavaScript `Map`. That works beautifully on your laptop,
where there is one process. In production there are two or more pods, and here is
what actually happens:

```
                    Load Balancer (round-robins requests)
                    /                                    \
        ┌───────── api pod A ─────────┐        ┌───────── api pod B ─────────┐
        │  in-memory Map (its own)    │        │  in-memory Map (its own)    │
        │  balance:USD::acct1 = 500   │        │        (empty for acct1)    │
        └─────────────────────────────┘        └─────────────────────────────┘
                    │                                        │
        request 1 → pod A: cache HIT (fast)      request 2 → pod B: cache MISS
                                                 pod B must call Formance again
```

Pod A warms its cache and serves fast. The next request lands on pod B, whose `Map`
knows nothing about that account, so pod B calls Formance again. Each pod maintains
its own private, partial copy. You get **cache misses that shouldn't happen and
redundant calls to the source of truth** — precisely what the TODO warns about. And
it gets worse: if pod A learns a balance changed (because someone posted a new
transaction) and clears *its* cache entry, pod B still serves the stale value,
because there is no shared place to record "this is now invalid."

A per-process cache can never solve a cross-process problem. What you need is a
single, **shared** store that *every* pod can read and write instantly — one that
lives outside all of them, in memory, reachable over the network in microseconds.

**Enter Redis:** a shared, in-memory, sub-millisecond data store. One place, visible
to every pod, that holds hot data, ephemeral data, and coordination primitives — and
optionally photographs itself to disk so it survives a restart. That is the entire
reason Redis exists, and it is the gap sitting in our accounts service today.

---

## 2. Mental model: a giant shared whiteboard in RAM

Here is the one picture to hold in your head:

> **Redis is a giant whiteboard on the wall of a room, held entirely in RAM, that
> every app instance in the building can walk up to and read from or write to
> instantly. Occasionally someone takes a photograph of the whiteboard so that if
> the room burns down, you can redraw it.**

Map the analogy back to the real terms — this is the whole chapter in miniature:

| Analogy | Real Redis thing | Why it matters |
|---|---|---|
| The whiteboard | A single Redis server process | One shared surface, not one-per-app |
| "on the wall of the room" | A network service (default TCP port `6379`) | Every pod reaches the *same* one over the network |
| "in RAM" | Data lives in memory, not on disk | Reads/writes are microseconds, not milliseconds |
| Every app can walk up to it | Many clients connect concurrently | Solves the cross-pod sharing problem from §1 |
| A labelled note on the board | A **key** and its **value** | Redis is fundamentally a key→value store |
| A note that fades after an hour | **TTL / expiry** | Ephemeral data cleans itself up |
| The board is finite | **maxmemory** limit | You must decide what to erase when it fills |
| Erasing old notes to make room | **eviction policy** (e.g. LRU) | Redis can forget on its own |
| Photographing the board | **persistence** (RDB / AOF) | Survives restarts — but it's a *photo*, a backup, not the primary ledger |
| One person writes at a time | **single-threaded** command execution | Each command is atomic; no half-writes |

Two consequences fall straight out of this picture, and they govern almost
everything else:

- **Because it's in RAM, it's fast but finite and volatile.** RAM is expensive and
  smaller than disk. RAM also vanishes on power loss. So Redis is not where you keep
  the *only* copy of anything you can't afford to lose. In our world: **Redis is
  never the source of truth for money.** The ledger (Formance / Postgres) is. Redis
  is a fast helper standing in front of it.

- **Because it's shared over the network, everyone sees the same thing.** That is the
  superpower a per-process `Map` can never have, and the whole point of §7.

Keep this whiteboard in mind. Every fancy feature later — sorted sets, Lua scripts,
streams, Sentinel — is just a smarter kind of note or a way to keep the board
consistent when there are several boards.

---

## 3. Core concepts & vocabulary

Redis has a small, sharp vocabulary. Learn these words once and the docs read easily
forever. Skim the table, then read the notes under it.

| Term | One-line meaning |
|---|---|
| **Key** | A unique string label, e.g. `balance:USD::acct1`. Everything is found by key. |
| **Value** | The data stored under a key. Its *type* determines what you can do with it. |
| **String** | The simplest value: text or bytes or a number. Up to 512 MB. Counters live here. |
| **Hash** | A map *inside* a value: field→value pairs, like a small object. Great for records. |
| **List** | An ordered sequence you push/pop from either end. A cheap queue or stack. |
| **Set** | An unordered collection of unique members. Membership tests, deduping. |
| **Sorted set (ZSET)** | A set where every member has a numeric **score**; kept sorted by score. Leaderboards, priority queues, time-ordered indexes. |
| **Stream** | An append-only log of entries with IDs and consumer groups. A lightweight message queue / event log. |
| **Bitmap** | A string treated as an array of bits. Compact flags: "did user N do X?" |
| **HyperLogLog (HLL)** | A tiny probabilistic counter for "how many *unique* things?" using ~12 KB regardless of count. |
| **TTL / expiry** | An optional "self-destruct timer" on a key. After it elapses, the key vanishes. |
| **Eviction policy** | The rule Redis uses to delete keys when memory is full (e.g. `allkeys-lru`). |
| **Persistence** | Saving memory to disk so data survives restarts: **RDB** (snapshots) and **AOF** (append log). |
| **Single-threaded event loop** | Redis runs your commands one at a time on one thread. Simple, fast, atomic per command. |
| **Pipelining** | Sending many commands at once without waiting for each reply — cuts network round-trips. |
| **Transaction (MULTI/EXEC)** | Queue several commands and run them as one uninterrupted batch. |
| **Lua script** | A small script Redis runs atomically server-side — the tool for "read-then-write as one step." |
| **Pub/Sub** | Publish messages to channels; subscribers get them live. Fire-and-forget fan-out. |
| **Keyspace notifications** | Events Redis can emit when keys change/expire, delivered via Pub/Sub. |
| **Replication** | A **replica** keeps a live copy of a **primary** for reads / failover. |
| **Sentinel** | A watchdog system that detects a dead primary and promotes a replica automatically. |
| **Cluster** | Redis sharded across many nodes, splitting keys by **hash slot** for scale beyond one machine's RAM. |

A few notes that repay reading now rather than later:

- **Types are the point, not an afterthought.** A beginner reaches for strings for
  everything. But storing a user record as a **hash** lets you update one field
  without rewriting the whole blob; a **sorted set** gives you a leaderboard or a
  "next N due items" index for free; a **set** gives you O(1) "have I seen this?".
  Choosing the right type is most of using Redis well.

- **TTL is what makes Redis safe for ephemeral data.** Sessions, rate counters, and
  cache entries should almost always carry an expiry so they clean themselves up.
  *Forgetting* the TTL is the #1 cause of a Redis instance slowly filling with
  garbage (see §10).

- **Single-threaded is a feature, not a limitation, until it isn't.** Because one
  command runs at a time, every single command is atomic — no locking needed for
  `INCR`. But a *slow* command (say, sorting a million-element list) blocks
  *everyone* while it runs. §4 makes this concrete.

- **Persistence is a safety net, not a promise.** Redis can save to disk, but by
  default you can still lose the last few seconds of writes on a crash. That is fine
  for a cache and unacceptable for a ledger — which is exactly why the ledger stays
  in Postgres/Formance and Redis only ever *fronts* it.

---

## 4. How it actually works (the deep version)

This section is the "why," and understanding it is what separates someone who uses
Redis from someone who gets paged by it.

### 4.1 The single-threaded event loop — why it's fast, and its trap

Redis handles commands on **a single thread**, using an event loop (the same idea as
Node.js). It reads a command off a socket, executes it to completion, writes the
reply, then moves to the next. There is no multi-core parallelism for command
execution and, crucially, **no locking** — because only one command touches the data
at any instant.

Why is that fast? Because the expensive things in a database — disk seeks, lock
contention, context switches between threads — are largely absent. Everything is in
RAM, and there is nothing to lock against. A simple `GET` or `SET` completes in
*microseconds*. Redis routinely serves hundreds of thousands of operations per
second from one core.

The trap is the flip side: **a slow command blocks every other client.** If you run
an O(N) command against a huge key — `KEYS *` on a database with a million keys,
`LRANGE biglist 0 -1` on a giant list, `SMEMBERS` on a massive set — Redis is busy
walking that whole structure and *cannot answer anyone else* until it's done. Your
p99 latency spikes; health checks time out; it looks like an outage.

> **Rule that follows directly:** avoid O(N)-over-a-big-key commands on the hot path.
> Use `SCAN` instead of `KEYS` (it returns results in small chunks). Keep individual
> keys reasonably sized. Never `LRANGE 0 -1` a list you let grow unbounded. §10
> lists the worst offenders.

### 4.2 Memory and eviction — Redis is a bounded box

Everything lives in RAM, which is finite. You cap it with `maxmemory` (e.g. 2gb).
When Redis hits that cap, its behaviour depends on the **eviction policy**:

| Policy | What it does when full | Use it when |
|---|---|---|
| `noeviction` (default) | Rejects writes with an error; reads still work | Redis holds data you must not silently drop |
| `allkeys-lru` | Evicts the least-recently-used key, any key | **Pure cache** — the classic choice |
| `allkeys-lfu` | Evicts the least-*frequently*-used key | Cache where some keys are perennially hot |
| `volatile-lru` | Evicts LRU **only among keys that have a TTL** | Mixed use: cache entries (with TTL) evictable, other data protected |
| `volatile-ttl` | Evicts the key expiring soonest | You want the shortest-lived stuff to go first |
| `allkeys-random` / `volatile-random` | Evicts a random key | Rarely the right answer |

- **LRU** = Least Recently Used. Analogy: your desk is full, so you file away the
  paper you haven't touched in the longest time.
- **LFU** = Least Frequently Used: you file away the paper you *reach for* least
  often, even if you touched it recently.

The choice matters. If you run Redis as a **cache**, `allkeys-lru` (or `-lfu`) means
"when full, forget the coldest thing" — graceful. If you run Redis with the default
`noeviction` and it fills up, **writes start failing**, which can take down anything
that depends on writing to it. Decide this on purpose. (§8, §9.)

### 4.3 Persistence — RDB vs AOF, and why "durable-ish"

Redis is in-memory, but it can write to disk so a restart doesn't start empty. Two
mechanisms, often used together:

- **RDB (Redis Database) snapshots.** Every so often (e.g. "if ≥1000 keys changed in
  the last 5 minutes"), Redis forks and writes a compact binary *photograph* of the
  whole dataset to a `.rdb` file. Pros: small files, fast restart, cheap. Con: you
  lose everything written **since the last snapshot** if you crash — potentially
  minutes of data.

- **AOF (Append Only File).** Redis appends every write command to a log file. On
  restart it replays the log. With `appendfsync everysec` (the common setting) you
  lose at most ~1 second of writes on a crash. Pros: much better durability. Cons:
  bigger files, slightly slower, needs periodic *rewrite/compaction* so the log
  doesn't grow forever.

| | RDB | AOF |
|---|---|---|
| Shape | Point-in-time snapshot (photo) | Continuous log of writes (journal) |
| Worst-case data loss | Minutes | ~1 second (`everysec`) |
| Restart speed | Fast | Slower (replays log) |
| File size | Small | Larger (until rewritten) |
| Typical use | Backups, fast recovery | Durability |

**The honest headline:** even AOF-with-everysec can lose ~1 second of writes. Redis
persistence is a *safety net for a cache or a queue*, not a guarantee for
money-grade records. This is the technical reason Redis must never be the sole home
of ledger data. Keep the real ledger in Postgres/Formance; let Redis lose a second
of *cache* and shrug.

### 4.4 Atomicity: single commands, MULTI/EXEC, and Lua

Because of the single thread, **every individual command is atomic.** `INCR
counter` reads, adds one, and writes back with zero chance of another client
interleaving. That is why Redis counters are safe under massive concurrency without
any lock.

But real logic often needs *several* commands to happen as one indivisible step —
"read this value, and only if it's X, set it to Y." Between your `GET` and your
`SET`, another client could sneak in. Redis gives you three tools:

- **`MULTI` / `EXEC` (transactions).** Queue commands after `MULTI`; `EXEC` runs
  them back-to-back with nothing interleaved. Note: it's *not* rollback-on-error like
  SQL — it's "run this batch without interruption," optionally guarded by `WATCH`
  (optimistic locking: abort if a watched key changed).

- **Lua scripts (`EVAL`).** You send a small Lua script; Redis runs the *whole
  script* atomically on its single thread. This is the real workhorse for
  "read-decide-write as one step" — safe locks, token-bucket rate limiters, and
  conditional updates are all Lua one-liners. If you remember one advanced tool,
  remember this one.

- **Native compound commands.** Many "check and act" needs already exist as one
  atomic command: `SET key val NX PX 30000` = "set only if absent, with a 30s
  expiry" — the foundation of a distributed lock (§6.3).

### 4.5 Replication & failover — one box isn't enough for HA

A single Redis is a single point of failure. Redis supports **replication**: a
**primary** streams its writes to one or more **replicas** that hold live copies.
Replicas serve reads and stand ready to take over. Two ways to make that automatic:

- **Sentinel.** A set of watchdog processes that monitor the primary. If it dies,
  they agree it's dead, **promote a replica to primary**, and tell clients the new
  address. Same single dataset, now highly available. Best when your data fits on one
  machine and you just need failover.

- **Cluster.** For when your data (or throughput) is *too big for one machine*.
  Cluster **shards** the keyspace across many primaries using **16384 hash slots** —
  each key hashes to a slot, each slot lives on one primary (with its own replicas).
  You scale horizontally, but you accept constraints: multi-key operations must land
  on the same slot (use *hash tags* like `{userid}` to force related keys together),
  and it's more moving parts. Reach for Cluster when a single node's RAM or CPU is
  genuinely the ceiling — not before.

Managed services hide most of this. Google Cloud **Memorystore for Redis**, for
example, offers a Standard (HA) tier that runs a replica and does automatic failover
for you — you get Sentinel-grade availability without operating Sentinel (see §7, §8).

### 4.6 When Redis is the *wrong* tool

First-principles honesty, because half of using a tool well is knowing when not to:

- **Not your source of truth for anything you can't lose** — especially money. It's
  in-memory and only "durable-ish." The ledger stays authoritative in
  Postgres/Formance.
- **Not a relational database.** No joins, no ad-hoc queries across your data, no
  foreign keys. If you need to *query* by many dimensions, that's Postgres.
- **Not for data much larger than RAM.** RAM is expensive; if your dataset is
  terabytes of cold data, that's a disk-based store, not Redis.
- **Not a durable message broker for critical events** out of the box. Streams are
  good, but if you need strong delivery guarantees and huge backlogs, evaluate Kafka.
  For app-level *background jobs*, a Redis-backed queue like BullMQ is the right size
  (see §7 and **[06-bullmq.md](./06-bullmq.md)**).

Redis is a brilliant *accelerator and coordinator* sitting in front of your durable
systems. Treat it as that and it will never surprise you.

---

## 5. Setup from scratch (hands-on)

Let's actually run one. You need Docker. Everything below is copy-paste runnable.

### 5.1 The fastest possible Redis

```bash
# Run Redis 7 (alpine = tiny image), listening on the default port 6379.
docker run --name redis-lab -p 6379:6379 -d redis:7-alpine

# Talk to it with the built-in CLI, which ships inside the same image:
docker exec -it redis-lab redis-cli

# You are now at a redis> prompt. Try:
127.0.0.1:6379> PING
PONG
```

`PONG` means it's alive. That's a working Redis. But it has **no password**, **no
persistence**, and **no memory limit** — fine for a five-minute experiment, unsafe
for anything real. Let's do it properly.

### 5.2 A real setup: password, persistence, memory cap, config file

Create a config file. This is a **minimal but production-shaped `redis.conf`** —
read the comments; they *are* the lesson:

```conf
# ── redis.conf ────────────────────────────────────────────────
# Require a password. Clients must AUTH before doing anything.
requirepass CHANGE_ME_TO_A_LONG_RANDOM_SECRET

# Cap memory so Redis can never eat the whole machine.
maxmemory 512mb

# When full, evict the least-recently-used key (cache behaviour).
# Use 'noeviction' instead if this Redis holds data you must not drop.
maxmemory-policy allkeys-lru

# Durability: append every write to a log (AOF), fsync ~once a second.
# Worst-case loss on crash ≈ 1 second of writes.
appendonly yes
appendfsync everysec

# Also keep periodic RDB snapshots (belt and braces / fast restart + backups).
save 900 1        # snapshot if ≥1 key changed in 900s
save 300 10       # ...or ≥10 keys in 300s
save 60 10000     # ...or ≥10000 keys in 60s

# Only listen on loopback by default; the container maps the port out.
# NEVER bind Redis to a public interface without auth + a firewall (§8).
bind 0.0.0.0
protected-mode yes
# ──────────────────────────────────────────────────────────────
```

Run it with that config and a **named volume** so the AOF/RDB files survive
container restarts:

```bash
# Assume redis.conf is in the current directory.
docker run --name redis-prod -d \
  -p 6379:6379 \
  -v redis-data:/data \
  -v "$(pwd)/redis.conf":/usr/local/etc/redis/redis.conf \
  redis:7-alpine \
  redis-server /usr/local/etc/redis/redis.conf

# Connect (note the password flag). In real life pass it via env, not on the CLI,
# so it doesn't land in your shell history.
docker exec -it redis-prod redis-cli -a 'CHANGE_ME_TO_A_LONG_RANDOM_SECRET'
```

> **`--requirepass` alternative:** if you don't want a full config file, you can pass
> the password inline: `redis-server --requirepass 'SECRET' --appendonly yes`. Same
> effect for the auth bit. The config-file approach scales better as you add settings.

### 5.3 Core commands — a guided tour of each data type

At the `redis-cli` prompt, walk through the types. This is your fluency drill.

```bash
# ── Strings & counters ───────────────────────────────────────
SET greeting "hello"            # store a string
GET greeting                    # -> "hello"
SET session:token "xyz" EX 30   # store WITH a 30-second TTL
TTL session:token               # -> 30 (seconds left); -1 = no TTL; -2 = gone
EXPIRE greeting 60              # add/replace a TTL on an existing key
DEL greeting                    # delete a key

INCR page:views                 # atomically 0 -> 1 (creates it as a number)
INCR page:views                 # -> 2
INCRBY page:views 10            # -> 12  (atomic add)

# ── Hashes (a small object under one key) ────────────────────
HSET user:1 name "Ada" role "admin"
HGET user:1 role                # -> "admin"
HGETALL user:1                  # -> name/Ada, role/admin
HINCRBY user:1 loginCount 1     # atomic per-field counter

# ── Lists (ordered; push/pop either end) ─────────────────────
LPUSH queue:jobs "job1"         # push to the head
RPUSH queue:jobs "job2"         # push to the tail
LRANGE queue:jobs 0 -1          # read all (fine when small!)
RPOP queue:jobs                 # pop from the tail

# ── Sets (unique, unordered) ─────────────────────────────────
SADD seen:ips "1.2.3.4"
SADD seen:ips "1.2.3.4"         # -> 0 (already present; deduped)
SISMEMBER seen:ips "1.2.3.4"    # -> 1 (yes)
SCARD seen:ips                  # -> count

# ── Sorted sets (members with scores, kept in order) ─────────
ZADD leaderboard 100 "alice"
ZADD leaderboard 250 "bob"
ZADD leaderboard 175 "cleo"
ZREVRANGE leaderboard 0 2 WITHSCORES   # top 3 by score, high to low

# ── Inspecting keys safely ───────────────────────────────────
SCAN 0 MATCH "user:*" COUNT 100 # iterate keys in chunks (NEVER use KEYS in prod)
TYPE user:1                     # -> hash
MEMORY USAGE leaderboard        # bytes this key consumes
```

### 5.4 Verify it works (including that persistence and auth are real)

```bash
# 1) Auth is enforced: connecting without the password should be refused.
docker exec -it redis-prod redis-cli PING
#   -> (error) NOAUTH Authentication required.   ✅ good, that's what we want

# 2) With auth, it answers:
docker exec -it redis-prod redis-cli -a 'CHANGE_ME_TO_A_LONG_RANDOM_SECRET' PING
#   -> PONG

# 3) Persistence survives a restart: write, restart the container, read it back.
docker exec -it redis-prod redis-cli -a 'CHANGE_ME_TO_A_LONG_RANDOM_SECRET' SET survivor "I lived"
docker restart redis-prod
sleep 1
docker exec -it redis-prod redis-cli -a 'CHANGE_ME_TO_A_LONG_RANDOM_SECRET' GET survivor
#   -> "I lived"   ✅ AOF/RDB replayed on startup

# 4) Overall health snapshot:
docker exec -it redis-prod redis-cli -a 'CHANGE_ME_TO_A_LONG_RANDOM_SECRET' INFO server | head
```

If all four behave as annotated, you have a correctly configured Redis: authenticated,
memory-capped, evicting gracefully, and persisting across restarts.

### 5.5 TLS and auth notes (read before you deploy anything)

- **Always set a password** (`requirepass`) — or, better on Redis 6+, use **ACLs**
  (`ACL SETUSER`) to give each client a named user with least-privilege command
  access. A wide-open Redis on a reachable network gets found and abused within
  minutes.
- **Encrypt in transit with TLS** when Redis traffic crosses any untrusted boundary.
  Redis supports TLS natively (`--tls-port`, certs). Managed services like
  Memorystore offer an in-transit-encryption toggle so you don't manage certs by
  hand.
- **Never expose Redis to the public internet.** Bind it to the private network only,
  put it behind a firewall / VPC, and let *only* your app reach it. Historically the
  single biggest Redis incident class is "unauthenticated Redis reachable from the
  internet." (§8.)

---

## 6. Using it in code + the patterns that matter (Node.js)

Two mature clients dominate the Node ecosystem:

- **`ioredis`** — batteries-included, first-class Cluster/Sentinel support, built-in
  Lua-script helpers (`defineCommand`), robust auto-reconnect. The common choice for
  serious apps and the one BullMQ builds on.
- **`node-redis`** (the official `redis` package) — modern promise API, lighter.

Examples below use **ioredis** unless noted, because its ergonomics suit the patterns.

### 6.1 Connecting + graceful failure

The cardinal rule: **your app must not fall over because Redis blipped.** Redis is an
*accelerator*; if it's briefly unavailable you degrade (go slower, hit the real
source) rather than crash.

```ts
import Redis from 'ioredis';

// Prefer a single URL from config/secrets, e.g. redis://:password@host:6379
// (In this repo that would come from GCP Secret Manager via ESO — see §7.)
export const redis = new Redis(process.env.REDIS_URL!, {
  // Don't queue commands forever while disconnected; fail fast and fall back.
  maxRetriesPerRequest: 2,
  enableOfflineQueue: false,
  // Exponential-ish backoff on reconnect (ms), capped.
  retryStrategy: (attempt) => Math.min(attempt * 200, 2000),
});

redis.on('error', (err) => {
  // Log, alert — but do NOT let this take the process down.
  console.error('[redis] connection error', err.message);
});
redis.on('connect', () => console.log('[redis] connected'));

// A tiny helper that treats Redis as best-effort:
async function safeGet(key: string): Promise<string | null> {
  try {
    return await redis.get(key);
  } catch (err) {
    console.warn('[redis] GET failed, degrading to source', (err as Error).message);
    return null; // caller will fetch from the real source of truth
  }
}
```

### 6.2 Cache-aside (read-through) with TTL — the pattern you'll use most

**Problem:** an expensive fetch (DB query, external API) is called constantly for the
same key. **Solution:** check Redis first; on a miss, fetch, store with a TTL, return.
This is *cache-aside* — the application, not Redis, orchestrates the caching.

```ts
async function getWithCache<T>(
  key: string,
  ttlSeconds: number,
  fetchFromSource: () => Promise<T>,
): Promise<T> {
  const cached = await safeGet(key);
  if (cached !== null) {
    return JSON.parse(cached) as T; // cache HIT
  }

  const fresh = await fetchFromSource(); // cache MISS -> go to source of truth
  // Best-effort write-back; a failed cache write must not fail the request.
  try {
    await redis.set(key, JSON.stringify(fresh), 'EX', ttlSeconds);
  } catch { /* degrade silently: we still return the fresh value */ }
  return fresh;
}

// Usage:
const user = await getWithCache(`user:${id}`, 300, () => db.users.findById(id));
```

The **TTL is doing quiet, essential work**: it bounds staleness ("worst case, this is
5 minutes old") and it guarantees the entry eventually leaves, so the cache can't grow
without limit. Choose the TTL from how stale the data may safely be.

### 6.3 Write-through / invalidation — keeping the cache honest

A cache that's read but never invalidated will serve stale data forever. Two
approaches when the underlying data changes:

- **Invalidate (delete) on write** — simplest, safest default. After you write the
  source of truth, delete the cache key so the next read re-fetches fresh:

  ```ts
  async function updateUser(id: string, patch: UserPatch) {
    await db.users.update(id, patch);   // 1) write the SOURCE OF TRUTH first
    await redis.del(`user:${id}`);      // 2) then invalidate the cache
  }
  ```

- **Write-through** — write the source *and* update the cache in the same operation.
  Keeps the cache warm but is more code and more failure modes.

> **Ordering matters.** Update the source of truth *first*, then the cache. If you
> update the cache first and the DB write fails, your cache now lies. And even with
> the right order there's a subtle race (a concurrent read can re-populate the cache
> with the old value between your DB write and your `DEL`); the pragmatic mitigations
> are short TTLs and, where correctness is critical, deleting *after* the write plus a
> brief second delete. Cache invalidation is famously the hard part — §8 dwells on it
> because for a *ledger* it's the whole ballgame.

### 6.4 Distributed lock — "only one pod does this at a time"

**Problem:** across many pods, exactly one should run some critical section (send an
invoice once; process a job once). **Solution:** a lock key that only one holder can
own, set atomically with `SET NX PX` (set-if-absent, with an auto-expiry so a crashed
holder can't deadlock forever).

```ts
import { randomUUID } from 'crypto';

async function withLock<T>(
  resource: string,
  ttlMs: number,
  fn: () => Promise<T>,
): Promise<T | null> {
  const lockKey = `lock:${resource}`;
  const token = randomUUID(); // proves WE are the owner when releasing

  // Acquire: set only if absent (NX), auto-expire after ttlMs (PX).
  const ok = await redis.set(lockKey, token, 'PX', ttlMs, 'NX');
  if (ok !== 'OK') return null; // someone else holds it; back off / skip

  try {
    return await fn();
  } finally {
    // Release SAFELY: only delete if the value is still OUR token, atomically,
    // so we never delete a lock that expired and was re-acquired by someone else.
    const release = `
      if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('del', KEYS[1])
      else
        return 0
      end`;
    try { await redis.eval(release, 1, lockKey, token); } catch { /* lock will expire on its own */ }
  }
}
```

Two things make this correct: the **unique token** (so you only release *your* lock,
never someone else's), and the **Lua release** (check-and-delete as one atomic step).

> **Redlock caveat — be honest about locks.** A single-Redis lock is fine for
> "prevent duplicate work" where a rare double-execution is merely wasteful. It is
> **not** a bulletproof mutex: if the holder pauses (GC, scheduler) past the TTL, the
> lock expires and two workers can run at once. The multi-node **Redlock** algorithm
> hardens this, but even Redlock is [debated](#11-glossary--further-reading) among
> distributed-systems experts. The safe stance: use Redis locks for *optimization and
> deduplication*, and make the protected operation **idempotent** so a rare double-run
> is harmless. Never lean on a Redis lock as the *only* thing standing between you and
> double-spending money — enforce that invariant in the ledger itself.

### 6.5 Rate limiting — INCR + EXPIRE, and the token bucket

**Problem:** cap how often a client may act ("100 requests per minute per IP").
**Simple fixed-window** version:

```ts
async function allowRequest(ip: string, limit = 100, windowSec = 60): Promise<boolean> {
  const key = `rate:${ip}:${Math.floor(Date.now() / 1000 / windowSec)}`;
  const count = await redis.incr(key);         // atomic increment
  if (count === 1) await redis.expire(key, windowSec); // set TTL on first hit
  return count <= limit;
}
```

`INCR` is atomic, so this is race-safe across all pods. The `EXPIRE` on first hit
makes the window self-clean. The weakness of fixed windows is burstiness at window
edges; a **token-bucket** limiter smooths that, and because it needs read-decide-write
atomically it's a natural **Lua script**:

```lua
-- KEYS[1]=bucket key, ARGV: rate(perSec), burst(capacity), now(ms), cost
local data   = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(data[1]) or tonumber(ARGV[2])   -- start full
local ts     = tonumber(data[2]) or tonumber(ARGV[3])
local rate, burst, now, cost = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4])
tokens = math.min(burst, tokens + (now - ts) / 1000 * rate)  -- refill by elapsed time
local allowed = tokens >= cost
if allowed then tokens = tokens - cost end
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(burst / rate * 1000))
return allowed and 1 or 0
```

The whole refill-and-consume happens atomically inside Redis — no client-side race.

### 6.6 Sessions

**Problem:** HTTP is stateless, but a logged-in user must be recognised across
requests that may hit *different* pods. **Solution:** store the session server-side in
Redis, keyed by a random session id held in the user's cookie, with a TTL that acts as
idle-timeout. Every pod reads the same session from the same Redis.

```ts
// On login:
await redis.set(`sess:${sessionId}`, JSON.stringify({ userId, role }), 'EX', 3600);
// On each request:
const raw = await redis.get(`sess:${sessionId}`);
const session = raw ? JSON.parse(raw) : null; // null -> not logged in / expired
// Sliding expiry: refresh the TTL on activity
if (session) await redis.expire(`sess:${sessionId}`, 3600);
```

### 6.7 Leaderboards (ZSET) and Pub/Sub

**Leaderboard** — the sorted set is purpose-built: `ZADD` to set a score, `ZREVRANGE`
/ `ZRANK` to read ranks, all in log time regardless of size:

```ts
await redis.zadd('leaderboard', 250, 'bob');
const top10 = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');
const bobsRank = await redis.zrevrank('leaderboard', 'bob'); // 0-based rank
```

**Pub/Sub** — fire-and-forget fan-out: publishers send to a channel, every current
subscriber receives it. Use it for live notifications, cache-invalidation broadcasts,
or "tell all pods to reload config." *Caveat:* it's **at-most-once** — if no one is
subscribed at that instant, the message is gone. For durable delivery, use Streams.

```ts
// Subscriber pod:
const sub = new Redis(process.env.REDIS_URL!);
await sub.subscribe('cache:invalidate');
sub.on('message', (channel, msg) => { localCache.delete(msg); });

// Publisher pod (after updating the source of truth):
await redis.publish('cache:invalidate', `user:${id}`);
```

### 6.8 Streams as a lightweight queue

**Problem:** you want a durable, ordered log of events with multiple consumers that
each track their own position and can retry failures. **Solution:** Redis **Streams**
(`XADD` to append, consumer groups via `XREADGROUP` + `XACK`). Unlike Pub/Sub,
entries persist and unacknowledged work can be re-delivered.

```ts
// Producer:
await redis.xadd('events:billing', '*', 'type', 'invoice.created', 'id', '42');
// Consumer group (create once): XGROUP CREATE events:billing workers $ MKSTREAM
const msgs = await redis.xreadgroup('GROUP', 'workers', 'pod-A', 'COUNT', 10, 'STREAMS', 'events:billing', '>');
// ...process, then acknowledge so it isn't redelivered:
await redis.xack('events:billing', 'workers', messageId);
```

For *application* job queues (retries, delayed jobs, concurrency, dashboards), you
usually don't hand-roll Streams — you use **BullMQ**, which is built on Redis and
gives you all of that. See **[06-bullmq.md](./06-bullmq.md)**.

### 6.9 The two things people get wrong: invalidation & stampede

- **Invalidation** (covered in 6.3) — a cache is only as trustworthy as your
  discipline in clearing it. Every write path that changes cached data must also
  clear/refresh the cache.

- **Cache stampede** (a.k.a. thundering herd). When a hot key expires, *many*
  concurrent requests all miss at once and all hammer the source of truth
  simultaneously — sometimes crushing it. Mitigations:
  1. **Single-flight / request coalescing:** ensure only *one* refill runs per key at
     a time; others wait for its result. (Our accounts service already does exactly
     this for its per-process cache via an "in-flight fetches" map — see §7. That
     idea carries directly to Redis with a short lock.)
  2. **Slightly randomized (jittered) TTLs** so a batch of keys created together
     don't all expire in the same instant.
  3. **Serve-stale-while-refreshing:** keep a `staleUntil` window and return the old
     value while one worker refreshes in the background. (Our accounts cache also
     already implements a stale-fallback — again, §7.)

---

## 7. How Redis would slot into THIS repo (honest — not used today)

**Reminder, in bold, because it matters:** this repository does **not** use Redis.
There is no Redis dependency, no Redis service, no connection code. This section is a
*proposal* grounded in a real gap, not a description of current behaviour.

### 7.1 The exact gap

Look at `apps/api/src/modules/accounts/accounts.service.ts`. Around **line 1757** sits
this comment, verbatim:

```ts
// TODO we need to use different approch to cache account balances as this in-memory
// cache will not be shared across multiple instances of the API server, leading to
// cache misses and redundant calls to Formance. Consider using a distributed cache
// like Redis or Memcached for better scalability and performance.
private async getBalanceLookupWithCache(
  ledgerName: string,
  addresses: string[]
): Promise<Map<string, AccountBalance>> { ... }
```

The method underneath is a genuinely thoughtful **per-process** cache. It already
implements, in local memory, three of the patterns from §6:

- **cache-aside with TTL** — entries carry an `expiresAt`; fresh ones are served,
  stale ones trigger a refetch;
- **single-flight / stampede protection** — an `inFlightBalanceFetches` map ensures
  concurrent requests for the same batch share *one* Formance call rather than
  dog-piling;
- **serve-stale-on-failure** — a `staleUntil` window lets it fall back to a slightly
  old balance if Formance errors, instead of failing the request.

That is good engineering — but it all lives in **one Node process's `Map`**. As §1
showed, our `api` runs on 2+ pods, so each pod keeps its own private copy: warm on one
pod, cold on another, and — worse for a ledger — **invalidation can't propagate**. If
pod A learns (via a new posting) that an account's balance changed and clears its
entry, pod B has no way to know, and keeps serving the stale number until *its own*
TTL lapses. The very sibling method `clearAccountTreeCacheForOrg` (a few lines up)
loops over a local `Map`'s keys to invalidate — an operation that, likewise, only
touches the pod it runs on.

### 7.2 What the Redis version looks like

Move the shared surface out of the process and onto the whiteboard. Concretely:

- **Cache keys** already exist in the code: `getBalanceCacheKey(ledgerName, address)`
  returns `` `${ledgerName}::${address}` ``. That is a perfectly good Redis key.
  Prefix it, e.g. `bal:{ledgerName}::{address}`.
- **Read path (cache-aside):** `MGET` the batch of balance keys from Redis; whatever
  misses, fetch from Formance (reusing the existing `fetchAndCacheBalances` logic),
  then write back with a short TTL. The batch is a natural `MGET` / pipeline.
- **Single-flight across pods:** replace the in-process `inFlightBalanceFetches` map
  with a short Redis lock (§6.4) per batch key, so only one pod calls Formance for a
  given cold batch — cross-pod stampede protection, which the current design can't
  give you.
- **Invalidation that actually reaches every pod:** on any new posting for an org, the
  write path deletes the affected `bal:*` keys in Redis (and/or publishes to a
  `cache:invalidate` channel per §6.7). Because every pod reads the *same* Redis, the
  next read on *any* pod gets the fresh number. This is the property the per-process
  cache fundamentally cannot have.

A sketch (illustrative — not wired into the repo):

```ts
private async getBalanceLookupViaRedis(
  ledgerName: string,
  addresses: string[],
): Promise<Map<string, AccountBalance>> {
  const unique = Array.from(new Set(addresses));
  const keys = unique.map((a) => `bal:${this.getBalanceCacheKey(ledgerName, a)}`);

  const cached = await this.redis.mget(keys);          // one shared round-trip
  const lookup = new Map<string, AccountBalance>();
  const misses: string[] = [];

  unique.forEach((address, i) => {
    const raw = cached[i];
    if (raw) lookup.set(address, JSON.parse(raw) as AccountBalance);
    else misses.push(address);
  });

  if (misses.length > 0) {
    const fetched = await this.fetchAndCacheBalances(ledgerName, misses, Date.now());
    const pipeline = this.redis.pipeline();
    for (const [address, balance] of fetched.entries()) {
      lookup.set(address, balance);
      pipeline.set(`bal:${this.getBalanceCacheKey(ledgerName, address)}`,
        JSON.stringify(balance), 'EX', BALANCE_CACHE_TTL_SECONDS);
    }
    await pipeline.exec();                              // best-effort write-back
  }
  return lookup;
}
```

### 7.3 Correctness first — this is a ledger

For a money system, a *stale balance is worse than a slow one*. So the Redis version
must lean on invalidation and short TTLs, and — critically — **the balance shown from
cache is never authoritative.** Any operation that *acts* on a balance (posting,
settlement) must read the truth from Formance/Postgres inside its transaction, not
from Redis. Redis accelerates *display and read-heavy lookups*; it must never become
the number we debit against. This is §4.6 and §8's golden rule applied to our exact
domain.

### 7.4 How we'd actually run it here (GCP/GKE)

Because our stack is **GKE Autopilot on GCP** (see the
**[deployment lifecycle guide](../deployment-lifecycle-guide.md)** and
**[08-gcp.md](./08-gcp.md)**), the obvious managed option is **Google Cloud
Memorystore for Redis** — a fully-managed Redis that Google runs, patches, and (on the
Standard tier) fails over automatically. We would:

- Provision a Memorystore instance in the **same VPC/region** as the cluster, reached
  over its **private IP in-cluster** — never a public endpoint (§8 security).
- Store its connection URL/password in **GCP Secret Manager**, and deliver it into the
  `api` pods the same way every other secret already travels: the **External Secrets
  Operator (ESO)** syncs it from Secret Manager into the `api-secrets` Kubernetes
  secret, which is mounted as an env var (`REDIS_URL`). That mechanism is exactly what
  the deployment guide describes for our existing secrets — no new pattern needed.
- Turn on **in-transit encryption (TLS)** and **AUTH** on the instance.

There is a second reason Redis is likely coming to this repo regardless of the balance
cache: **BullMQ, our job-queue library, is built on Redis** — Redis *is* its required
backend. The moment we add background jobs via BullMQ, we stand up a Redis (Memorystore)
anyway, and the balance cache can ride the same instance (logically separated by key
prefix, or a separate logical DB). See **[06-bullmq.md](./06-bullmq.md)**.

**Bottom line for §7:** the gap is real and documented in our own code; the fix is a
textbook Redis cache-aside with cross-pod invalidation; the managed home is
Memorystore reached privately with secrets via ESO; and the non-negotiable constraint
is that Redis stays an *accelerator in front of* the ledger, never the ledger. None of
this is built today.

---

## 8. Production concerns

Running Redis for real means owning these decisions:

- **Sizing memory.** Estimate: (number of keys × average value size) + overhead, then
  add headroom (Redis needs spare RAM for replication buffers, fork-on-save copy, and
  fragmentation — plan for ~1.5–2× your data at peak). Undersizing leads straight to
  the next bullet.
- **Eviction vs OOM.** Decide *on purpose* (see §4.2). A **cache** should run
  `allkeys-lru`/`-lfu` so it forgets cold data gracefully when full. A store you must
  not silently drop should run `noeviction` — but then you must monitor memory and
  scale *before* it fills, because at the cap **writes start erroring**.
- **Persistence choice.** Cache-only? You may not need persistence at all (a cold
  restart just re-warms). Queue/session data? Enable **AOF `everysec`** so a restart
  doesn't lose everything. Remember: even then, ~1s of writes can be lost on a crash
  — acceptable for caches/queues, never for ledger truth.
- **High availability.** Don't run a lone instance in production. Use **Memorystore
  Standard tier** (managed replica + automatic failover), or self-managed
  **Sentinel** (failover for a single dataset), or **Cluster** (only when data
  outgrows one node). Ensure clients reconnect cleanly on failover (ioredis does).
- **Security.** Require **AUTH** (or ACL users), enable **TLS** across untrusted
  boundaries, and **never expose Redis publicly** — private VPC/subnet, firewall to
  only the app. Rotate the password via Secret Manager. The classic breach is an
  open, unauthenticated Redis.
- **Cache invalidation correctness — the hard part.** For anything derived from the
  source of truth, every write path must invalidate/refresh the cache, in the right
  order (source first, cache second), with awareness of the read-during-write race.
  For a **ledger** this is the whole ballgame: prefer short TTLs, explicit
  invalidation on every posting, and *never trusting a cached balance for a
  money-moving decision.*
- **Avoid hot keys & big keys.** A single key hammered by every request (a "hot key")
  bottlenecks on the one thread; a single enormous key (a "big key") makes every
  operation on it slow and blocks the server. Shard hot keys; cap collection sizes;
  watch for keys that grow without bound.
- **Monitoring.** Track, at minimum: **cache hit rate** (`keyspace_hits` vs
  `keyspace_misses`), **memory used vs maxmemory** and **evicted_keys**, **command
  latency** and **slowlog**, **connected clients**, and **replication lag**. Alert on
  rising evictions, falling hit rate, and latency spikes.
- **The golden rule.** **Never store the only copy of important data in a cache.**
  Redis is fast and mostly-durable; treat every byte in it as *reconstructable* from a
  durable system. If you can't afford to lose it, it doesn't live *only* in Redis.

---

## 9. Operations & debugging playbooks

### 9.1 redis-cli cheat sheet

```bash
# Health & overview
redis-cli -a "$PW" PING                 # PONG = alive
redis-cli -a "$PW" INFO                  # everything; pipe to grep a section:
redis-cli -a "$PW" INFO memory | grep -E 'used_memory_human|maxmemory_human'
redis-cli -a "$PW" INFO stats  | grep -E 'keyspace_hits|keyspace_misses|evicted_keys'
redis-cli -a "$PW" INFO replication      # role: master/slave, connected replicas, lag
redis-cli -a "$PW" DBSIZE                 # number of keys

# Watch traffic LIVE (dev only — high overhead, never leave running in prod)
redis-cli -a "$PW" MONITOR                # streams every command as it runs

# Find slow commands (they're logged automatically past a threshold)
redis-cli -a "$PW" SLOWLOG GET 10         # last 10 slow entries
redis-cli -a "$PW" SLOWLOG RESET

# Inspect a specific key
redis-cli -a "$PW" TYPE some:key
redis-cli -a "$PW" TTL  some:key          # -1 no TTL, -2 gone
redis-cli -a "$PW" MEMORY USAGE some:key  # bytes (find your big keys)

# Iterate keys SAFELY — SCAN, never KEYS (see §10)
redis-cli -a "$PW" --scan --pattern 'bal:*' | head
```

### 9.2 "The cache isn't shared across pods"

*Symptom:* hit rate is low; each pod re-fetches the same data; behaviour differs
depending on which pod served the request. *Cause:* a per-process cache (a `Map`) —
**exactly our current accounts situation (§7)**, not a Redis misconfiguration.
*Diagnosis:* if there's no shared Redis and you're on multiple pods, this is expected.
*Fix:* move the cache to a shared Redis (cache-aside, §6.2) so all pods read one
surface. If you *do* have Redis and it's still not shared, confirm every pod points
at the **same host/DB** (`INFO server`, check `REDIS_URL`) and not a local sidecar
each.

### 9.3 "Memory full / evictions climbing"

*Symptom:* `evicted_keys` rising, or (on `noeviction`) writes failing with OOM errors;
latency creeping up. *Diagnose:*

```bash
redis-cli -a "$PW" INFO memory | grep -E 'used_memory_human|maxmemory_human|mem_fragmentation_ratio'
redis-cli -a "$PW" INFO stats  | grep evicted_keys
redis-cli -a "$PW" --bigkeys           # sample the biggest keys per type
```

*Fixes:* raise `maxmemory` / scale the instance; confirm the **eviction policy** suits
your use (cache → `allkeys-lru`); find and fix **missing TTLs** (keys that should
expire but don't — §10) and **big keys**; check `mem_fragmentation_ratio` (well above
1.0 for a long time → consider `activedefrag` or a restart during a window).

### 9.4 "Stale data / users see old values"

*Cause:* a write path changed the source of truth but didn't invalidate the cache, or
did so in the wrong order, or a stampede re-populated the old value. *Diagnose:* find
the key (`GET`/`HGETALL`), check its `TTL`. *Fix:* ensure every write invalidates
(source-of-truth first, then `DEL`/refresh — §6.3); shorten the TTL for sensitive
data; for ledger balances, don't trust cache for decisions (§7.3).

### 9.5 "Latency spike from a big key / slow command"

*Cause:* an O(N) command over a large structure blocking the single thread (§4.1).
*Diagnose:* `SLOWLOG GET`, then `MEMORY USAGE <key>` / `--bigkeys` on the culprits.
*Fix:* stop using `KEYS`/`LRANGE 0 -1`/`SMEMBERS`/`HGETALL` on big keys on the hot
path; page through with `SCAN`/`HSCAN`/`SSCAN`/`ZSCAN`; split big keys; cap collection
growth.

### 9.6 Flushing safely

```bash
# NUCLEAR — deletes ALL keys. Never on a shared/prod instance without certainty.
redis-cli -a "$PW" FLUSHDB    # current logical DB
redis-cli -a "$PW" FLUSHALL   # every DB on the instance

# Safer targeted purge: delete only matching keys, without blocking on KEYS.
redis-cli -a "$PW" --scan --pattern 'bal:*' | \
  xargs -r -n 100 redis-cli -a "$PW" UNLINK   # UNLINK frees memory in the background
```

Prefer **`UNLINK`** over `DEL` for large keys — it reclaims memory on a background
thread instead of blocking. Prefer targeted `SCAN`-driven deletes over `FLUSH*` on
anything shared.

---

## 10. Gotchas & hard-won lessons

- **`KEYS` blocks the whole server.** `KEYS *` walks the *entire* keyspace on the
  single thread, freezing every other client until it finishes. It's a foot-gun on any
  non-trivial dataset. **Always use `SCAN`** (cursor-based, chunked). This is the most
  common way people accidentally cause a "Redis outage."
- **Forgotten TTL → slow memory leak.** A key written without an expiry that nobody
  ever deletes lives forever. Do this across millions of keys and you fill memory and
  trigger evictions or OOM. Give ephemeral data a TTL *at write time*, and audit for
  keys that lack one when they shouldn't.
- **Cache stampede (thundering herd).** A hot key expires and a flood of concurrent
  misses stampede the source of truth at once. Use single-flight/locking, jittered
  TTLs, and serve-stale-while-refreshing (§6.9). Our accounts service already does the
  first and third in-process; carry them to Redis.
- **Using Redis as the source of truth.** In-memory + only-mostly-durable. Losing a
  second of writes is fine for a cache and catastrophic for a ledger. **The durable
  system owns the truth; Redis owns speed.** (§4.6, §8.)
- **Lock correctness / the Redlock debate.** A Redis lock is great for *deduplication
  and optimization* but is not a hard mutex under pauses/failover. Make the protected
  operation idempotent; don't make money-safety depend on it. Know that even Redlock is
  contested. (§6.4.)
- **Single-threaded O(N) traps.** Beyond `KEYS`: `SMEMBERS`, `HGETALL`, `LRANGE 0 -1`,
  `ZRANGE` over huge sets, and un-bounded Lua scripts all block everyone while they
  run. Keep keys and scripts small; page with `*SCAN`.
- **Not durable by default.** Out of the box you can lose recent writes on a crash;
  even AOF `everysec` risks ~1 second. Choose persistence deliberately and never assume
  "it was in Redis" means "it's safe forever."
- **Multiple clients ≠ shared state within a process.** And multiple *pods* with
  per-process caches are the opposite of shared — the exact §1/§7 lesson. If it must be
  shared, it must live in the shared store.
- **`MULTI/EXEC` is not SQL transactions.** No rollback on logical errors — it's an
  uninterrupted batch. For real read-decide-write atomicity, reach for Lua.

---

## 11. Glossary + further reading

### Glossary

| Term | Meaning |
|---|---|
| **AOF** | Append Only File — durability via a replayed write-log. |
| **RDB** | Point-in-time binary snapshot of the dataset. |
| **TTL** | Time To Live — a key's self-destruct timer. |
| **Eviction** | Deleting keys to stay under `maxmemory`. |
| **LRU / LFU** | Least Recently / Frequently Used — eviction heuristics. |
| **Cache-aside** | App checks cache → on miss, fetch source → write back with TTL. |
| **Invalidation** | Deleting/refreshing a cache entry when the source changes. |
| **Stampede / thundering herd** | Many simultaneous misses hammering the source. |
| **Single-flight** | Coalescing concurrent misses into one refill. |
| **Pipelining** | Sending many commands without waiting per-reply. |
| **MULTI/EXEC** | An uninterrupted batch of queued commands. |
| **Lua / EVAL** | Server-side script run atomically — read-decide-write in one step. |
| **Pub/Sub** | At-most-once channel fan-out to live subscribers. |
| **Stream** | Durable, ordered append-only log with consumer groups. |
| **Sentinel** | Watchdogs that auto-promote a replica on primary failure. |
| **Cluster** | Sharding across nodes via 16384 hash slots. |
| **Hash slot** | The bucket (0–16383) a key maps to in Cluster. |
| **Hot key / big key** | A single over-hit / over-large key that bottlenecks the one thread. |
| **Memorystore** | Google Cloud's managed Redis service. |
| **Redlock** | Multi-node Redis distributed-lock algorithm (and its debates). |

### Further reading

- **Redis official docs** — https://redis.io/docs/latest/
- **Redis data types** — https://redis.io/docs/latest/develop/data-types/
- **Key expiration / TTL** — https://redis.io/docs/latest/commands/expire/
- **Eviction policies** — https://redis.io/docs/latest/develop/reference/eviction/
- **Persistence (RDB & AOF)** — https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- **Transactions (MULTI/EXEC)** — https://redis.io/docs/latest/develop/interact/transactions/
- **Lua scripting (EVAL)** — https://redis.io/docs/latest/develop/interact/programmability/eval-intro/
- **Distributed locks & Redlock** — https://redis.io/docs/latest/develop/use/patterns/distributed-locks/
  (and Martin Kleppmann's well-known critique: https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- **Streams** — https://redis.io/docs/latest/develop/data-types/streams/
- **SCAN vs KEYS** — https://redis.io/docs/latest/commands/scan/
- **ioredis** — https://github.com/redis/ioredis
- **node-redis** — https://github.com/redis/node-redis
- **Google Cloud Memorystore for Redis** — https://cloud.google.com/memorystore/docs/redis

### Related chapters in this handbook

- **[06-bullmq.md](./06-bullmq.md)** — BullMQ job queues, which *require* Redis as
  their backend.
- **[08-gcp.md](./08-gcp.md)** — our GCP/GKE stack, where a managed Redis
  (Memorystore) would live.
- **[../deployment-lifecycle-guide.md](../deployment-lifecycle-guide.md)** — how
  secrets (like a `REDIS_URL`) travel from GCP Secret Manager into pods via the
  External Secrets Operator.

---

*Final honesty check, restated: nothing in this chapter describes Redis actually
running in the GIBP ledger repo today. It does not. The one real artifact is the TODO
at `apps/api/src/modules/accounts/accounts.service.ts:1757`. Everything in §7 is a
grounded proposal for closing that gap, not a report of existing behaviour.*
