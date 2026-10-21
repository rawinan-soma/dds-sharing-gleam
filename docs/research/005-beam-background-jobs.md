# Durable background jobs on the BEAM

- **Question:** What can replace BullMQ + Redis for the extraction pipeline while meeting spec §7.6 (retry, resume and stall) and §7.7 (queue mechanics): jobs surviving a restart, one-at-a-time concurrency, delayed or deferred jobs (ADR 0014), and the 72-hour deletion? The options compared are plain OTP processes with Postgres as the durable record, existing Gleam, Erlang and Elixir job libraries, and keeping Redis. Each is judged on durability, operational cost, and fit to the pilot's `extraction` and `scheduler` modules.
- **Issue:** rawinan-soma/dds-sharing-gleam#6 (map #1; blocks #9)
- **Date:** 2026-10-21
- **Primary sources:** the pilot's code at `33a9491` (`apps/api/src/extraction/*`, `apps/api/src/scheduler/*`, `apps/api/src/mail/mail-queue.ts`); `docs/spec.md` §7.6–§7.8, §9.3, §9.5, §10.8, §13.2, §14.4, §15.1, §15.3; ADR 0008 and ADR 0014; the hex.pm package API (`https://hex.pm/api/packages/<name>`, read 2026-10-21); GitHub repo metadata and source (`gh api repos/<owner>/<repo>/...`); and hexdocs for `gleam_otp`, `gleam_erlang`, `pog`, `m25` and `oban`. Every claim names its source. ⚠️ marks a claim I did not verify.

## Answer

**Use plain OTP processes, with the existing `extraction_job` table as the queue.** The pilot already treats Postgres as the system of record and BullMQ as nothing more than an executor (§7.7: *"BullMQ executes; PostgreSQL is the system of record"*). Every durability property the spec asks for already lives in Postgres rows and the 60-second tick, not in Redis. Dropping Redis removes a container, its AOF config, and the whole class of "Postgres row with no live BullMQ job" faults that the reconcile exists to patch. What replaces BullMQ is a small set of parts:

- a supervised runner actor;
- a claim query;
- a timer for the stall guard;
- a `not_before` column for the low-disk deferral.

