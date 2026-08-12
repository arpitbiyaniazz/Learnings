# Chapter 6 — BullMQ: Background Jobs & Queues on Redis

**What this chapter is.** A from-zero teaching book on **BullMQ**, the Redis-backed
background-job and message-queue library for Node.js. By the end you will be able to
explain *why* job queues exist, reason about how BullMQ works under the hood, stand one
up from scratch in plain TypeScript, wire it into a NestJS service, and — most
importantly — know *when to reach for BullMQ versus Kafka*. This is written for a junior
developer who has never built a job queue before.

> **Status in this repo: NOT used.**
> This project has **no BullMQ and no Redis**. It does all of its asynchronous and
> background work through **Kafka** (see `03-kafka.md`) with a dedicated `kafka-worker`
> service, and it has no Redis instance provisioned yet (see `05-redis.md`). Nothing in
> this chapter describes code that runs in production today. Section 7 is the honest part:
> it explains where BullMQ *could* slot in, and gives you a fair **BullMQ-vs-Kafka**
> decision guide so you pick the right tool.

**How to read this.**
- If you have never built a background job, read **§1–§5** front to back. They build the
  mental model and get you a working queue on your laptop.
- If you know queues already and just want the NestJS pattern, jump to **§6**.
- If you are deciding *"should this new feature use BullMQ or Kafka?"*, read **§7** — that
  is the decision this chapter most wants you to get right.
- **§8–§10** are the "don't get paged at 3am" material: production concerns, debugging
  playbooks, and hard-won gotchas.
- **§11** is a glossary and links.

Terminology is introduced slowly and every hard concept gets an analogy that is mapped
back to the real mechanism. Nothing is assumed.

---

## 1. The problem: some work can't happen inside the request

Picture the simplest possible web server. A user clicks a button, the browser sends an
HTTP request, your server does the work, and sends a response. When the work is fast —
read a row, return some JSON — this is perfect. The user waits a few milliseconds and
gets their answer.

Now the work gets slow, or delayed, or flaky. Suddenly "do it inline, inside the request"
falls apart. Here are the four situations, each of which is a real problem you *will* hit:

**1. The work is slow.**
Generating a PDF report, resizing an uploaded image, running a big export — these take
seconds, not milliseconds. If you do them inside the HTTP handler, the user's browser
sits on a spinner. Worse, your web server process is now *blocked* holding that request
open, so it can serve fewer other users. In this very repo, the report **PDF export**
launches a headless Chromium browser and renders HTML to a PDF *inside the HTTP request*,
with a **120-second timeout** (`apps/api/src/modules/reports/report-pdf.constants.ts`,
`REPORT_PDF_DEFAULT_TIMEOUT_MS = 120_000`). Two minutes is an eternity to hold a request
open. (We come back to this in §7 — it is the poster child for "should have been a job.")

**2. The work must happen later.**
"Send this reminder email in 10 minutes." "Cancel this unpaid order in 24 hours." You
cannot do that inside a request that finished a second ago. You need something that
remembers to run later.

**3. The work must happen on a schedule.**
"Every night at 03:00, run the reconciliation job." "Every 5 minutes, poll the exchange
rate feed." No user click triggers these. Something must fire them on a clock.

**4. The work can fail and must be retried.**
Calling a third-party API — a payment gateway, an email provider, an FX-rate service — is
*flaky*. The network blips, the vendor has a bad minute, you get a `503`. Doing it inline
means the user's action fails because a vendor sneezed. What you actually want is: "try it;
if it fails, wait a bit and try again; give up only after several attempts."

The naive fixes all have a fatal flaw:

| Naive fix | Why it breaks |
| --- | --- |
| Do the slow work inline | User waits; web process is blocked; a crash loses the work |
| `setTimeout(() => sendEmail(), 600_000)` | Lives in one process's memory. **Deploy, crash, or restart → the timer is gone and the email never sends.** No retries. No visibility. |
| A `cron` entry on one server | Runs on exactly one box; if you scale to 3 pods it either runs 3× or 0×; no retries; no record of what ran |
| A `while(true)` loop polling the DB | You just hand-rolled a worse queue, with race conditions when you scale out |

The word that keeps appearing is **durable**. The moment a promise like "I'll do this
later / retry this / run this nightly" has to survive a process restart, a deploy, or a
crash, you cannot keep it in one process's memory. You need it written down somewhere
outside the process, and you need something that reliably picks it back up.

That "written down somewhere durable, with retries, delays, scheduling, and controlled
concurrency" is exactly what a **job queue** is. **BullMQ** is one of the most popular job
queues for Node.js, and it writes that to-do list into **Redis** (a fast in-memory data
store — see `05-redis.md`).

> **First principle to hold onto:** a job queue exists to move slow / delayed / scheduled
> / retryable work *out of the request path* and into a durable list that a separate pool
> of workers processes reliably.

---

## 2. Mental model: a shared to-do list with a team of workers

Here is the whole thing in one picture. Forget the code for a moment.

Imagine a **kitchen**. Orders (tickets) come in from the front of house and get clipped
onto a rail. Cooks (the workers) grab the next ticket off the rail, cook the dish, and
either put the finished plate on the pass or — if they burn it — put the ticket in a
"remake" pile to try again. A ticket that keeps failing eventually goes into a "we
couldn't make this" pile that the manager reviews.

```
  Front of house              THE RAIL (Redis)                 The line cooks
  (producers)                                                  (workers)

  POST /reports  ──add──►  ┌──────────────────────────┐   ──pull──►  Cook A
  "make PDF #42"           │  waiting: [#42, #43, #44] │   ──pull──►  Cook B
                           │  delayed: [#50 @ 10:00]   │   ──pull──►  Cook C
  cron  ─────────add──►    │  completed: [...]         │
  "nightly recon"          │  failed:    [...]         │
                           └──────────────────────────┘
```

Mapping the analogy back to BullMQ's real terms:

