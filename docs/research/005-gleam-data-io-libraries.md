# Which BEAM libraries carry the Gleam app's data and I/O

- **Question:** On the BEAM, what replaces the pilot's I/O dependencies? That covers the zip holding one CSV plus the Data dictionary (`yazl`) and reading it back (`yauzl`), a CSV writer that meets spec §8.2's eight writer rules (and does ADR 0009, "the writer is ours", still hold?), an HTTP client for the upstream API with timeout and retry (spec §5, §7.6), SMTP through the relay (`nodemailer`), and time-of-day scheduling (`@nestjs/schedule`) with Asia/Bangkok calendar arithmetic.
- **Issue:** #4 (map #1)
- **Date:** 2026-10-21
- **Primary sources:** the hex.pm package API (`https://hex.pm/api/packages/<name>`) for versions and release dates; package source on GitHub, read at `main`/`master` (`raw.githubusercontent.com/...`); Erlang/OTP's own `lib/stdlib/src/zip.erl`; `gh api` for the latest release and commit dates; and the pilot's code at `33a9491` (`git show 33a9491:apps/api/src/...`). Behaviour marked **[ran]** was checked in an `erlang:28-alpine` container (OTP 28). Every other claim names its source. ⚠️ marks what was not verified.

Toolchain at time of writing: Gleam v1.19.2 (2026-10-12), OTP-29.1.2 (2026-10-14) (`gh api repos/{gleam-lang/gleam,erlang/otp}/releases/latest`).

## Answer

| Concern | Pilot | Recommended on the BEAM | New dependency? |
|---|---|---|---|
| Write the Extract archive | `yazl` | **OTP `zip:create/3` with `[memory]`** | none (stdlib) |
| Read an archive back (fingerprint CLI, tests) | `yauzl` | **OTP `zip:unzip/2` with `[memory]`** | none (stdlib) |
| Extract writer + SHA-256 | hand-rolled + `node:crypto` | **Hand-rolled, as ADR 0009 says**. SHA-256 from `gleam_crypto` 1.6.0 (incremental `new_hasher` → `hash_chunk` → `digest`) or `crypto:hash_*` | `gleam_crypto` |
| Upstream HTTP | `fetch` + own retry | **OTP `httpc`, through our own small FFI module**, with our own retry loop. `gleam_httpc` 5.0.0 is the fallback if we accept its error-handling gaps | none, or `gleam_httpc` |
| JSON envelope | `JSON.parse` | `gleam_json` 3.1.0 + `gleam/dynamic/decode` | `gleam_json` |
| SMTP via relay | `nodemailer` | **`gen_smtp` 1.3.0** (`gen_smtp_client:send_blocking/2` + `mimemail:encode/1`), called through FFI | `gen_smtp` (+ `ranch`) |
| The 60 s tick | `@nestjs/schedule` interval | **No library.** A `gleam_otp` actor that re-arms itself with `process.send_after` | none |
| Asia/Bangkok arithmetic | fixed UTC+7 arithmetic + `Intl` | **`gleam_time` 1.11.0** with a fixed `+07:00` offset; Thai month names and the Buddhist era written as our own table | `gleam_time` |

**ADR 0009 still holds, and on the BEAM it holds more strongly.** `gsv` 5.1.0 is the only maintained Gleam CSV library, and it is a config-driven generator, which is exactly what the ADR rejects. And the language's own `string.trim` trims a different set of characters on Erlang than JavaScript's `trim()` does, which would silently change rule 4's output (see §CSV writer). The writer has to name its whitespace set itself.

**Gaps (nothing exists, so we write it):** the retry/backoff loop (the pilot hand-writes it too), cancelling an in-flight request when a stall is killed, the Thai Buddhist-era date wording (`Intl` `th-TH-u-ca-buddhist` has no BEAM counterpart), and thin FFI wrappers over `httpc` and `gen_smtp`.