| | Durability | Ops cost | Fit to the pilot |
|---|---|---|---|
| **A. Plain OTP + Postgres** (`gleam_otp`, `pog`) | Same as today's Postgres truth. One store, so nothing needs reconciling across stores | **Lowest.** No new service, and the Redis container goes | **Best.** It maps 1:1 onto the tick, `extraction_job` and the stall guard, and stall cancellation becomes stronger (`process.kill`) |
| **B. `m25`** (Gleam, Postgres-backed) | Good, because it uses `FOR UPDATE SKIP LOCKED` in the same database | Low. It adds an `m25` schema and its own migration CLI | **Partial.** It adds a second job table beside `extraction_job`, has a hard per-job deadline (§7.6 rejects a duration cap), and is a young library with one release |
| **C. Oban** (Elixir, via Gleam's Mix-dependency support) | Excellent and mature | Build cost is high: an Elixir toolchain, an Ecto Repo and Postgrex beside `pog`, and workers written in Elixir | **Poor.** It breaks "this repo ends Gleam-only" (map #1), and global concurrency is a paid Pro feature |
| **D. Keep Redis** (`valkyrie` or `radish`) | Only as good as AOF, plus the reconcile | **Highest.** It keeps a Redis container and AOF config | **Poor.** BullMQ does not exist on the BEAM, so you would build a queue on Redis yourself: all of A's work plus a second store |

**Leaning: A.** Two drift tickets come with it (see [Drift this forces](#drift-this-forces)). The more important one is **§10.8**: today the rendered Delivery, the only place the raw Download token exists, is kept *in Redis* for 72 hours so it can be resent. Without Redis it needs a new home, and the spec has already declined Postgres for it.

## What the pilot actually asks of BullMQ

Read from the pilot's code, the dependency on BullMQ is narrow.

**Used:**

- **The job id is the Postgres row id.** `ExtractionQueue.enqueue` passes `jobId` equal to `extraction_job.id`, so that the reconcile can ask *"is there a live BullMQ job for this Postgres row"* (`extraction-queue.ts`; `extraction-jobs.repository.ts` `insertQueuedJob`).
- **Concurrency is 1.** `createExtractionWorker` sets `concurrency: EXTRACTION_CONCURRENCY` = 1 (`extraction-worker.ts`, `extraction.config.ts`; spec §13.2).
- **One delayed job.** When free disk is below the 1 GB floor, the worker writes `job_deferred_low_disk` and calls `job.moveToDelayed(now + 5 min)` (`extraction-worker.ts`; `DISK_RECHECK_DELAY_MS` in `extraction.config.ts`; spec §7.8).
- **Retention.** Completed extraction jobs are removed (`removeOnComplete: true`) and the last 1000 failures are kept (`removeOnFail`) for Bull Board only (`extraction-queue.ts`; spec §14.4).
- **Mail** (`mail-queue.ts`):
  - every job runs with `attempts: 1`, and the tick calls `job.retry('failed')` once a row's 15 minutes are up;
  - a sent **Delivery** job is kept for `DOWNLOAD_TOKEN_LIFETIME_HOURS`, because *"the rendered message is the only place the raw Download token exists"* (spec §10.8).

**Not used, or deliberately rejected:**

- **BullMQ retries or backoff for extraction.** The code-level retry (3 attempts, 1 s then 2 s, from page 1) lives inside the pipeline (`EXTRACTION_DEFAULTS`; spec §7.6), and `no-automatic-rerun.spec.ts` pins that a failed job is not rerun automatically.
- **BullMQ's own stalled-job detection.** The stall guard is the app's own `StallGuard`: it fails the job if no Report code completes within 120 s, then aborts the in-flight call (`stall-guard.ts`; `extraction-worker.ts`).
- **Repeatable jobs.** They were *rejected*: *"a schedule living only in Redis is a schedule a Redis loss silently cancels"* (spec §15.3).
- **The scheduler itself.** It is a plain `setInterval` every 60 s that takes `pg_try_advisory_lock` (`scheduler.module.ts`, `tick.ts`).

**The 72-hour deletion never touches the queue.** Object deletion, materialising `expired`, the collection lapse and pruning all run *inline under the lock* (spec §15.3, decided in `dds-sharing` #72; `tick.ts` `deleteExpiredObjects`). Expiry itself is a read-time predicate (§15.1). So whatever replaces BullMQ, §9.3 and §9.5 need only a 60-second timer and the advisory lock.

**ADR 0014's "defer" is not a delayed job.** The ADR defers the *Alert*: the Alert stays open while a Re-run's job runs, and the system clears it when that job completes. The Re-run itself enqueues an ordinary immediate job. The only delayed work in the pilot is the low-disk recheck (5 min) and mail retries (15 min, and those are already started by the tick rather than by BullMQ).

## Option A: plain OTP with Postgres as the queue

### Building blocks (all verified on hexdocs)

- **`gleam_otp` 1.3.0** (2026-08-09; 909,846 downloads), from hex.pm.
  - `actor.new(state)`, `actor.on_message(builder, handler)`, `actor.start(builder) -> Result(Started(return), StartError)`.
  - *"Actors handle messages one at a time, in order."* `Started` carries a `pid` for monitoring (gleam-otp.hexdocs.pm `gleam/otp/actor.html`).
- **`gleam_erlang` 1.3.0** (2025-08-03), from hex.pm.
  - `process.send_after(subject, delay, message) -> Timer` and `process.cancel_timer(timer)`.
  - `process.kill(pid)`: *"Send an untrappable `kill` exit signal to the target process"*.
  - `process.monitor(pid)` and `process.spawn` (gleam-erlang.hexdocs.pm `gleam/erlang/process.html`).
- **`pog` 4.1.0** (2025-07-14; 141,644 downloads), from hex.pm.
  - `pog.supervised(config)` starts the pool in a supervision tree, and `pog.named_connection(name)` refers to it.
  - `pog.transaction(pool, callback)` commits on `Ok` and rolls back on error or panic.
  - Its docs do not mention LISTEN/NOTIFY, so wake-ups are messages or polling (pog.hexdocs.pm `pog.html`).

### How each pilot mechanism maps

| Pilot (BullMQ) | OTP replacement |
|---|---|
| Approve writes the `extraction_job` row, then enqueues after commit (§7.7) | Unchanged row write. After commit, send the runner a `Wake` message. If the message is lost, the next tick pass finds the `queued` row anyway |
| `concurrency: 1` | One named runner actor runs one job at a time. For a **global** N=1 that survives scaling the container (§15.3 keeps "the deployment shape free"), let Postgres enforce it with a partial unique index over `running` rows, e.g. `CREATE UNIQUE INDEX ... ON extraction_job ((true)) WHERE status = 'running'`, so a second claimer fails. ⚠️ Design sketch, not tested |
| BullMQ claims the next job | The runner claims the oldest due `queued` row (`... WHERE status='queued' AND (not_before IS NULL OR not_before <= now()) ORDER BY created_at LIMIT 1 FOR UPDATE SKIP LOCKED`) and marks it `running`, the same shape `m25` uses (see B) |
| `moveToDelayed(now + 5 min)` on low disk | Write `job_deferred_low_disk`, set `not_before = now() + 5 min`, leave the row `queued`, and arm `send_after(runner, 5 min, Wake)`. It is durable because the deferral is a column, not a timer that only exists in memory |
| `StallGuard` plus an `AbortController` | The job body runs in a spawned, monitored process. Each `code_fetched` sends a `Progress` message, and the runner re-arms a `send_after` stall timer. When the timer fires, `process.kill(job_pid)`. This is **stronger** than today's abort: the process, its in-flight HTTP call and any backoff sleep all die at once, so `stallSettleMs` is no longer needed. **Constraint:** the call count for `job_failed.callsMade` (§13.6) must be sent to the runner as the calls happen, because it would die with the job process |
| Worker restart, then resume from checkpoint (§7.6) | Unchanged. Code checkpoints are on the scratch volume, and the restarted runner reclaims the row |
| Startup reconcile: "`queued`/`running` with no live BullMQ job" (§7.7, §15.3) | Simpler. A `running` row this process is not executing (the runner knows its one job id), and that shows no progress past the stall window, is reset to `queued` or failed. Rows that are only `queued` need no action: they are the queue. The "Postgres says yes, Redis says no" fault disappears |
| Mail: `attempts: 1`, tick-driven retry from the failed set | Retry state is already in `mail_delivery` rows (`tick.ts` `retryDueMail`). A send becomes a supervised task the tick starts. **The rendered message is the open question** (see Drift) |
| `@nestjs/schedule` `setInterval` 60 s plus `pg_try_advisory_lock` | A ticker actor that re-arms `send_after(self, 60_000, Tick)` each pass. The advisory lock is a session lock, so take it on one checked-out connection for the whole pass, e.g. `pg_try_advisory_xact_lock` inside a `pog.transaction`, or a dedicated size-1 pool. ⚠️ Not prototyped |
| Bull Board (§14.4) | None. This is already in map #1's *Not yet specified → Observability*. `extraction_job` rows plus `/health` cover the watcher duty, and §14.4 says Bull Board carries none |

**Durability:** each piece of state is either a Postgres row or something rebuilt from one: the `queued`/`running` status, `not_before`, `last_progress_at`, and `mail_delivery` attempts. Timers are only wake-ups, and the tick's poll makes up for any one that is lost. So the claim *"a restart needs no catch-up"* (§15.3) holds exactly as it does today.

**Ops cost:** one fewer container. There is no AOF config to get wrong, and no Redis volume, backup or health check.

**Build cost:** we own a claim query, a runner actor (about 150–300 lines ⚠️ estimate), and a ticker. We also lose a ready-made job inspector.

## Option B: `m25`, a Gleam library on Postgres

- **Maturity**, from hex.pm and GitHub:
  - v1.0.0 is its **only release** (2025-08-28); the repo was last pushed 2026-08-25; 838 downloads; 36 stars.
  - It depends on `pog >= 4.1.0 and < 5.0.0` and `gleam_otp >= 1.0.0 and < 2.0.0` (`gleam.toml`), so it is compatible with today's Gleam stack.
  - Open issues include #6 and #8, *"Add handler retry policies"* and *"Apply retry policies to worker failures"*, and #9, *"Harden deadline timeout cleanup"* (all opened 2026-08-25).
- **Mechanics**, from source and docs:
  - Jobs live in an `m25.job` table, and the claim is `update ... where id in (select ... for update skip locked)` ordered by `created_at` (`src/m25/internal/sql/reserve_jobs.sql`).
  - Per-queue settings are `max_concurrency` (*"per application, not per cluster"*), `default_job_timeout`, `poll_interval`, `heartbeat_interval`, `allowed_heartbeat_misses` and `reserved_timeout`.
  - Job options are `schedule(at:)`, `retry(max_attempts:, delay:)`, `unique_key` and `timeout`.
  - `enqueue(conn, queue, job)` takes a `pog.Connection`. The connection that `pog.transaction` hands its callback is also a `pog.Connection`, so an enqueue *in the approval transaction* looks possible. ⚠️ Not tested.
  - Failure reasons are `Errored | HeartbeatTimeout | JobTimeout | Crash` (m25.hexdocs.pm `m25.html`).
  - Its stated semantics are *"at least once"* (README).
- **Fit:**
  - **It adds a second job record** (`m25.job`) beside `extraction_job`. That keeps a two-record shape, though with transactional enqueue the two cannot drift the way Redis and Postgres can.
  - **Its timeout is a deadline** (`JobRecord.deadline`, set from the configured timeout when the job starts). That is the duration cap §7.6 rejects as dead code. You would set a large timeout and still write the stall guard yourself.
  - `retry` must be `max_attempts: 1` to keep *no automatic Re-run*.
  - `max_concurrency: 1` is per node, so global N=1 still needs our own guard.
  - What is left for `m25` to do is the claim and the scheduled-at column: the two easiest parts of A. The price is a dependency with one release and one maintainer that owns a schema and a migration CLI (`gleam run -m m25 migrate`).
- **Ruled out:** `bg_jobs` 0.5.2 (last release 2025-03-12; repo last pushed 2025-05-12). It pins `gleam_otp < 1.0.0` and `pog < 4.0.0` (`gleam.toml`), which conflicts with current `gleam_otp` 1.x and `pog` 4.x. Its README says *"Docs are WIP"* and *"Durability: TODO: update, this is not exactly correct any more"*.

## Option C: Oban (Elixir)

- **Maturity:** v2.24.2 (2026-10-14), 110 releases since 2019, 28.4M downloads (hex.pm). The repo was pushed 2026-10-20 and has about 4k stars (GitHub).
- **Gleam can depend on it:** *"Erlang rebar3 packages and Elixir mix packages can also be added as dependencies without any change in syntax"* (gleam.run `gleam-toml-reference`). ⚠️ This needs Elixir in the build image; the page does not say so, but compiling a Mix package needs it.
- **What it demands:**
  - *"Ecto configured to use Postgrex"*, a `repo:` option without which *"the supervisor won't start"*, an `oban_jobs` migration (`Oban.Migration.up(version: 14)`), and `{Oban, config}` in the supervision tree (oban.hexdocs.pm `installation.html`, `Oban.html`).
  - That means an Elixir Ecto Repo module and a Postgrex pool running beside `pog`, plus workers defined with Elixir's `use Oban.Worker`. ⚠️ The installation page does not show the worker module, but it is Oban's standard worker API.
- **Concurrency:** `queues: [extraction: 1]` limits *each queue*. **Global** concurrency is listed under the Pro Smart Engine (*"Enables global concurrency, global rate limiting…"*, `Oban.html`).
- **Strengths:**
  - `Lifeline` rescues jobs orphaned by node shutdown, and `Pruner` cleans up old jobs.
  - Oban Web (2.13.1, 2026-10-08) is a ready Bull Board replacement, but it needs Phoenix LiveView. ⚠️ Not verified beyond the hex.pm description.
- **Fit:** the most robust engine, but it imports a second language, a second database layer and a second migration system into a repo whose map says *"This repo ends Gleam-only."* Its stall and timeout semantics would still be ours.

## Option D: keep Redis

- **BullMQ has no BEAM implementation.** Its queue semantics are Lua scripts driven by the Node client. ⚠️ The Lua-script detail comes from general knowledge; the absence of a BEAM port was checked on hex.pm, which lists no `bullmq` package.
- **Gleam clients exist:** `valkyrie` 4.2.0 (2026-05-16; 4,833 downloads) and `radish` 0.16.0 (2025-01-25). Erlang and Elixir clients: `eredis` 1.7.1 (2023-07-11) and `redix` 1.9.2 (2026-09-17) (hex.pm).
- **Fit:** you would hand-build claim, delay and retention on Redis data structures. That is option A's runner and claim, plus a second store, plus the AOF requirement (§7.7) and the cross-store reconcile. The one thing it keeps literally is §10.8's *rendered Delivery in Redis for 72 h*. It is not worth a service.

## Drift this forces

Per map #1, each of these becomes its own ticket for the repo owner. None is settled here.

1. **§10.8: where the rendered Delivery lives for resend.** Today: *"a sent Delivery's BullMQ job is kept in Redis for 72 hours… Storing the raw token in Postgres, even encrypted, was declined."* Without Redis:
   - **In-memory (ETS)** is lost on every restart or deploy, not just on a Redis loss. Resend would then be refused (remedy: a Re-run) far more often.
   - **Postgres** was declined.
   - **Scratch-file storage** is a third shape the spec has not considered.

   The owner must choose. This applies to options A, B and C alike; only D avoids it.
2. **§7.7, §15.3, §14.4 and §14.5 name BullMQ, Redis, AOF, Bull Board and "BullMQ job payload".** The behaviour carries over (Postgres is truth, the startup reconcile, N=1, a payload that holds no rows), but the text needs re-wording to the OTP mechanism. §7.7's *"Redis runs with AOF persistence"* rule simply drops out.

## Open, not answered here

- Whether the runner's call-count messages give *exactly* the `callsMade` the pilot records on a stall (§13.6). The kill is abrupt, so counting must happen before each call goes out, not after. A prototype or test belongs in the build slice.
- Whether global N=1 should be enforced by the partial unique index, or whether the pilot's single-container deployment makes it moot. §15.3's lock-based reasoning suggests enforcing it.