| Kitchen analogy | BullMQ term | What it really is |
| --- | --- | --- |
| The order ticket | **Job** | A unit of work + its data (e.g. `{ reportId: 42 }`) |
| The rail everything hangs on | **Queue** | A named list stored in **Redis** |
| Clipping a ticket to the rail | **Producer** / `queue.add(...)` | Any code that enqueues work |
| A cook | **Worker** | A process that pulls jobs and runs your handler |
| How many cooks work at once | **Concurrency** | How many jobs one worker runs in parallel |
| The "remake" pile | **Retry** (attempts + backoff) | Automatic re-runs on failure |
| The "we couldn't make this" pile | **Failed** jobs (a de-facto dead-letter list) | Jobs that exhausted their retries |
| "Serve this at 8pm, not now" | **Delayed** job | A job that becomes runnable at a future time |
| "Every night at 3am" | **Repeatable** / **scheduled** job | A job that re-adds itself on a cron/interval |

The two crucial properties this buys you:

1. **The rail lives in Redis, not in a Node process.** If every cook goes home (all
   workers restart / redeploy), the tickets are still clipped to the rail. When cooks come
   back, they pick up where they left off. That is *durability*.

2. **Producers and workers are decoupled.** The front of house doesn't wait for the dish
   to be cooked — it clips the ticket and moves on (your HTTP request returns immediately).
   Cooks can be scaled independently of waiters (you run more worker pods without touching
   the web pods).

Keep this kitchen in your head for the rest of the chapter. Every BullMQ feature is just a
refinement of "tickets on a rail, cooks pulling them."

---

## 3. Core concepts & vocabulary

Before we write code, here is the vocabulary. Read it once now; it will click harder after
§4 and §5, and it's here to refer back to.

| Term | One-line definition | Kitchen analogy |
| --- | --- | --- |
| **Queue** | A named channel of jobs, stored in Redis. You create it to *add* jobs. | The rail |
| **Job** | One unit of work: a name, a JSON `data` payload, and options. | A ticket |
| **Producer** | Any code that calls `queue.add(...)`. | A waiter clipping a ticket |
| **Worker** | A long-running process that pulls jobs and runs your handler function. | A cook |
| **Processor / handler** | The function that actually does the work for a job. | The act of cooking |
| **Job lifecycle** | The states a job moves through (below). | Ticket's journey |
| **`waiting`** | Enqueued, not yet picked up. | On the rail, untouched |
| **`active`** | A worker is currently running it. | A cook is on it |
| **`completed`** | Finished successfully. | Plated |
| **`failed`** | The handler threw and retries are exhausted. | In the "couldn't make it" pile |
| **`delayed`** | Scheduled to become `waiting` at a future time. | "Serve at 8pm" |
| **Attempts** | How many times BullMQ will try a job before marking it `failed`. | Remakes allowed |
| **Backoff** | The wait between retries (fixed or exponential). | "Take a breath before remaking" |
| **Delayed job** | A job with `delay`/timestamp so it runs later. | Timed ticket |
| **Repeatable / scheduled job** | A job that re-enqueues on a cron pattern or interval. | Standing nightly prep |
| **Rate limiting** | A ceiling on how many jobs run per time window. | "Max 100 dishes/min" |
| **Concurrency** | How many jobs a single worker runs in parallel. | Cook juggling N pans |
| **Priority** | Lets some jobs jump ahead of others in the waiting list. | VIP tickets first |
| **Flows (parent/child)** | Jobs that depend on other jobs; parent runs after children. | A dish that waits on its sides |
| **Events** | Notifications a Worker emits locally (`completed`, `failed`, `progress`). | Cook shouting "up!" |
| **`QueueEvents`** | A separate listener that reports queue-wide events across all workers. | The expediter watching the whole pass |
| **Job `data`** | The input payload you pass in (JSON, stored in Redis). | What's written on the ticket |
| **Job return value** | Whatever your handler returns; stored on the completed job. | Notes stapled to the finished plate |
| **`jobId` / idempotency** | A custom stable id; adding a job with an existing id is de-duplicated. | "We already have this ticket, don't clip a second" |
| **Stalled job** | An `active` job whose worker died/hung without finishing; auto-recovered. | Cook collapsed mid-dish; ticket goes back |
| **Built on Redis** | Every one of the above is a Redis data structure manipulated atomically. | The rail is a physical object everyone shares |