**Risks:**
1. **Rule 4's trim set.** Gleam's `string.trim` on Erlang uses `string:trim/2`'s default whitespace. JavaScript strips NBSP (U+00A0) and U+FEFF; Erlang does not **[ran]**. If we port with `string.trim`, an Extract whose cells have NBSP edges gets a different fingerprint from the pilot's. The writer has to use an explicit character set copied from ECMAScript's `WhiteSpace` + `LineTerminator`. The spec never defines "whitespace", so this may be worth a spec ticket.
2. **`gleam_httpc` crashes on error shapes it does not know.** Its FFI handles only `{failed_connect, _}` and `timeout`. Any other `httpc` error (for example, the connection closes after the request was sent) hits `erlang:error({unexpected_httpc_error, Error})`, and the caller process crashes instead of getting an `Error` back. It also cannot tell "dropped after sending" from "never left the host", and the pilot needs that distinction for traffic accounting (§13.6). That is why we recommend our own FFI.
3. **`gen_smtp`'s default TLS options** pin `versions => [tlsv1, tlsv1.1, tlsv1.2]` and set no verification options. The defaults also fall back to plaintext when STARTTLS fails if `tls` is left at `if_available`. We must pass `{tls, always}` (= nodemailer `requireTLS`) and explicit `tls_options` (verify_peer, cacerts, SNI, TLS 1.2/1.3).
4. **Build toolchain.** `gen_smtp` and `ranch` are rebar3 packages, so the build image needs `rebar3` beside `gleam`.

## What the pilot actually uses

Read from `33a9491:apps/api/package.json` and each call site:

- **`yazl`**: `extraction/extract-archive.ts`. `new ZipFile()`, two `addBuffer` calls (the Extract, then `data-dictionary.csv`), the whole output buffered in memory. Default compression and timestamps. The archive is transport and is never fingerprinted (spec §8.4, ADR 0005), so byte-identity with yazl's output is **not** a parity requirement.
- **`yauzl`**: `extraction/read-zip-entries.ts`. `fromBuffer(..., {lazyEntries: true})` reads every entry into memory. Used by the fingerprint command (`cli/verify-extract.ts`) and by specs.
- **CSV**: `extraction/extract-writer.ts`. No library. BOM, CRLF, `raw.trim()`, quote on `/[",\r\n]/` with `""` doubling, `toUpperCase()` on the caller-named positions, and an incremental SHA-256 over the same bytes.
- **HTTP**: `upstream/upstream-client.ts`, `upstream-error.ts`, `upstream.config.ts`. Global `fetch` with `AbortSignal.timeout(60_000)` combined with a caller cancel signal (`AbortSignal.any`). Up to 3 attempts with 1 s/2 s backoff, retrying only `gateway_timeout | server_error | timeout | network | malformed_response`. The timeout also covers reading the body. It reads the `x-request-id` and `x-process-time-ms` headers, never chains or logs a body (spec §14.5), and counts a call (`onCall`) only if it reached upstream, using `cause.code ∈ {UND_ERR_SOCKET, ECONNRESET, EPIPE}` to detect "dropped after sending".
- **`nodemailer`**: `mail/mail-transport.ts`. `createTransport({host, port, secure, requireTLS, auth})`, `sendMail({to, from, subject, html})`, and `info.response`, which is recorded **verbatim** on `mail_sent` (spec §12.4). Retries are ours (Postgres-timed, `scheduler/tick.ts`), not the library's.
- **`@nestjs/schedule`**: `scheduler/scheduler.module.ts`. Used **only** as a registry for one `setInterval` of 60 s (spec §15.3). There is no cron expression and no time-of-day trigger anywhere (`grep Cron|Interval` → that one file). "Due" is a Postgres query, and expiry is derived from instants (§15.1). `scheduler/with-timeout.ts` races MinIO calls against a 30 s timer.
- **Asia/Bangkok**: `clock/business-hours.ts` does plain epoch arithmetic with a fixed `+7 h` offset ("ICT is a fixed UTC+7 with no daylight saving"). `Intl.DateTimeFormat` with `timeZone: 'Asia/Bangkok'` is used for display stamps in `clock/format-ict.ts`, `extraction/archive-naming.ts` and `requests/reference-number.ts`. `i18n/ask-copy.ts` uses `th-TH-u-ca-buddhist` for long-form Thai dates (Buddhist era, Thai month names) and `en-GB` for English.

So the question's "time-of-day scheduling" is really **a fixed 60 s interval plus calendar arithmetic**. No cron library is needed.

