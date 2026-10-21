# Gleam libraries for storage and infrastructure

- **Question:** For each storage dependency of `dds-sharing@pilot` (Postgres access with `pg` and Drizzle, schema migrations with drizzle-kit, and S3/MinIO object storage with `minio`), which Gleam package, or which Erlang/Elixir library through FFI, would carry it on the BEAM? For each one: the candidates, how well maintained and mature they are, how they fit the way the pilot uses the dependency, and which gaps would force FFI or our own code.
- **Issue:** rawinan-soma/dds-sharing-gleam#2 (map: #1)
- **Date:** 2026-10-21
- **Primary sources:** the hex.pm package API (`https://hex.pm/api/packages/<name>`: versions, release dates, downloads), the GitHub repo API (last commit, archived flag, licence), package source read from the hex tarballs (`repo.hex.pm/tarballs/<pkg>-<ver>.tar`) or from `main` on GitHub (links below), upstream issue threads, and the pilot's code at `33a9491` (`apps/api/…`, still in this repo's tree). ⚠️ marks claims that were not verified.

## Answer

| Pilot dependency | Recommended on the BEAM | Gaps we would own |
|---|---|---|
| `pg` (Pool) | **`pog` 4.1.0** (lpil), running on Erlang `pgo` | Session-level advisory lock for the tick: there is no public "check out one connection" call outside a transaction (workaround below). `timestamptz` and `inet` need casts once Squirrel is used. |
| Drizzle (typed queries) | **`squirrel` 4.7.0**: hand-written `.sql` files become typed Gleam functions that call `pog` | No `timestamptz` or `inet` types: cast in SQL. Codegen needs a live, migrated Postgres (in dev and in CI). |
| drizzle-kit (migrations) | **`cigogne` 5.0.6**: hand-written SQL up/down files, hash-checked, applied with `pog` | No schema diffing or generation (we write SQL, which is already what the pilot's migrations are). No lock against two migrators running at once: wrap it in our own `pg_advisory_lock` when that matters (tests). |
| `minio` (S3 client) | **`bucket` 1.5.0** (lpil, sans-IO) + **`aws4_request`** + **`gleam_httpc`** | No lifecycle-configuration call, so we build PutBucketLifecycleConfiguration ourselves. `get_object` does not take a range and treats 206 as an error, so we add the `Range` header and read the response ourselves. `head_object` drops the size, so we read `content-length` from the raw response. Bodies are whole `BitArray`s, not streams. |

All four are Gleam-native, run on Erlang, and are Apache-2.0 or MIT. Nothing needs Elixir in the build, and no foreign library beyond what `pog` already pulls in. The FFI we might write ourselves is at most one function, a `pgo:checkout/1` binding for the tick lock, and even that has a pure-`pog` workaround. The S3 gaps cost roughly 100 lines of our own Gleam on top of `bucket`.

**Main risks:**
1. `pog`'s last release is **2025-07-14**, 15 months ago. Commits continue (last 2026-03-09), the author is Gleam's creator, and Squirrel, Cigogne and Parrot all build on it.
2. `pgo` `main` carries an **unreleased breaking change** (2026-09-10, "change default pool name to pgo_default"). `pog` allows any `pgo` `>= 0.14.0 and < 1.0.0`, so a future `pgo` 0.21 lands on `gleam update`. Pin `pgo` in `manifest.toml` and upgrade on purpose.
3. Squirrel's missing `timestamptz` touches **23 columns** in the pilot's schema. Either cast in every query, or let the Gleam app's schema use `timestamp` holding UTC. That is a free choice, since nothing migrates (map #1).

## What the pilot actually uses

### Postgres (`pg` ^8.16.3, `drizzle-orm` ^0.44.5)

Read from `apps/api/src/db/*`, the `db.transaction` call sites, `scheduler/tick.ts` and `requests/requests.service.ts`:

- **One `pg.Pool` per process** on the app role's URL. A separate **owner URL (`DATABASE_URL`) is used only to migrate** (`db/database.module.ts`, `db/migrate.ts`, `drizzle.config.ts`). The app role `dds_app` gets column-level grants inside migrations (`migrations/0006_decision_state_grant.sql`: `GRANT UPDATE ("state") ON "request" TO dds_app`).
- **Transactions everywhere:** about 20 `db.transaction(async (tx) => …)` sites (decisions, reviewer auth and sessions, accounts, download tokens, throttles, alerts, in-flight, tick). None sets an isolation level (`grep isolationLevel` finds nothing), so all run at the default READ COMMITTED.
- **Advisory locks, three kinds** (`db/advisory-locks.ts`):
  - transaction-scoped `pg_advisory_xact_lock(hashtextextended('request-submit:'||email, 0))`, the per-email submit lock (`requests.service.ts:67`);
  - transaction-scoped `pg_advisory_xact_lock(64)`, the two-reachable-Reviewers check (`reviewer-accounts.ts:151`);
  - **session-scoped** `pg_try_advisory_lock(7_215_003)` on a checked-out client, held for a whole tick pass that runs its own transactions on the pool, then `pg_advisory_unlock` (`scheduler/tick.ts:127-140`);
  - the test suite also holds a session lock (`6_425_001`) on its own connection to serialise migrations (`test/support/scratch-database.ts:33`).
- **Upserts** (`onConflictDoUpdate` / `onConflictDoNothing`), `returning()`, and 38 raw `` sql`…` `` fragments.
- **Column types** (`db/schema.ts`): 16 `uuid`, 26 `timestamp` (23 `withTimezone: true`), 24 `text`, 10 `pgEnum`, 3 `jsonb`, 4 `inet`, 2 `date`, `smallint`/`integer`/`bigint`/`serial`, 9 `check()` constraints.
- **No dynamic query building:** no spread `and(...conditions)`, no `ilike`. Every query has a fixed shape, which suits a SQL-file codegen tool.
- No streaming or cursor reads, and no LISTEN/NOTIFY.

### Migrations (`drizzle-kit` ^0.31.5)

There are 12 SQL files in `db/migrations/` (`0000_no_op` … `0011_drop_request_event_submitted_ip`), including a **seed** (`0003_seed_province.sql`) and **grants**. They run through `drizzle-orm/node-postgres/migrator` as a host step (`pnpm db:migrate`) under the owner role. drizzle-kit's diffing (`schema.ts` → SQL) is the only thing a SQL-file tool would not replace, and the Gleam app has no `schema.ts` to diff from.

### Object storage (`minio` ^8.0.7)

Already inventoried in [`004-object-storage-image.md`](004-object-storage-image.md#what-the-app-actually-needs), and re-checked against `extraction/archive-store.ts`:

| S3 operation | Pilot call | Detail that matters for the port |
|---|---|---|
| HeadBucket, CreateBucket | `bucketExists`, `makeBucket` | at startup (`prepareExtractBucket`) |
| PutBucketLifecycleConfiguration | `setBucketLifecycle` | one rule, `Expiration.Days = 3`, `Filter.Prefix = ''` (`bucket-lifecycle.ts`) |
| PutObject | `putObject(key, Buffer, len, {Content-Type: application/zip})` | **single call from an in-memory `Buffer`**, not a streamed upload (`extraction-worker.ts:163`). The archive is one upstream page, `page_size=10000` (ADR 0008) |
| HeadObject | `statObject` | needs **size**; `NotFound` means deleted |
| GetObject, whole or ranged | `getObject`, `getPartialObject(start, length)` | proxied to the client through the API with 200 or 206, `Content-Range` and `Content-Length` (`delivery.controller.ts:84-101`) |
| DeleteObject | `removeObject` | at the token's expiry |

There are **no presigned URLs** (004 already checked: `grep presign` is empty). Downloads are always **served by the app**, so the token stays the only way in.

## Postgres client

### Candidates

| Package | Kind | Latest | Released | Repo activity | Downloads (all / recent) | Licence |
|---|---|---|---|---|---|---|
| **`pog`** | Gleam, wraps `pgo` | 4.1.0 | 2025-07-14 | last commit 2026-03-09, not archived, 268★ | 141,644 / 33,800 | Apache-2.0 |
| `gleam_pgo` | Gleam (old name of `pog`) | 0.14.0 | 2024-08-15 | `gleam-experiments/pgo` | 27,879 / 2,328 | Apache-2.0 |
| `pgo` | Erlang, pooled | 0.20.0 | 2025-12-12 | last commit 2026-09-10 | 243,111 / 50,679 | Apache-2.0 |
| `epgsql` | Erlang, single connection (no pool) | 4.8.0 | 2025-06-06 | last commit 2026-08-16 | 2.75M / 96k | BSD-3 |
| `postgrex` | Elixir | 0.22.x | ⚠️ | elixir-ecto | 142M | Apache-2.0 |

Sources: hex API, GitHub API (2026-10-21).

`gleam_pgo` is superseded by `pog`. `epgsql` would mean writing our own pool and Gleam bindings. `postgrex` would bring Elixir into the build for no capability `pog` lacks. **`pog` it is.**

### Fit of `pog` against real usage

Read from [`src/pog.gleam`](https://github.com/lpil/pog/blob/main/src/pog.gleam) and [`CHANGELOG.md`](https://github.com/lpil/pog/blob/main/CHANGELOG.md):

- **Pool and roles:** `url_config(name, url)` plus `start` or `supervised` gives a supervised pool under a `process.Name` (v4.0.0). One pool for `dds_app` in the server and a short-lived one for the owner in the migrate step is direct.
- **Transactions:** `pog.transaction(pool, fn(conn) -> Result(t, e))` checks out one connection, runs `begin`, commits on `Ok`, and rolls back on `Error` **or on a crash** (`exception.on_crash`). Since v4.0.0 the pool stays usable inside the callback for queries outside the transaction. This matches every `db.transaction` site. There is no isolation-level argument, but the pilot never sets one; if needed, run `SET TRANSACTION ISOLATION LEVEL …` as the callback's first query.
- **Transaction-scoped advisory locks** (submit lock, Reviewer-accounts lock): these are plain queries inside the transaction (`select pg_advisory_xact_lock($1)`), so there is **no gap**.
- **Session-scoped tick lock: gap.** `pog`'s `checkout` is private (`@external(erlang, "pog_ffi", "checkout") fn checkout`); the only public way to pin one connection is `transaction`. Two options:
  1. **No FFI:** `pog.transaction(pool, fn(lock_conn) { select pg_try_advisory_xact_lock(7215003) via lock_conn; if locked, run the pass on `pool`; Ok(_) })`. The lock connection sits *idle in transaction* for the pass, and the lock is released by commit, or by the server if the BEAM process dies (because the connection drops). Watch `idle_in_transaction_session_timeout`, which defaults to 0 (off).
  2. **About 5 lines of FFI** binding `pgo:checkout/1` and `pgo:checkin/2`, both exported by `pgo` (`pog` itself calls `pgo:checkin` through `@external`), then a session lock and unlock exactly as `tick.ts` does.

  The same choice applies to the test suite's migration lock. On the BEAM, a single supervised scheduler process is a further option if the Gleam app runs as one node. That belongs to the job-mechanism ticket, not this one.
- **Upserts, `returning`, raw SQL:** `pog` is raw SQL, so `on conflict … do update … returning …` is just text.
- **Types:** `Value` constructors exist for `null`, `bool`, `int`, `float`, `text`, `bytea`, `array`, `timestamp`, `calendar_date` and `calendar_time_of_day`. `uuid`, `jsonb`, `inet` and enums go in as `text` with a cast (`$1::uuid`, `$1::inet`, `$1::request_state`). `pgo` 0.15.0 added `timestamptz` with UTC offsets ([pgo CHANGELOG](https://github.com/erleans/pgo/blob/main/CHANGELOG.md)), so `pog.timestamp_decoder()` works on `timestamptz` columns. ⚠️ Not exercised against a live database here.
- **Per-query timeout:** `pog.timeout(query, ms)`, which matters for the Probe and tick timeouts in `scheduler/with-timeout.ts`.

## Typed queries (replacing Drizzle)

| Package | Approach | Latest | Released | Downloads (all / recent) | Licence |
|---|---|---|---|---|---|
| **`squirrel`** | `.sql` files → Gleam functions, types inferred by asking Postgres | 4.7.0 | 2026-06-03 (last commit 2026-09-10, 691★) | 44,275 / 11,391 | Apache-2.0 |
| `parrot` | `.sql` files → Gleam, via the **sqlc** binary (downloaded automatically) | 2.3.0 | 2026-06-04 | 17,293 / 9,868 | Apache-2.0 |
| `cake` | query builder (composable AST → SQL string) | 4.1.0 | 2026-06-06 | 16,855 / 2,498 | MPL-2.0 |

**Squirrel fits best.** The pilot has no dynamic queries, which is the one thing a query builder like `cake` adds. Its migrations are already SQL. Squirrel's generated code calls `pog` directly. `gleam run -m squirrel check` keeps generated code in sync in CI ([README](https://github.com/giacomocavalieri/squirrel/blob/main/README.md)). Parrot works too, but brings in an external Go binary (sqlc) and sqlc's own inference quirks. Its README devotes a section to annotating columns sqlc cannot type.

**Squirrel's type gaps against `schema.ts`.** The supported list ([README "Supported types"](https://github.com/giacomocavalieri/squirrel/blob/main/README.md#supported-types); code: `pg_to_gleam_type` in [`internal/database/postgres.gleam`](https://github.com/giacomocavalieri/squirrel/blob/main/src/squirrel/internal/database/postgres.gleam)) covers bool, text/varchar/citext, int2/4/8, float/numeric, json/jsonb, uuid (`youid`), bytea, date, time, `timestamp`, arrays, and **Postgres enums as Gleam custom types**. Any other type is an error. Specifically:

- **`timestamptz` is unsupported.** Issue [#21](https://github.com/giacomocavalieri/squirrel/issues/21) was closed with *"Squirrel supports the `timestamp` time now and has a nice error message suggesting to use it if `timestampz` type is used."* lpil adds there that it *"would need to be solved in `pgo` and then `pog`"*. Options: cast (`submitted_at at time zone 'UTC'` when selecting, `$1::timestamp at time zone 'UTC'` when passing parameters), or declare the Gleam app's columns `timestamp` and always store UTC. The second is cleaner, and allowed because nothing migrates.
- **`inet` is unsupported.** Issue [#98](https://github.com/giacomocavalieri/squirrel/issues/98) is open as a "type request". It affects 4 columns, all IP-keyed (submitter IP and the download throttles). Cast: `ip::text` when selecting, `$1::text::inet` when writing. Or store `text`: the pilot never uses inet operators (a grep for `<<=`, `>>=` and `::inet` in `apps/api/src/**/*.ts` outside `schema.ts` and specs finds nothing).
- **Codegen needs a live, migrated database.** It reads `DATABASE_URL` or the `PG*` variables. Dev and CI therefore need Postgres up and migrated *before* `gleam run -m squirrel`. The generated `sql.gleam` is committed, so the release build does not.

## Migrations (replacing drizzle-kit)

| Package | Latest | Released | Repo | Downloads (all / recent) | Licence |
|---|---|---|---|---|---|
| **`cigogne`** | 5.0.6 | 2026-04-23 | Billuc/cigogne, last commit 2026-04-23, 48★ | 21,345 / 6,323 | MIT |
| `migrant` | 1.1.0 | 2026-06-14 | aosasona/migrant | 2,793 | SQLite only, so out |
| hand-rolled with `pog` | n/a | n/a | n/a | n/a | n/a |

Read from the 5.0.6 hex tarball and the [README](https://github.com/Billuc/cigogne/blob/main/README.md):

- Migrations are hand-written `priv/migrations/<timestamp>-<name>.sql` files with `--- migration:up`, `--- migration:down` and `--- migration:end` sections. Each one is applied in a `pog.transaction`; `--- migration:up:disable_transaction` opts out, for single-statement migrations only. `apply_migrations` can also apply a batch in one transaction.
- A hash of each file is stored when it is applied, and an edited file is refused afterwards (since v3). This is the same guarantee as drizzle's journal.
- It runs as a CLI (`gleam run -m cigogne all`) **or as a library** (`cigogne.create_engine(config) |> cigogne.apply_all`). The library form makes a **release command** in the BEAM release possible, which matters for the open "Operator CLI on the BEAM" question on the map. Without a `url` in `cigogne.toml` it connects through **`DATABASE_URL`**, the same variable name the pilot uses for the owner role.
- Depends on `pog >= 4.0.0 and < 5.0.0`, so it shares our driver.
- **Gaps:**
  - no schema diffing, which we don't need;
  - **no locking**: `grep -i lock` across `cigogne/src` finds nothing, so two concurrent `apply_all` calls race on `_migrations`. That is harmless for a one-shot host step, but the parallel test suite needs the pilot's trick: our own `pg_advisory_lock(6425001)` around it;
  - seed and grant migrations are plain SQL, so there is no gap there.
- Maturity is moderate. It is a single maintainer with 20 releases since 2024-11, and the latest merge (#53, from CrowdHailer) moved it to stdlib 1.0. The fallback if it stalls is cheap: a 50-line runner on `pog` that reads a folder and a `_migrations` table.

## Object storage (replacing `minio`)

| Package | Kind | Latest | Released | Repo | Downloads (all / recent) | Licence |
|---|---|---|---|---|---|---|
| **`bucket`** | Gleam, sans-IO S3 (builds `Request(BitArray)`, parses `Response`) | 1.5.0 | 2026-07-06 | lpil/bucket | 17,038 / 2,248 | Apache-2.0 (per `gleam.toml`; the GitHub repo shows no licence file) |
| `aws4_request` | Gleam SigV4 signer (used by `bucket`) | 1.2.3 | 2026-06-17 | lpil/aws4_request | 34,565 | ⚠️ |
| `ex_aws` + `ex_aws_s3` | Elixir | 2.7.0 / 2.5.9 | 2026-05-06 / 2025-12-09 | ex-aws | 72M / 60M | MIT |
| `erlcloud` | Erlang, all of AWS | 3.8.3 | 2025-02-21 | erlcloud | 1.3M | BSD-2 ⚠️ |

**`bucket` is the recommended client.** It is sans-IO: each operation module has `request(…) |> build(creds)`, which gives a signed `gleam/http` request, and `response(resp)`. We send the request with `gleam_httpc` 5.0.0. Credentials take `with_port` and `with_scheme` (plain `http` to the compose-internal MinIO or Silo) and `with_region`; MinIO accepts any region. Modules in 1.5.0: `head_bucket`, `create_bucket`, `delete_bucket`, `list_buckets`, `put_object`, `get_object`, `head_object`, `delete_object(s)`, `list_objects`, and multipart (`create_multipart_upload`, `upload_part`, `complete_multipart_upload`).

Fit against the operations the pilot uses:

| Operation | `bucket` 1.5.0 | What we add |
|---|---|---|
| HeadBucket / CreateBucket | ✅ `head_bucket` → `Bool`, `create_bucket` | none |
| **PutBucketLifecycleConfiguration** | ❌ no module | Build it ourselves: `PUT /<bucket>?lifecycle` with the one-rule XML and `Content-MD5` (`gleam_crypto` has MD5), signed with `aws4_request.signer(…) \|> aws4_request.sign_bits`, the same call `bucket/internal.request` makes. ~30 lines. |
| PutObject from memory | ✅ `put_object.request(bucket, key, body: BitArray)` | `content-type: application/zip` header: the builder takes only `bucket`, `key` and `body` and signs with no headers, so we add it to the built request (unsigned, like `Range` below) or sign our own request with `aws4_request` |
| HeadObject with size | ⚠️ `head_object.response` returns `Found` / `NotFound` / `PreconditionFailed`, **without size** | Read `content-length` from the raw `Response` we already hold (sans-IO makes this free) |
| GetObject, whole | ✅ `get_object.response` → `Found(body)` / `NotFound` | none |
| **GetObject, ranged** | ❌ no range option, and `response` maps any status other than 200 or 404 to `UnexpectedResponseError`, **so a 206 is an error** | Add `range: bytes=a-b` to the built request (S3 SigV4 allows unsigned extra headers ⚠️ verify against Silo/MinIO; or add it before signing through `aws4_request` directly), and handle 206 in our own 10-line response function |
| DeleteObject | ✅ | none |

**Streaming:** `bucket` and `gleam_httpc` move whole `BitArray` bodies. That matches the pilot's upload, which is already one in-memory `Buffer`. For downloads, the pilot streams MinIO → Express, while the Gleam app would hold the whole archive (or the requested range) in memory per request. One upstream page of 10,000 rows (ADR 0008) keeps that small. ⚠️ We did not measure archive sizes. If memory ever matters, the app can serve ranged sub-requests in fixed chunks through `mist`'s chunked response, still without any FFI. Wisp's response body is `Text | Bytes | File`.

**Why not the alternatives.** `ex_aws_s3` covers everything, including lifecycle and streaming, but brings Elixir and `hackney`/`sweet_xml` into the Gleam build to save about 100 lines. `erlcloud`'s last release is 2025-02 and it pulls in all of AWS. Presigned URLs, which `bucket` lacks, are **not needed**: the pilot proxies every download (see 004).

The server image question (`minio/minio` is gone; 004 recommends `pgsty/silo`) is separate. `bucket` speaks plain S3 with path-style URLs (`"/" <> bucket <> "/" <> key` in each `build`), which suits MinIO, Silo and Garage alike.

## Open points for the build tickets

- Decide the tick-lock mechanism: an xact lock held on a pinned transaction, a `pgo:checkout` FFI, or a single scheduler process. This belongs to the job-mechanism ticket.
- Decide `timestamptz` (cast) against `timestamp` UTC columns, and `inet` (cast) against `text`, when the Gleam schema is written.
- Pin `pgo` in `manifest.toml` because of the unreleased breaking change on `pgo` `main`.
- CI needs a Postgres service for `squirrel check`.