The **job lifecycle** is worth drawing, because almost every operational question ("why is
my job stuck?") is really "which state is it in and why isn't it moving?":

```
                    add()                 worker picks it up
   (nothing)  ───────────────►  waiting  ───────────────────►  active
                    │                                            │
       add({delay}) │                                     ┌──────┴───────┐
                    ▼                                  success          throws
                 delayed ──(time is up)──► waiting        │                │
                                                          ▼                ▼
                                                     completed    attempts left?
                                                                   │        │
                                                              yes  │        │  no
                                                    (wait backoff) │        │
                                                        delayed ◄──┘        ▼
                                                                         failed
```

Read that as: a job is born `waiting` (or `delayed` if you asked for a delay). A worker
moves it to `active`. If the handler resolves, it's `completed`. If the handler throws and
there are attempts left, it goes back to `delayed` for the backoff wait, then `waiting`
again. When attempts run out, it lands in `failed` — which is your dead-letter pile.

---

## 4. How it actually works (the deep version)

You do not *need* this section to use BullMQ, but you *do* need it the first time
something goes wrong in production. Understanding the machinery is what separates "I use a
library" from "I can debug it at 3am."

### 4.1 Everything is a Redis data structure

BullMQ does not have a server of its own. **BullMQ is a client library**; the "brain" is
Redis. A queue named `reports` is really a *family of Redis keys*, all prefixed
`bull:reports:`:

| Redis key (conceptually) | Redis type | Holds |
| --- | --- | --- |
| `bull:reports:wait` | List | IDs of `waiting` jobs (FIFO) |
| `bull:reports:active` | List | IDs of jobs a worker has picked up |
| `bull:reports:delayed` | **Sorted set** | Job IDs scored by their "run at" timestamp |
| `bull:reports:completed` | Sorted set | Recently completed job IDs |
| `bull:reports:failed` | Sorted set | Failed job IDs |
| `bull:reports:<jobId>` | Hash | One job's data, options, attempts, return value |
| `bull:reports:events` | Stream | The event log (`QueueEvents` reads this) |

That is the whole trick: a job "moving from `waiting` to `active`" is literally an atomic
Redis operation that pops an ID off the `wait` list and pushes it onto the `active` list.

### 4.2 Atomic moves via Lua scripts — why races don't happen

Here is the scary question: if you run **three** worker pods, and there is **one** job
waiting, how do you stop all three from grabbing it and doing the work three times?

The answer is that BullMQ never does "read the list, then pop the item" as two separate
steps (that would be a classic race). Instead every state transition is a **Lua script**
that Redis executes **atomically** — Redis runs the whole script start-to-finish without
letting any other command interleave. So "pop from `wait` AND push to `active` AND set the
lock" happens as one indivisible operation. Only one worker can win the pop. The others get
nothing and move on.

> **Analogy:** the rail has a rule enforced by physics — a ticket can only be in one cook's
> hand at a time. Two cooks reaching for the same ticket is impossible because grabbing it
> is a single physical act, not "look, then grab." Lua scripts are BullMQ's "single
> physical act."

This is why BullMQ is safe to scale horizontally without you writing any locking code.

### 4.3 Locks and the stalled-job mechanism

When a worker takes a job, it also sets a **lock** on that job (a Redis key with a TTL,
say 30 seconds — the `lockDuration`). While the worker processes the job it periodically
**renews** the lock (like saying "still cooking, don't reassign my ticket").

Now suppose the worker's pod is killed mid-job (GKE evicts it, OOM, `kubectl delete`). The
worker stops renewing the lock. After the lock TTL expires, the job is considered
**stalled**. A background check (`stalledInterval`, default ~30s) run by the other workers
notices the `active` job whose lock is dead and **moves it back to `waiting`** so another
worker can retry it — up to `maxStalledCount` times, after which it is failed to avoid an
infinite loop.

This is the mechanism that makes "a worker died mid-job" recoverable instead of a silent
lost task. It also has a sharp edge you must respect (§10): if your handler blocks the Node
event loop for longer than the lock duration — a giant synchronous loop, a huge JSON
parse — it **can't renew its own lock**, so BullMQ thinks it stalled and *hands the same
job to a second worker while the first is still running it*. That is a duplicate. The
lesson: keep handlers async and don't block the event loop.

### 4.4 Retries with backoff

When your handler throws, BullMQ looks at the job's `attempts` option. If attempts remain,
it schedules a retry after a **backoff** delay:

- **Fixed backoff:** wait the same amount every time (e.g. 5s, 5s, 5s).
- **Exponential backoff:** wait `delay * 2^(attemptsMade)` (e.g. 1s, 2s, 4s, 8s…). This is
  almost always what you want for a flaky external call — it backs off politely instead of
  hammering a struggling vendor.

Mechanically, a retry is just a **delayed** re-enqueue: the job goes back into the
`delayed` sorted set scored with "now + backoff," and re-enters `waiting` when its time
comes. When `attemptsMade` reaches `attempts`, the job goes to `failed` for good.

You can also add **jitter** (randomness) to backoff so that a thousand jobs that all failed
at the same instant don't all retry at the exact same instant (a "thundering herd").

### 4.5 Delayed jobs = a sorted set + a clock

A delayed job isn't a `setTimeout` living in a process (that would die on restart). It's an
entry in the `delayed` **sorted set**, scored by the Unix timestamp at which it should run.
Workers (and a small internal scheduler) periodically ask Redis "give me every delayed job
whose score is ≤ now" and atomically move those into `waiting`. Because the schedule lives
in Redis, a delayed job survives every worker restart. That is the entire reason you can
trust "send this in 10 minutes" — the promise is in Redis, not in RAM.

### 4.6 Repeatable / scheduled jobs

A repeatable job (cron: `"0 3 * * *"` = 3am nightly) works by BullMQ keeping a small
"scheduler" record and, each time an occurrence fires, computing the *next* occurrence and
enqueueing a fresh delayed job for it. So a nightly job is really "an endless chain of
delayed jobs, each one queuing the next." In modern BullMQ this is exposed through the
**Job Scheduler** API (`upsertJobScheduler`), which supersedes the older `repeat` option
(§5.6). The important property: because the schedule is stored in Redis and keyed by a
stable scheduler id, running three worker pods does **not** give you three nightly runs —
they all share the one scheduler record.

### 4.7 Concurrency & rate limiting

- **Concurrency** is per-worker: `new Worker(name, fn, { concurrency: 5 })` means *this
  one worker process* will run up to 5 jobs at once (great when jobs are I/O-bound — waiting
  on the network — because Node can happily juggle many awaits). Total throughput ≈
  `concurrency × number of worker processes`.
- **Rate limiting** is a ceiling across the queue: `{ limiter: { max: 100, duration:
  60_000 } }` = "no more than 100 jobs start per 60 seconds," no matter how many workers you
  run. This is how you respect a third party's "1000 requests/minute" limit without
  hand-rolling a token bucket.

### 4.8 At-least-once delivery → handlers must be idempotent

This is the single most important sentence in the chapter:

> **BullMQ gives you *at-least-once* execution, not *exactly-once*. Your handler will
> occasionally run twice for the same job. Design every handler so running it twice is
> harmless.**

Why at-least-once? Consider: a worker finishes the *work* (charges the card, sends the
email) and then, in the microsecond before it marks the job `completed` in Redis, its pod
is killed. The job's lock expires, it's seen as stalled, and another worker runs it again —
double charge, double email. BullMQ cannot prevent this, because "did the side effect
happen?" lives in the outside world (the payment provider), not in Redis. No queue on any
platform truly solves this for you; they all push it to you via **idempotency**.

Making a handler idempotent means: running it with the same input twice produces the same
end state as running it once. Techniques:
- Use a **stable `jobId`** (e.g. `report-42`) so duplicate *enqueues* de-dupe automatically.
- Give external calls an **idempotency key** (Stripe, most payment/email APIs support one).
- **Check-before-act:** "is report 42 already generated? then skip." Guard on a DB row /
  unique constraint.

We hammer this again in §10 because it is the mistake juniors make most.

### 4.9 Retention & trimming

Completed and failed jobs don't vanish — they sit in Redis so you can inspect them. That's
useful *and* a memory leak waiting to happen: a queue doing 1M jobs/day will fill Redis
with a million completed-job hashes. So you tell BullMQ to trim them with
`removeOnComplete` / `removeOnFail`, either by **count** (keep the last N) or **age** (keep
for N seconds). Typical: keep completed jobs briefly (or `true` = delete immediately) but
keep *failed* jobs much longer, because failed jobs are the ones you need to inspect.
Redis is memory; unbounded retention is the number-one way people accidentally OOM it (§8,
§10).

---

## 5. Setup from scratch (plain TypeScript)

Time to build one. This section assumes zero prior queue setup and gets you to a
working, verifiable queue on your machine.

### 5.1 Prerequisites

1. **Node.js** (this repo pins Node in `.nvmrc`; BullMQ needs a modern LTS).
2. **A Redis instance.** BullMQ *is* a Redis client — no Redis, no BullMQ. For local dev
   the easiest path is Docker (5.8). For the concepts of Redis itself, see `05-redis.md`.
   BullMQ needs Redis **≥ 6.2** (it uses newer commands).
3. **The library:**

```bash
npm install bullmq
# bullmq bundles ioredis (the Redis client) as a dependency — you don't install it separately
```

### 5.2 A shared connection

Every Queue, Worker, and QueueEvents needs a Redis connection. Centralise it so you don't
repeat the host/port everywhere. **One gotcha up front:** connections used by a *Worker*
must set `maxRetriesPerRequest: null` — BullMQ uses long-lived blocking commands and
ioredis will otherwise abort them. This trips up almost everyone once.

```ts
// src/queue/connection.ts
import { RedisOptions } from 'ioredis';

export const redisConnection: RedisOptions = {
  host: process.env.REDIS_HOST ?? '127.0.0.1',
  port: Number(process.env.REDIS_PORT ?? 6379),
  // Required by BullMQ for the blocking commands Workers use:
  maxRetriesPerRequest: null,
};
```

### 5.3 A minimal Queue (the producer side)

```ts
// src/queue/reports.queue.ts
import { Queue } from 'bullmq';
import { redisConnection } from './connection';

// Give the type of the job DATA so `queue.add` is type-checked.
export interface ReportJobData {
  reportId: number;
  requestedBy: string;
}

// Naming the queue 'reports' fixes the Redis key prefix bull:reports:*
export const reportsQueue = new Queue<ReportJobData>('reports', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2_000 },
    removeOnComplete: { age: 3_600, count: 1_000 }, // keep 1h / last 1000
    removeOnFail: { age: 24 * 3_600 },              // keep failures 24h to inspect
  },
});
```

### 5.4 Adding a job

```ts
// src/produce.ts
import { reportsQueue } from './queue/reports.queue';

async function main() {
  const job = await reportsQueue.add(
    'generate-pdf',                 // job NAME (a label; a queue can hold several names)
    { reportId: 42, requestedBy: 'gibpdev@wisflux.com' }, // job DATA
    { jobId: 'report-42' }          // stable id → adding 'report-42' twice de-dupes
  );
  console.log(`Enqueued job ${job.id}`);
  await reportsQueue.close();
}

main();
```

At this instant, nothing runs the job yet — it's sitting in `bull:reports:wait` in Redis.
That's the point: the producer returns immediately. A worker does the actual work.

### 5.5 A Worker (the consumer side)

```ts
// src/worker.ts
import { Worker, Job } from 'bullmq';
import { redisConnection } from './queue/connection';
import type { ReportJobData } from './queue/reports.queue';

const worker = new Worker<ReportJobData>(
  'reports', // MUST match the queue name
  async (job: Job<ReportJobData>) => {
    // ---- your actual work goes here ----
    console.log(`Processing ${job.id}: report ${job.data.reportId}`);
    await job.updateProgress(10);

    // Simulate slow work (e.g. render a PDF)
    await new Promise((r) => setTimeout(r, 1_000));

    await job.updateProgress(100);
    // Whatever you RETURN is stored as the job's return value.
    return { url: `s3://reports/${job.data.reportId}.pdf` };
  },
  { connection: redisConnection, concurrency: 5 }
);