## Zip: OTP's built-in `zip`

- `zip:create(Name, [{FileName, Binary}], [memory])` returns `{ok, {Name, Binary}}`. It is documented in `zip.erl`'s `create/3` doc (example: `zip:create(Name, [{"foo", <<"FOO">>}, {"bar", <<"BAR">>}], [memory])`). `zip:unzip(Bin, [memory])` returns `{ok, [{FileName, Binary}]}` (`unzip/2` doc, option `memory`). Source: `erlang/otp` `lib/stdlib/src/zip.erl`.
- DEFLATE and STORE are supported. Files ending in `.Z .zip .zoo .arc .lzh .arj` are stored by default and everything else is deflated (`zip.erl`, `uncompress` option default). Encryption is not supported, and we don't need it.
- **[ran]** Building a two-entry archive (`dds-envocc-sharing-20261021-101500.csv` with a BOM and Thai text, plus `data-dictionary.csv`) in memory gave 428 bytes. `zip:unzip/2` round-tripped the CSV byte-for-byte, and Python's `zipfile -l` listed both entries.
- The entry mtime is the build host's local time, as DOS timestamps are, and the archive also carries extended-timestamp and UID/GID extra fields by default (`zip.erl` module doc). Neither matters, because the archive is not fingerprinted. Pass `{extra, []}` if we want a leaner archive.
- ⚠️ Not tested: opening the archive in Windows Explorer and macOS Archive Utility. This is low risk (standard PKZIP 2.0 DEFLATE), but it is worth a manual check during the slice, since the audience double-clicks.
- **Gleam access:** two `@external(erlang, "zip", ...)` declarations plus a small decoder. No Gleam zip package exists on hex. A search for `zip` returned only Elixir libraries such as `zstream` 0.6.7 and `unzip`, and stdlib makes them unnecessary.

## CSV writer: ADR 0009 holds

- **Library landscape:** `gsv` 5.1.0 (2026-04-18, `giacomocavalieri/gsv`) is the one maintained Gleam CSV package. It is a parser and generator whose line ending and quoting are parameters. That is the "rules as configuration" shape ADR 0009 rejects, and the ADR's reasons (fingerprint stability, 1,952-row ceiling, no type inference) do not depend on the language. **The writer stays ours.** If it is needed at all, `gsv` could serve build-time scripts that read `docs/*.csv`, but nothing that touches the Extract.
- **Mapping each rule to BEAM primitives:**
  1. BOM: literal `<<0xEF,0xBB,0xBF>>`.
  2. CRLF: literal `"\r\n"`. There is no `os.linesep` equivalent to trip over.
  3. and 5. Bare empty cells and minimal quoting: a hand-written check for `,`, `"`, CR, LF, doubling `"`.
  4. **Trim: hand-written, with an explicit set.** Gleam `string.trim` → `string:trim/2` (`gleam-lang/stdlib` `src/gleam/string.gleam`, `erl_trim` external). **[ran]** `string:trim` leaves NBSP (U+00A0) and U+FEFF in place, while ECMAScript `String.prototype.trim` strips both, along with U+1680, U+2000–U+200A, U+202F, U+205F and U+3000. Erlang's default set includes U+0085, U+200E and U+200F, which JS keeps. For parity, port JS's `WhiteSpace` ∪ `LineTerminator` set exactly.
  6. Uppercase: `string.uppercase` → `string:uppercase`. **[ran]** `a150` → `A150`, and `ß` → `SS`, the same as JS `toUpperCase`. The ICD-10 columns are ASCII in practice anyway.
  7. and 8. Header and Data dictionary: static data. The Data dictionary bytes can be embedded at compile time from a generated Gleam module, as the pilot's `data-dictionary.generated.ts` is, or read from `priv/` with `simplifile` 2.7.0.
- **One-pass SHA-256:** `gleam_crypto` 1.6.0 (2026-04-18) exposes `new_hasher`, `hash_chunk` and `digest` (`gleam-lang/crypto` `src/gleam/crypto.gleam`). Raw `crypto:hash_init/1`, `hash_update/2` and `hash_final/1` are the OTP equivalent. **[ran]** `crypto:hash(sha256, _)` works on OTP 28. The NFR-30 tests (same rows twice give one checksum, the file starts with a BOM, and it uses CRLF) carry over as gleeunit tests. ADR 0009's condition is unchanged: the writer is trusted only once those tests exist.