// Local worker events — useful for logging.
worker.on('completed', (job, result) =>
  console.log(`✅ ${job.id} done →`, result)
);
worker.on('failed', (job, err) =>
  console.error(`❌ ${job?.id} failed: ${err.message}`)
);
```

Run `worker.ts`, then run `produce.ts` in another terminal, and you'll watch the job move
`waiting → active → completed`. **That is a complete queue.** Everything else is options.

### 5.6 Attempts, backoff, delay, and repeatable jobs

**Retryable job that runs 60 seconds from now:**

```ts
await reportsQueue.add(
  'call-flaky-vendor',
  { vendorId: 7 },
  {
    delay: 60_000,                                   // run 1 minute later
    attempts: 5,                                     // try up to 5 times
    backoff: { type: 'exponential', delay: 1_000 },  // 1s, 2s, 4s, 8s…
  }
);
```

**A nightly (cron) job — modern Job Scheduler API:**

```ts
// Idempotent: calling upsertJobScheduler again with the same id just updates it.
await reportsQueue.upsertJobScheduler(
  'nightly-reconciliation',           // stable scheduler id
  { pattern: '0 3 * * *', tz: 'Asia/Tokyo' }, // every day at 03:00 JST
  { name: 'run-reconciliation', data: { scope: 'all' } }
);
```

> The older equivalent was `queue.add('run-reconciliation', data, { repeat: { pattern:
> '0 3 * * *' } })`. You'll see it in older codebases; new code should use
> `upsertJobScheduler`, which is easier to reason about (one record you upsert, rather than
> repeat-keys you have to find and remove).

### 5.7 Graceful shutdown

When your worker process is asked to stop (a deploy, `SIGTERM` from Kubernetes), you must
let in-flight jobs finish and stop pulling new ones — otherwise you kill a job mid-flight
and rely on stalled-recovery (a duplicate risk). `worker.close()` does exactly this: stop
accepting new jobs, wait for active ones to finish, release the connection.

```ts
// src/worker.ts (continued)
async function shutdown() {
  console.log('Shutting down worker gracefully…');
  await worker.close(); // stops pulling new jobs, waits for active ones
  process.exit(0);
}
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

### 5.8 Run Redis + your worker end-to-end (verify it works)

You need a Redis to talk to. The quickest is Docker:

```bash
# Start a throwaway Redis on the standard port
docker run --rm -p 6379:6379 --name bullmq-redis redis:7-alpine
```

Now, in two more terminals:

```bash
# terminal 2 — the worker (leave it running)
npx ts-node src/worker.ts

# terminal 3 — enqueue a job
npx ts-node src/produce.ts
```

**What "working" looks like:** terminal 3 prints `Enqueued job report-42`; terminal 2
prints `Processing … → ✅ report-42 done`. To *see the raw Redis behind it*:

```bash
docker exec -it bullmq-redis redis-cli
> KEYS bull:reports:*          # the queue's Redis keys
> LRANGE bull:reports:wait 0 -1  # job IDs currently waiting
> HGETALL bull:reports:report-42 # the job's stored data/state
```

Seeing `bull:reports:*` keys appear and change is the "aha" moment: BullMQ is *just* Redis
data structures, exactly as §4 described.

---

## 6. Using BullMQ in NestJS

This repo is a **NestJS 11** monorepo (`@nestjs/core@11`, `@nestjs/microservices@11`), so
if BullMQ were ever adopted here, you'd use the official Nest integration
**`@nestjs/bullmq`** rather than raw `new Queue()` calls. It gives you dependency
injection, lifecycle-managed workers, and decorator-based processors that fit Nest's
module system. (Again: *this is illustrative* — none of this exists in the repo today.)

```bash
npm install @nestjs/bullmq bullmq
```

### 6.1 Register the connection and a queue

```ts
// app.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    // One-time root config: the shared Redis connection.
    BullModule.forRoot({
      connection: {
        host: process.env.REDIS_HOST,
        port: Number(process.env.REDIS_PORT ?? 6379),
      },
    }),
    // Declare a queue named 'reports' (repeat per queue).
    BullModule.registerQueue({ name: 'reports' }),
  ],
})
export class AppModule {}
```

### 6.2 Enqueue from a service or controller

Inject the queue with `@InjectQueue` and call `add`. The HTTP handler returns *immediately*
after enqueueing — the caller gets a `202 Accepted`-style "we're working on it," not a
blocked 2-minute request.

```ts
// reports.controller.ts
import { Controller, Post, Param } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Controller('reports')
export class ReportsController {
  constructor(@InjectQueue('reports') private readonly reportsQueue: Queue) {}

  @Post(':id/export')
  async requestExport(@Param('id') id: string) {
    const job = await this.reportsQueue.add(
      'generate-pdf',
      { reportId: Number(id) },
      { jobId: `report-${id}`, attempts: 3, backoff: { type: 'exponential', delay: 2_000 } }
    );
    // Return a handle; the client polls /reports/jobs/:jobId for status.
    return { jobId: job.id, status: 'queued' };
  }
}
```

### 6.3 The processor (the worker, Nest-style)

In Nest you don't `new Worker()`; you declare a `@Processor` class that extends
`WorkerHost`. Nest starts and stops the underlying worker with the app lifecycle
(including graceful shutdown, if you enable Nest shutdown hooks).