## Upstream HTTP: OTP `httpc` behind our own FFI

- **Candidates (hex.pm):** `gleam_httpc` 5.0.0 (2025-07-12; repo last commit 2026-09-07), `gleam_hackney` 1.4.0 (2026-05-26) over `hackney` 4.8.5 (2026-10-03), and `gleam_http` 4.5.0 (2026-10-09) for the shared request/response types.
- **What `gleam_httpc` gives:** `configure() |> timeout(60_000) |> dispatch(req)`, `follow_redirects` off by default, TLS verification on by default, and `HttpError = InvalidUtf8Response | FailedToConnect(ip4, ip6) | ResponseTimeout` (`gleam-lang/httpc` `src/gleam/httpc.gleam`). httpc's `timeout` covers the whole request including the body, which matches the pilot, where `AbortSignal.timeout` also covers `response.text()`.
- **Why not use it directly:**
  - `gleam_httpc_ffi:normalise_error/1` maps only `{failed_connect, _}` and `timeout`. Every other reason goes to `erlang:error({unexpected_httpc_error, Error})` (`src/gleam_httpc_ffi.erl`), so the worker process crashes. The pilot turns those into the retryable `network` kind.
  - The pilot's traffic count (§13.6) needs "never left the host" (`failed_connect`, which covers DNS failures and refused connections **[ran]**: `{failed_connect, [{to_address, _}, {inet, [inet], econnrefused}]}`) told apart from "dropped after sending" (httpc reasons such as `socket_closed_remotely`). `gleam_httpc` turns the latter into a crash.
  - Spec §7.6 says "killing a stall cancels it: the call in flight is aborted". A synchronous `dispatch` cannot be cancelled except by killing its process. httpc's async mode (`{sync, false}`, then `httpc:cancel_request/1`) can, and is reachable only through our own FFI.
- **Recommendation:** a ~50-line `upstream_httpc_ffi.erl` that calls `httpc:request(get, {Url, Headers}, [{timeout, 60000}, {autoredirect, false}, {ssl, TlsOpts}], [{body_format, binary}, {sync, false}])`, waits with `receive ... after`, maps **every** reason to one of three: `connect_failed` (not counted), `sent_then_dropped` (counted), or `timeout` (counted). It also exposes `cancel/1`. The retry loop, backoff and error classification are ported from `upstream-client.ts` as plain Gleam, with an injectable `sleep` as in the pilot. Rationale for httpc over hackney: it ships with OTP (the `inets` app), adds no dependencies, and the pilot sends one request at a time (concurrency budget 1, §13.2), so hackney's pooling buys nothing.
- **Log discipline (§14.5):** an FFI that returns tagged atoms and never wraps the raw `httpc` reason or body in a crash keeps case data out of crash reports. Gleam's `let assert` on a decode failure would print the value it failed on. So use `case` everywhere in the upstream path.
- ⚠️ TLS defaults: modern OTP's `ssl` client verifies peers by default (OTP 26+ release notes), but this was not re-read here. The FFI should pass `{verify, verify_peer}, {cacerts, public_key:cacerts_get()}` explicitly rather than rely on defaults.

## SMTP: `gen_smtp`