```ts
// reports.processor.ts
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { Logger } from '@nestjs/common';

@Processor('reports', { concurrency: 5 }) // 5 jobs in parallel per instance
export class ReportsProcessor extends WorkerHost {
  private readonly logger = new Logger(ReportsProcessor.name);

  async process(job: Job): Promise<{ url: string }> {
    this.logger.log(`Processing ${job.name} #${job.id}`);

    // IDEMPOTENCY GUARD: at-least-once means this may run twice.
    // Check before doing the expensive/irreversible work.
    // if (await this.reports.alreadyExported(job.data.reportId)) {
    //   return this.reports.existingUrl(job.data.reportId);
    // }

    const url = await this.renderPdf(job.data.reportId); // your real work
    return { url };
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job, err: Error) {
    this.logger.error(`Job ${job.id} failed: ${err.message}`);
  }

  private async renderPdf(reportId: number): Promise<string> {
    // e.g. reuse the existing report-pdf-playwright util here
    return `s3://reports/${reportId}.pdf`;
  }
}
```

### 6.4 Separate the web process from the worker process

A key architectural decision — and one this repo *already models* for Kafka. In this repo,
the HTTP API lives in `apps/api` and Kafka consumers live in a **separate deployable**,
`apps/kafka-worker`, with its own `main.ts`, its own Dockerfile, and its own Kubernetes
deployment under `infra/k8s/apps/`. You would do the exact same thing for BullMQ: the API
pods enqueue jobs (import the queue, no processor), and a **separate worker deployment**
imports the `ReportsProcessor` and actually does the work. Why:
- **Isolation:** a runaway PDF render (headless Chromium eating RAM/CPU) can't take down
  your API pods.
- **Independent scaling:** scale worker replicas on queue depth without scaling the web tier.
- **Independent resource profiles:** the worker pod ships Chromium; the API pod may not need
  to.

### 6.5 Observability: QueueEvents and Bull Board

- **`QueueEvents`** is a queue-wide event listener (distinct from a Worker's *local* events).
  It reads the Redis event stream and lets any process observe `completed` / `failed` /
  `progress` / `stalled` across *all* workers — ideal for metrics or pushing progress to a
  websocket.

  ```ts
  import { QueueEvents } from 'bullmq';
  const events = new QueueEvents('reports', { connection: redisConnection });
  events.on('completed', ({ jobId }) => metrics.increment('reports.completed'));
  events.on('failed', ({ jobId, failedReason }) => metrics.increment('reports.failed'));
  ```

- **Bull Board** (`@bull-board/api` + `@bull-board/express` or `@bull-board/nestjs`) is a
  small web dashboard showing every queue: counts of waiting/active/completed/failed, each
  job's data and error, and buttons to retry/remove. It's the first thing you open when
  someone says "the reports aren't generating." Mount it behind auth — it exposes job
  payloads.

**The three rules to carry from §6:** (1) processors **must be idempotent**; (2) set
**sensible `attempts` + backoff** on every job; (3) treat the **`failed` list as your
dead-letter queue** and actually monitor it.

---

## 7. How it would slot into THIS repo — and when to pick BullMQ vs Kafka

This is the honest, repo-grounded section. Read it carefully; it's the one that changes
how you build.

### 7.1 The truthful status

**BullMQ is not used in this project, and there is no Redis provisioned.** All
asynchronous and background work today flows through **Kafka**:
- Producers in `apps/api` publish events (see `docs/kafka-producer-setup.md`,
  `03-kafka.md`).
- A dedicated `apps/kafka-worker` service consumes them via `@nestjs/microservices`' Kafka
  transport, with multiple consumer groups (CDC sync, audit, etc.).
- Deployment is GKE via ArgoCD (`infra/k8s/`). There is no Redis Deployment, no
  `redis`/`ioredis` in application code, and no `REDIS_*` env keys (`.env.example` has
  none). The only "redis" strings in the tree are OpenTelemetry auto-instrumentation
  packages and a `TODO` in `accounts.service.ts` musing about a distributed cache someday.

So nothing below runs today. This is "if we needed a job queue, here is where it would go
and why."

### 7.2 Candidate use cases in this codebase

**1. Report PDF generation — the strongest candidate.** Today
`apps/api/src/modules/reports/reports.controller.ts` `exportPdf` calls
`reports.service.ts` → `buildReportPdfBuffer` → `report-pdf-playwright.util.ts`, which
**launches headless Chromium and renders a PDF inside the HTTP request**, returning a
`StreamableFile`. It even hand-rolls its own concurrency limiter — a module-level counter
`activePdfJobs` capped at `REPORT_PDF_DEFAULT_MAX_CONCURRENT = 2` with a busy-wait — and
carries a **120-second timeout**. That homegrown limiter is *literally a tiny, in-process,
non-durable job queue*. Every problem from §1 is present: slow work in the request path, a
crash loses the render, no retries, a concurrency cap that can't be shared across pods (each
API pod counts to 2 independently), and a documented production failure
(`docs/report-pdf-export-chromium-runtime-issue.md` — Chromium missing in the image caused
500s). A BullMQ `reports` queue would: return instantly with a `jobId`, run the render on a
**separate worker pod** with a **shared** concurrency limit and **rate limit**, retry on
transient Chromium/launch failures with backoff, and let the client poll for the finished
PDF's URL.

**2. Outbound email / notifications.** "Email this statement," "notify on batch
consolidation." Classic fire-and-forget-with-retry work: enqueue, let a worker call the
email provider with an idempotency key, retry with exponential backoff on `5xx`, dead-letter
after N attempts.

**3. Retryable outbound webhooks / side-effects.** Any call to a flaky third party (FX-rate
providers, banking/ACH partners) where you want *at-least-once with backoff* rather than
failing the user's request. Give each call an idempotency key; let BullMQ own the retry
schedule.

**4. Scheduled reconciliation / housekeeping.** A nightly "reconcile the ledger / re-check
drain-to-zero invariants" job, or a periodic exchange-rate poll, fits BullMQ's repeatable
(Job Scheduler) jobs — with the Redis-shared schedule guaranteeing it runs *once* across all
worker pods, not once per pod (which a naive `cron` in each container would give you).

### 7.3 BullMQ vs Kafka — the comparison that matters

Juniors constantly ask "we have Kafka, why would we add BullMQ?" They solve *different
problems*. The one-line intuition:

> **A task queue (BullMQ) is about work that must be DONE.
> An event log (Kafka) is about facts that HAPPENED.**

BullMQ answers *"please do this thing (once), and retry it if it fails."* Kafka answers
*"this fact occurred; anyone who cares can read it, now or later, as many times as they
want."*

| Dimension | **BullMQ** (task queue) | **Kafka** (event log / stream) |
| --- | --- | --- |
| **Core abstraction** | A **job**: a command to do specific work | An **event**: an immutable record of something that happened |
| **Mindset** | Imperative — "do X" | Declarative fact — "X happened" |
| **Consumption** | Each job processed by **one** worker, then gone | Each event read by **many** independent consumer groups |
| **After processing** | Job is removed (ephemeral; trimmed) | Event stays in the log for the whole **retention** window; replayable |
| **Replay history** | No — a completed/removed job is gone | Yes — a new consumer can re-read from the start |
| **Per-item retry/backoff** | **First-class** (attempts, exp. backoff, per job) | Not built in — you build retry topics / DLQ yourself |
| **Delayed / scheduled** | **First-class** (`delay`, cron via Job Scheduler) | Not native (needs extra tooling) |
| **Ordering** | Not guaranteed (priorities exist; not ordered delivery) | **Ordered per partition** |
| **Backing store** | **Redis** (in-memory; needs Memorystore/HA on GCP) | Distributed commit log on disk (brokers) |
| **Throughput ceiling** | High, bounded by Redis | Very high; built for massive streams |
| **Scaling model** | Add worker processes; concurrency per worker | Add partitions + consumers in a group |
| **Typical latency** | Very low (ms) | Low, but tuned for throughput over latency |
| **Best for** | Commands, side-effects, jobs, cron, retryable calls | Event sourcing, CDC, streaming, fan-out to many services |
| **Operational cost** | Just needs Redis | Brokers, ZooKeeper/KRaft, schema registry, more moving parts |
| **In this repo** | **Not used** | **Used** (`apps/kafka-worker`, Debezium CDC, audit) |

**Rule of thumb:**
> **Commands / jobs → BullMQ. Events / streams → Kafka.**
> If the sentence is "**do** this (and retry it, or later, or on a schedule)," reach for
> BullMQ. If the sentence is "this **happened**, and multiple systems may need to react and
> could need to replay history," reach for Kafka.

**They coexist happily — and this repo is a natural place for that.** Kafka is exactly right
for what it does here: CDC from `gibp04`, audit event streams, fan-out to independent
consumer groups — durable, ordered, replayable *facts*. But Kafka is a clumsy fit for "render
this one PDF now and retry on failure," because you'd be reinventing per-job retry, backoff,
delay, and one-worker-does-it-once *on top of* an event log that fundamentally wants to
deliver the same fact to *many* consumers and keep it forever. The idiomatic split: **Kafka
carries the facts; BullMQ carries the to-dos.** A common combined pattern is "a Kafka
consumer reacts to an event by enqueuing a BullMQ job" — Kafka for the durable ordered
stream, BullMQ for the retryable unit of work it triggers.

**The cost of adopting it here.** BullMQ needs Redis, which this stack does not have. On GCP
you'd provision **Memorystore for Redis** (managed, with an HA/replica tier — see §8 on
Redis as a single point of failure), add `REDIS_*` config, and — mirroring the existing
Kafka split — likely add a **separate worker deployment** so PDF/Chromium work is isolated
from the API pods. That's a real operational addition, so BullMQ earns its place only when a
use case genuinely needs *durable, retryable, one-worker-does-it jobs* — which the PDF export
plausibly does, and which Kafka would serve awkwardly.

---

## 8. Production concerns

Running a queue in production is mostly about four failure modes: workers can't keep up,
Redis dies, jobs poison themselves, and memory grows unbounded. Here's how each is managed.

**Worker scaling & concurrency.** Throughput ≈ `concurrency × worker replicas`. For
**I/O-bound** jobs (network calls, waiting on Chromium) crank `concurrency` up — Node
juggles awaits cheaply. For **CPU-bound** jobs (parsing, crypto) keep `concurrency` low
(often 1–2) per pod and scale by adding *replicas*, because CPU work blocks the single Node
thread. Scale replicas on **queue depth** (waiting count) — that is the true "am I keeping
up?" signal. On GKE this is a HorizontalPodAutoscaler driven by a queue-depth metric.

**Redis is a single point of failure.** Every job lives in Redis. If Redis goes down, *the
whole queue stops* — no enqueue, no processing. This is the biggest operational difference
from Kafka's replicated brokers. Mitigations: run **managed HA Redis** (GCP Memorystore
Standard tier gives a replica + automatic failover), pick sensible timeouts so workers
reconnect cleanly after a blip, and understand that Redis persistence (RDB/AOF) is *best
effort* — a hard crash can lose the last fraction of a second of writes. For most job
workloads that's acceptable; for money-movement it's a reason to also have a
**source-of-truth** (e.g. a DB row you reconcile against), not just the queue.

**Retry/backoff tuning & poison jobs.** A **poison job** is one that will *never* succeed —
malformed data, a permanently-deleted record — but keeps failing and retrying, burning
resources. Defenses: cap `attempts` (never infinite), use exponential backoff + jitter so
retries don't stampede, and treat the **`failed` list as a dead-letter queue** you actively
monitor and drain. Distinguish *transient* errors (retry) from *permanent* ones (fail fast —
don't retry a `400`); you can throw an `UnrecoverableError` to tell BullMQ "don't bother
retrying this one."

**Idempotency (again).** Because delivery is at-least-once (§4.8), every side-effecting
handler needs an idempotency strategy or you will double-charge / double-send under retries
and stalls. This is a *production* concern, not a nicety.

**Job retention & memory.** Redis is RAM. Set `removeOnComplete` (often aggressive — keep a
small count or delete immediately) and `removeOnFail` (keep longer, for debugging).
Unbounded retained jobs is the classic way to OOM Redis and take down the queue (§10).

**Monitoring.** Alert on: **queue depth** (waiting count climbing = workers behind or dead),
**failed count** (spiking = a bad deploy or a sick dependency), **stalled count** (workers
dying or blocking the event loop), and **oldest job age** (something's stuck). Wire these
from `QueueEvents` / `queue.getJobCounts()` into your metrics stack.

**Graceful shutdown on pod termination.** GKE sends `SIGTERM` then waits (the grace period)
before `SIGKILL`. Your worker must call `worker.close()` on `SIGTERM` to stop pulling new
jobs and finish in-flight ones. Also set the pod's `terminationGracePeriodSeconds` longer
than your longest job (or your longest job will be killed mid-flight and rely on stalled
recovery — a duplicate). For a 2-minute PDF render, a 30-second grace period is not enough.

**At-least-once duplicates** are a permanent fact of life here, not a bug to be fixed —
budget for them with idempotency rather than hoping they won't happen.

---

## 9. Operations & debugging playbooks

Concrete "it's broken, what do I do" recipes. Keep this section bookmarked.

**Inspect the queues.**
- **Bull Board** (§6.5) is the fastest: open the dashboard, look at waiting/active/
  completed/failed counts and click into individual jobs to see data + error.
- **`redis-cli`** when you need the raw truth:
  ```bash
  redis-cli LLEN bull:reports:wait        # how many jobs waiting
  redis-cli LLEN bull:reports:active      # how many in flight
  redis-cli ZCARD bull:reports:failed     # how many dead-lettered
  redis-cli ZCARD bull:reports:delayed    # how many scheduled for later
  redis-cli HGETALL bull:reports:report-42 # one job's full state
  ```
- Programmatically: `await queue.getJobCounts()` returns `{ waiting, active, completed,
  failed, delayed, ... }`.

**"Jobs stuck in `waiting`, nothing processes them."** The rail is full but no cook is
grabbing tickets. Check, in order: (1) **Is a worker running at all?** (worker pod crashed /
never deployed) — the #1 cause. (2) **Does the worker's queue name exactly match** the
producer's? A typo means the worker listens to a different rail. (3) **Concurrency saturated
by long jobs?** All worker slots busy on slow jobs → new ones wait. (4) **Rate limiter**
throttling starts. (5) Worker can reach the *same* Redis the producer wrote to (right
host/db).

**"Jobs failing / retrying forever."** Open a failed job and read `failedReason` /
`stacktrace` (Bull Board shows both). Then: is it **transient** (vendor `503` → let backoff
handle it) or **poison** (bad data → it'll never pass, so fix `attempts`, throw
`UnrecoverableError`, or fix the data)? If retries hammer a struggling dependency, lengthen
backoff and add jitter. Anything that exhausted attempts sits in `failed` — that's your DLQ;
triage it, fix root cause, then bulk-**retry** from Bull Board or `queue.getFailed()` +
`job.retry()`.

**"The same work happened twice."** Almost always a missing idempotency guard meeting
at-least-once delivery, or a job **enqueued twice** without a stable `jobId`. Fixes: pass a
deterministic `jobId` (e.g. `report-${id}`) so duplicate *enqueues* de-dupe; add a
check-before-act guard in the handler; use idempotency keys on external calls. If duplicates
appear under load/deploys specifically, suspect **stalled re-delivery** from a handler that
blocks the event loop past its lock duration (§4.3, §10).

**Drain / clean a queue.**
```ts
await queue.drain();                 // remove waiting + delayed jobs (keeps active)
await queue.clean(0, 1000, 'failed');   // remove up to 1000 failed jobs
await queue.clean(3_600_000, 5000, 'completed'); // completed older than 1h
await queue.obliterate();            // NUKE the entire queue — use with extreme care
```

**Pause / resume** (e.g. a dependency is down and you want to stop processing but keep
*accepting* jobs):
```ts
await queue.pause();   // workers stop picking up new jobs; producers can still add
await queue.resume();  // workers resume
```
This is the humane way to handle "the email provider is down for an hour" — pause the queue,
let jobs pile up in `waiting`, resume when the provider recovers; nothing is lost.

---

## 10. Gotchas & hard-won lessons

The mistakes that bite everyone at least once. Learn them here instead of in an incident.

1. **Handlers MUST be idempotent — delivery is at-least-once.** This is the big one (§4.8).
   Your handler *will* occasionally run twice (a worker died between doing the work and
   marking it complete; a stall re-delivered it). If "twice" means "charged twice / emailed
   twice / double-posted a ledger entry," you have a production incident. Guard with stable
   `jobId`s, idempotency keys, and check-before-act. Never assume exactly-once.

2. **Don't block the event loop in a handler.** A long synchronous computation (a giant `for`
   loop, a massive `JSON.parse`, sync crypto) stops the worker from renewing its job **lock**.
   BullMQ then thinks the job **stalled** and hands it to *another* worker — while the first
   is still running it. Result: duplicate execution. Keep handlers `async` and yield;
   offload true CPU-heavy work to a child process/worker thread or a bigger box.

3. **Unbounded retained jobs eat Redis memory.** Redis is RAM. If you don't set
   `removeOnComplete` / `removeOnFail`, completed and failed jobs accumulate forever and
   eventually OOM Redis — which takes down *the whole queue*. Always cap retention; keep
   failures longer than successes for debugging, but keep both bounded.

4. **Deploy the web tier and the worker tier separately.** Enqueueing (in the API) and
   processing (in a worker) are different resource profiles and different failure blast
   radii. A runaway Chromium render should not be able to OOM the pods serving your HTTP API.
   This repo already does exactly this split for Kafka (`apps/api` vs `apps/kafka-worker`);
   mirror it for BullMQ.

5. **Don't put huge payloads in job `data` — store a reference.** Job data is serialized into
   Redis (RAM) and read on every state transition. Stuffing a 10 MB PDF, a base64 image, or a
   giant result set into `data` bloats Redis and slows everything. Store the blob in
   object storage / the DB and put a **small reference** (an id, an S3 key) in the job. Same
   for return values — keep them small.

6. **A Redis outage stops everything.** Unlike Kafka's replicated brokers, if Redis is down
   nothing enqueues and nothing processes. Run HA managed Redis (Memorystore Standard) and
   don't treat the queue as your only source of truth for critical state — reconcile against
   a durable DB.

7. **`maxRetriesPerRequest: null` on worker connections.** Forget this and workers throw
   confusing connection errors under BullMQ's blocking commands (§5.2). It's the most common
   "why won't my worker start" footgun.

8. **A missing/short pod grace period truncates long jobs.** If your longest job is 2 minutes
   but `terminationGracePeriodSeconds` is 30, deploys kill jobs mid-flight and lean on stalled
   recovery — i.e. duplicates. Size the grace period to your job length (§8).

9. **`repeat`/scheduled jobs need a stable id, or you get duplicates.** Registering a
   repeatable job on every app boot without a stable scheduler id can create multiple
   schedulers → the nightly job fires several times. Use `upsertJobScheduler` with a fixed id
   (§5.6).

---

## 11. Glossary & further reading

**Glossary**

- **Queue** — a named, Redis-backed channel of jobs; you add jobs to it.
- **Job** — one unit of work: a name, a JSON `data` payload, and options.
- **Producer** — code that enqueues jobs (`queue.add`).
- **Worker** — a process that pulls jobs and runs the handler; has `concurrency`.
- **Processor / handler** — the function that does a job's work.
- **Job lifecycle** — `waiting → active → completed | failed`, with `delayed` for
  later/backoff.
- **Attempts** — max tries before a job is `failed`.
- **Backoff** — the wait between retries (fixed or exponential; add jitter).
- **Delayed job** — a job scheduled to run at a future time (stored in a Redis sorted set).
- **Repeatable / scheduled job (Job Scheduler)** — a job that re-enqueues on a cron/interval.
- **Rate limiting** — a ceiling on jobs started per time window, across the queue.
- **Concurrency** — how many jobs one worker runs in parallel.
- **Priority** — lets some waiting jobs jump ahead of others.
- **Flow (FlowProducer)** — parent/child job dependencies; parent runs after children.
- **Events** — a Worker's local notifications (`completed`, `failed`, `progress`).
- **QueueEvents** — a queue-wide event listener reading Redis's event stream.
- **Job `data`** — the input payload (keep it small; store references, not blobs).
- **Return value** — what the handler returns; stored on the completed job.
- **`jobId`** — a custom stable id; duplicate enqueues with the same id de-dupe.
- **Idempotency** — designing a handler so running it twice equals running it once.
- **Stalled job** — an `active` job whose worker died/hung; auto-recovered via lock TTL.
- **At-least-once** — BullMQ's delivery guarantee: a job may run more than once, never zero.
- **Dead-letter queue (DLQ)** — here, the `failed` list where exhausted jobs land for triage.
- **Poison job** — a job that can never succeed and keeps retrying; cap attempts / fail fast.
- **Backing store: Redis** — the in-memory data store BullMQ persists everything to
  (`05-redis.md`); on GCP: **Memorystore for Redis**.

**Further reading**

- BullMQ documentation (guide + API): https://docs.bullmq.io/
- BullMQ GitHub: https://github.com/taskforcesh/bullmq
- `@nestjs/bullmq` — NestJS Queues docs: https://docs.nestjs.com/techniques/queues
- Bull Board (queue dashboard): https://github.com/felixmosh/bull-board
- Redis docs (data types this all rests on): https://redis.io/docs/latest/
- GCP Memorystore for Redis: https://cloud.google.com/memorystore/docs/redis
- **In this repo:** `03-kafka.md` (what async work actually uses today), `05-redis.md`
  (the Redis dependency BullMQ would need), `docs/kafka-producer-setup.md`,
  `docs/report-pdf-export-chromium-runtime-issue.md` (the inline-PDF pain that motivates
  §7's slot-in), `apps/api/src/modules/reports/report-pdf-playwright.util.ts` (the homegrown
  in-process limiter BullMQ would replace), and `apps/kafka-worker/` (the separate-worker
  deployment pattern you'd mirror).

---

*Chapter status: BullMQ is **not** used in this repository. This chapter teaches BullMQ from
first principles and is explicit throughout about where it would fit versus the Kafka-based
architecture that is actually in production. When in doubt: **commands/jobs → BullMQ;
events/streams → Kafka.***