- **Candidates (hex.pm):** `gen_smtp` 1.3.0 (2025-05-30; repo last commit 2026-10-09; ~939k recent downloads; the backend of Elixir's `swoosh`), `mua` 0.2.6 (Elixir, minimal), `lumenmail` 1.1.2 (Gleam, 2026-06-21, first release 2026-01-20, ~590 recent downloads), and `gsmtp` 0.2.0 (Gleam, 2025-01-05, ~111 downloads).
- **`lumenmail` is ruled out for parity:** `smtp.send(client, msg) -> SmtpResult(Nil)` (`davecaos/lumenmail` `src/lumenmail/smtp.gleam`) throws away the relay's reply. The pilot records `info.response` verbatim on `mail_sent` (spec §12.4, `mail-transport.ts`). It is also young and lightly used.
- **`gen_smtp` fits:** `gen_smtp_client:send_blocking({From, [To], Body}, Options)` returns the server's receipt binary, or `{error, Type, Message}` (`src/gen_smtp_client.erl`, `send_blocking/2` spec and doc). Options cover `{relay, Host}`, `{port, P}`, `{ssl, false}`, `{tls, always}` (the nodemailer `requireTLS` equivalent; `if_available` falls back to plaintext on TLS failure: `handle_smtp_throw` for `tls_failed`), `{auth, always}`, `{username, U}`, `{password, P}`, `{tls_options, [...]}`, `{retries, 0}` (our retries live in Postgres), `{no_mx_lookups, true}`, and `{timeout, _}`. `mimemail:encode/1` builds the message, and header values go through `rfc2047_utf8_encode` (`src/mimemail.erl`), so Thai subjects get RFC 2047 encoding.
- **Must override:** `DEFAULT_OPTIONS` sets `{tls_options, [{versions, ['tlsv1','tlsv1.1','tlsv1.2']}]}` and `{retries, 1}` (`gen_smtp_client.erl`). Pass TLS 1.2/1.3, `verify_peer`, cacerts and `server_name_indication` for the relay host, and `retries => 0`.
- Dependencies: `ranch >= 1.8.0`, built with `rebar3` (hex release metadata).
- ⚠️ Not exercised against Mailpit or the real relay. Do that in the mail slice: send one HTML Thai message through Mailpit and assert that the receipt string arrives.

## Scheduling and Asia/Bangkok time

- **The tick:** a `gleam_otp` 1.3.0 (2026-08-09) actor that runs the startup pass, then `process.send_after(self, 60_000, Tick)` after each pass. A pass that crashes is isolated by running it in a supervised child process, which replaces the pilot's try/catch-around-the-pass. `with-timeout.ts` becomes a `process.call` (or a spawned task) with a 30 s timeout. The cross-process "only one pass" guarantee is the Postgres advisory lock, which belongs to the data-layer ticket, not this one.
- **Cron libraries exist but are not needed:** `clockwork` 4.0.0 (2025-08-18) plus `clockwork_schedule` 2.0.0 (2025-10-16, ~237 downloads), `automata` 0.11.0 (2026-09-10, Gleam: cron, RRULE, retries), and `erlcron` 1.3.8 (2026-06-19). Spec §15.3 says "one schedule: there is no second one anywhere", so adding a cron engine would add surface the spec rules out.
- **Calendar arithmetic:** `gleam_time` 1.11.0 (2026-09-19, `gleam-lang/time`) provides `timestamp.system_time`, `add`, `difference`, `to_calendar(ts, offset)`, `from_calendar`, `to_rfc3339(ts, offset)` and `calendar.day_of_week` (`src/gleam/time/timestamp.gleam`, `calendar.gleam`). It deliberately ships **no time-zone database** (its README: "Time zone information has to be loaded from elsewhere"). None is needed: Asia/Bangkok is a fixed `+07:00` with no DST, the same assumption `business-hours.ts` already makes, so the port uses `duration.hours(7)` as the offset. `gleam_time` also compiles to JavaScript, so `shared` can use the same date types in the Lustre client. **[ran]** `calendar:system_time_to_rfc3339(T, [{offset, "+07:00"}])` gives the expected `+07:00` stamp.
- `birl` 2.0.0 and `gtempo` 7.4.0 are richer alternatives. `tzif` 2.0.0 reads system zoneinfo. None is warranted for one fixed offset.
- **Gap: Thai long dates.** The pilot's `th-TH-u-ca-buddhist` `Intl` output (for example, "21 ตุลาคม 2569") has no BEAM library. Write a 12-entry Thai month table plus `year + 543`, and test it against strings captured from the pilot. The English `en-GB` long form needs the same treatment (`calendar.month_to_string` gives English month names).

## Not covered here

- The data layer (Postgres driver, advisory locks), the job mechanism that replaces BullMQ, and object storage (MinIO client) belong to their own tickets on map #1.
- argon2 and TOTP (free choices; map #1 note).
