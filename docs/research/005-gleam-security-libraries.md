# Which Gleam/BEAM libraries carry the Reviewer surface's security

- **Question:** What replaces the pilot's `argon2`, `otpauth` and `qrcode` on the BEAM, and what does `wisp` give us for session cookies, CSRF protection and secure headers? For each: candidate, maturity, and whether it meets spec §10.5 (Sessions) and §17.5 (Reviewer accounts, TOTP enrolment) as the pilot implements them.
- **Issue:** rawinan-soma/dds-sharing-gleam#3 (map #1)
- **Date:** 2026-10-21
- **Primary sources:** the hex.pm package API (`https://hex.pm/api/packages/<name>` and `/releases/<version>` for versions, release dates, download counts and declared dependencies), each package's GitHub repo (README, `gleam.toml`, source and releases via `gh api repos/<owner>/<repo>/...`), wisp's source at tag `v2.2.2` (`src/wisp.gleam`), `gleam_http`'s `src/gleam/http/cookie.gleam`, `gleam_crypto`'s `src/gleam/crypto.gleam`, and the pilot code at `33a9491` (`apps/api/src/reviewer/*`, `apps/api/src/cli/reviewer-cli.ts`). Every claim carries its source. ⚠️ marks what was not verified.

## Answer

| Concern | Use | Notes |
|---|---|---|
| argon2id password hashing | **`argus` 2.0.0** (Gleam, MIT, 2026-08-28) | Reference C implementation through the `jargon` NIF. Runs on dirty CPU schedulers. Defaults to argon2id, and since 2.0.0 always uses a random salt. The build image needs a C compiler. |
| TOTP | **`totally` 2.0.0** (Gleam, Apache-2.0, 2026-02-27), **for code generation and the `otpauth://` URI only**. We write our own `verify_totp` (~40 lines) on top. | `totally`'s own validators check only the current step and compare with `==`. They have no ±1 window, no drift lookback and no constant-time compare. The pilot didn't rely on `otpauth`'s validator either: it wrote its own loop over `totp.generate({timestamp})`, and `totally.set_time` + `totp_from_config` supports the same port. |
| QR for enrolment (terminal, CLI only) | **`qrkit` 0.6.0** (pure Gleam, MIT, 2026-09-29) | Has a half-block terminal renderer, the equivalent of `qrcode`'s `{type:'terminal', small:true}`. It is young (first release 2026-05) and pre-1.0. Fallback: `glqr` 1.2.0. |
| Session cookie | **No session library.** Port the pilot's opaque-token design using `gleam_crypto` + `gleam/http/cookie` directly. | The pilot's session is a 32-byte random token. Postgres stores its SHA-256 hash, and the cookie carries only the token. Nothing is signed or encrypted, so we need no `secret_key_base`-backed session. Wisp has no session store. The only Gleam one (`kv_sessions`) is pinned to `wisp < 2.0`. |
| Cookie attributes | **`gleam/http/cookie.Attributes` + `response.set_cookie`, not `wisp.set_cookie`** | `wisp.set_cookie` hard-codes `HttpOnly` (the CSRF cookie must be readable by JS), base64-encodes or signs the value, and **drops `Secure` on plain-http localhost by itself**. §10.5 forbids that silent degradation. |
| CSRF | **Our own double-submit check (~20 lines, `crypto.secure_compare`)**, optionally layered with `wisp.csrf_known_header_protection` | Wisp only ships an Origin/Referer check. The spec requires a double-submit token. |
| Secure headers | **`wisp.content_security_policy_protection`** (nonce CSP) and/or **`plume` 1.0.1**, plus a hand-set `X-Robots-Tag` | The pilot sets **no** security headers apart from `X-Robots-Tag: noindex, nofollow` on `/reviewer`. Parity needs only that one. Anything more would be new behaviour (a drift ticket under the map's rule), not a library gap. |
| Login throttle | **No library.** Postgres table, as in the pilot. | §10.5/§17.5 require the state to live in Postgres. Every BEAM rate limiter we looked at (`ex_rated`, etc.) keeps it in ETS/memory. |

**Gaps (things no library gives us, so we write and test them ourselves):** the TOTP verifier (window ±1, drift lookback 2, replay guard by last-used step, constant-time compare); the session store (idle 1 h sliding, 6 h ceiling, cap 3 with oldest evicted, deletion on deactivation); the double-submit CSRF guard; the login throttle. All four were already hand-written in the pilot. Only `argon2`, `otpauth` (generation + URI) and `qrcode` were real dependencies.

**Risks:**
1. **NIF build.** `argus` → `jargon` compiles `argon2` C source with rebar3 at build time. The release builder image needs `build-base` / `build-essential` (argus README). This affects whatever release/Docker shape the deployment tickets pick.
2. **Small maintainer bases.** `argus`/`jargon` come from Pevensie (30 / 11 stars). `totally` has 8 stars and 161 recent downloads, `qrkit` 11 stars and was first published 2026-05-14. Every one is a single-maintainer project. We mitigate by using only their small cores (a hash NIF, an HMAC-based code generator, a QR matrix) and pinning versions. `totally`'s generator is easy to replace: ~15 lines over `crypto.hmac`. Mature Elixir alternatives exist (`argon2_elixir` 4.1.3, `nimble_totp` 1.0.0, `eqrcode` 0.2.1), but they pull Elixir into the build.
3. **`wisp.csrf_known_header_protection` behind a proxy.** It compares `Host` against `Origin`/`Referer`. If the tunnel or edge rewrites `Host`, every POST fails with 400. If we adopt it, test it behind the real edge. The spec only requires the double-submit token, so the origin check is optional.

## What the pilot actually does

Read from `33a9491`:

- **Password hashing**: `password-policy.ts` calls `argon2.hash(password, { type: argon2.argon2id })` with node-argon2's default costs, and `argon2.verify(hash, password)`. `reviewer-auth.ts` verifies against a dummy hash when the username is unknown, so response timing does not reveal whether an account exists. Formats need not carry over (map #1: no data migration).
- **TOTP** (`totp.ts`): SHA-1, 6 digits, 30 s, deliberately fixed because Google Authenticator ignores other values. The issuer is `DDS sharing` and the label is the username. The secret is 20 random bytes (`new Secret({size: 20})`), stored base32. `verifyTotp(secret, code, nowMs, lastUsedStep)` rejects anything but `^\d{6}$`, generates codes for offsets −2…+1 **without short-circuiting**, compares with `timingSafeEqual`, rejects matches at or before `lastUsedStep` (replay), accepts offsets −1…+1, and reports `drift` for a match at −1 or −2 (the `totp_clock_drift` field of `login_failed`, §17.5). `otpauth` is used only for `generate({timestamp})`, `Secret` and `toString()` (the URI).
- **Enrolment**: confirmation happens through ordinary sign-in. The first successful login sets `totpConfirmedAt` and writes `totp_enrolled` (`reviewer-auth.ts` `completeSignIn`). The QR is printed **only by the host CLI**: `QRCode.toString(uri, {type: 'terminal', small: true})` plus the raw key for typing in by hand (`reviewer-cli.ts` `printEnrolment`). The browser never renders a QR.
- **Sessions** (`reviewer-sessions.ts`): `randomBytes(32).toString('base64url')` token, `sha256` hash stored in `reviewer_session` with `createdAt`/`lastSeenAt`. `IDLE_TIMEOUT_MS` = 1 h, `CEILING_MS` = 6 h, `MAX_CONCURRENT_SESSIONS` = 3 with the oldest evicted. `resolve(token, {touch})` slides only on user-initiated requests and deletes rows for deactivated or unconfirmed Reviewers.
- **Cookies** (`cookies.ts`): `reviewer_session` is `httpOnly`, `SameSite=Lax`, `path=/`, `secure = config.secureCookies`, `expires` = ceiling. `reviewer_csrf` uses the same attributes but `httpOnly: false`, with 32 random bytes base64url.
- **CSRF** (`csrf.guard.ts`): every non-GET/HEAD/OPTIONS `/reviewer` request needs the `x-csrf-token` header to equal the `reviewer_csrf` cookie, checked with a length check + `timingSafeEqual`. Otherwise it gets 403 `{error: 'csrf'}`.
- **Headers**: no helmet or CSP anywhere in `apps/`, `docker/` or `deploy/` (`git grep` for `helmet|Content-Security|X-Frame|Strict-Transport` at `33a9491`). `noindex.ts` adds only `X-Robots-Tag: noindex, nofollow` for `/reviewer` paths.
- **Throttle**: `login-throttle.ts`, exponential backoff per account and per IP in Postgres. It has no library dependency.

## Candidates

### Password hashing (argon2id)

| Package | Version / date | Language, mechanism | Status |
|---|---|---|---|
| **`argus`** | 2.0.0, 2026-08-28 (1.0.4 2026-04-26, 1.0.3 2025-06-18) | Gleam, via `jargon` ≥ 1.1.0 (rebar3 NIF over the reference C impl) | Active, 0 open issues, 30★ |
| `jargon` | 1.1.0, 2026-04-26 | Erlang NIF | `hash_nif`/`verify_nif` registered `ERL_NIF_DIRTY_JOB_CPU_BOUND` (`c_src/jargon.c` L139–140), so hashing does not block normal schedulers |
| `antigone` | 1.1.0, 2024-04-18 | Gleam wrapper over `argon2_elixir ~> 4.0` | Needs Elixir installed (README). Repo last pushed 2024-06-01 |
| `argon2_elixir` | 4.1.3, 2025-04-27 | Elixir, C NIF | The ecosystem standard (8.8 M downloads), but Elixir |

`argus` API (README): `argus.hasher() |> argus.hash(password)` returns an `encoded_hash` (PHC string). Costs are configurable with `time_cost`/`memory_cost`/`parallelism`, and `argus.verify(encoded_hash, password) -> Result(Bool, _)`. Defaults follow OWASP: argon2id, t=2, m=19 MiB, p=1, 32-byte hash. The v2.0.0 release notes say "Changed password hashing to always use a random salt". **Meets §17.5** (argon2id). The dummy-hash timing defence ports directly.

### TOTP

| Package | Version / date | Notes |
|---|---|---|
| **`totally`** | 2.0.0, 2026-02-27 | Gleam (Erlang + JS). Deps are only `gleam_stdlib`, `gleam_crypto`, `gleam_time`. 8★, 0 open issues |
| `pot` | 1.0.2, 2021-08-07 | Erlang, very widely used (24 M downloads), base32-secret API, no release in 5 years |
| `nimble_totp` | 1.0.0, 2023-03-21 | Elixir (Dashbit) |

`totally` source (`src/totally.gleam`):
- `secret()` = `crypto.strong_random_bytes(20)`, the same 160 bits as the pilot. The secret is a raw `BitArray`, so it can go in a `bytea` column without base32 decoding.
- `new(secret)` defaults to SHA-1, 6 digits, 30 s. `set_time(config, timestamp)` + `totp_from_config(config)` generate the code for any instant. That is the hook a ported `verify_totp` needs to loop over steps −2…+1.
- `otpauth_uri(secret:, issuer:, account:)` emits `otpauth://totp/<issuer>:<account>?secret=…&issuer=…&algorithm=SHA1&digits=6&period=30`, percent-encoding issuer and account.
- **Do not use its validators for §17.5.** `is_valid_from_config` reads `timestamp.system_time()` itself (so it can't be tested against an injected clock), matches **only the current step**, and compares with `==`. It has no ±1 window, no drift, and no constant-time compare. Its replay guard compares a `last_use` timestamp's step against *now's* step, not against the matched step.
- `encode32` is private. For the CLI's "type this key in by hand" line we either read the `secret=` query parameter back out of the URI (`gleam/uri.parse_query`) or write a ~15-line base32 encoder. `gleam_stdlib`'s `bit_array` has base16/base64 but no base32.

**Meets §17.5 once our verifier wraps it.** That verifier is the same shape as the pilot's `verifyTotp`, using `crypto.secure_compare` (`gleam_crypto` 1.6.0, `src/gleam/crypto.gleam` L113).

### QR (terminal, enrolment ceremony)

| Package | Version / date | Notes |
|---|---|---|
| **`qrkit`** | 0.6.0, 2026-09-29 (first release 2026-05-14, 5 releases since) | Pure Gleam, only dep `gleam_stdlib`. Versions 1–40, ECC L/M/Q/H. Renderers `qrkit/render/ascii` (`to_string`, `to_string_compact` half-blocks, `with_inverse`), `svg`, `png`. CI + OpenSSF Scorecard badge. 11★ |
| `glqr` | 1.2.0, 2026-07-19 | Pure Gleam, only dep `gleam_stdlib`. `qr.new(s) \|> qr.generate()` then `qr.print` / `qr.to_printable()`. 14★ |
| `eqrcode`, `qr_code` | 0.2.1 / 3.2.0, both 2025-02-21 | Elixir |

`qrkit.encode(uri) |> ascii.to_string_compact` matches the pilot's `small: true` terminal output. Because only the host CLI renders the QR, a QR bug fails loudly at the seeding ceremony (the phone won't scan), and the printed raw key is the fallback. **Meets the enrolment flow.**

### Sessions and cookies (wisp 2.2.2)

Wisp 2.2.2 (2026-03-27, latest on hex and GitHub releases) has **no session abstraction**. The cookie API in `src/wisp.gleam`:
- `set_cookie(response, request, name, value, security: PlainText | Signed, max_age)` (L1932). It builds `cookie.defaults(scheme)`, which is `path=/`, `HttpOnly`, `SameSite=Lax`, `Secure` iff https (`gleam_http` `cookie.gleam` `defaults`). Wisp treats the scheme as https **unless** the request is plain http to `localhost`/`127.0.0.1`/`[::1]` with no `x-forwarded-proto` (L1941–1948). `PlainText` base64-encodes the value. `Signed` uses `crypto.sign_message` with `secret_key_base` and SHA-512.
- `get_cookie(request, name, security)` (L1985) decodes or verifies to match.
- No encrypted-cookie variant.

Fit against §10.5:
- The pilot's design needs neither signing nor encryption. The cookie is a high-entropy random token and the authority is the hashed row in Postgres, so a `Signed` cookie adds nothing. **Port the token design** using `crypto.strong_random_bytes(32)`, `bit_array.base64_url_encode`, and `crypto.hash(Sha256, _)` for the stored hash.
- **Write the `Set-Cookie` with `gleam/http/response.set_cookie` and explicit `cookie.Attributes`**, with `secure` taken from config. `wisp.set_cookie`'s localhost exception is the "silent degradation" §10.5 rules out (insecure must be opted into by an explicit dev flag). It also forces `HttpOnly` and base64, both wrong for the JS-readable CSRF cookie. `gleam_http` emits `Max-Age` rather than `Expires`. Setting `max_age` to the seconds left until the ceiling is equivalent to the pilot's `expires` = ceiling.
- Idle/ceiling/cap/eviction/deactivation stay in our own Postgres-backed module, as in the pilot. `kv_sessions` 0.1.1 (2025-01-06) and `wisp_kv_sessions` 0.1.0 (2024-12-19) declare `wisp >= 1.2.0 and < 2.0.0` and `gleam_erlang < 1.0.0` (hex release requirements), so they are incompatible with wisp 2.x. They also don't model a non-extendable ceiling.

### CSRF

`wisp.csrf_known_header_protection` (L2056) skips GET/HEAD. It rejects with 400 when `Host` is missing or does not match `Origin` (falling back to `Referer`). When both are absent it **strips the `cookie` header** and continues. Its doc comment says it implements OWASP's "verifying origin with standard headers" and points to token-based mitigation for "more complete" defence. §10.5 requires a **double-submit token**, so we write a ~20-line middleware: on non-GET/HEAD/OPTIONS `/reviewer` requests, compare the `reviewer_csrf` cookie with `x-csrf-token` using `crypto.secure_compare`, and on mismatch return 403 `{"error":"csrf"}`. The origin check may sit in front as defence in depth (see Risk 3). No Gleam CSRF-token package exists on hex (searched `csrf`).

### Secure headers

- `wisp.content_security_policy_protection(fn(nonce) -> Response)` (L2165) sets `script-src 'nonce-…' 'strict-dynamic'; object-src 'none'; base-uri 'none'` with a fresh 24-char nonce per request.
- **`plume`** 1.0.1 (2026-05-24, MIT, deps only `gleam_http >= 4.3.0` + stdlib; 9★, 250 total downloads) is a helmet-style middleware (`use <- plume.middleware(plume.default())`) with typed CSP, HSTS, X-Frame-Options and so on. It is very new and lightly used.
- The pilot sets none of these. Parity is only the `X-Robots-Tag` header, which is one `response.set_header`. Adding CSP/HSTS/frame headers would change behaviour, so under map #1's reference rule it needs a ticket for the repo owner. If accepted, `plume` or wisp's CSP helper covers it.

## Versions checked (hex.pm, 2026-10-21)

| Package | Latest | Released |
|---|---|---|
| wisp | 2.2.2 | 2026-03-27 |
| mist | 6.0.3 | 2026-04-18 |
| gleam_http | 4.5.0 | 2026-10-09 |
| gleam_crypto | 1.6.0 | 2026-04-18 |
| argus | 2.0.0 | 2026-08-28 |
| jargon | 1.1.0 | 2026-04-26 |
| totally | 2.0.0 | 2026-02-27 |
| qrkit | 0.6.0 | 2026-09-29 |
| glqr | 1.2.0 | 2026-07-19 |
| plume | 1.0.1 | 2026-05-24 |

⚠️ Not verified by running code: interoperability of `totally` codes with Google Authenticator (RFC 6238 test vectors were not run), `qrkit` output scanning on a phone, and `argus` compiling inside the eventual release image. The first build slice that touches Reviewer sign-in should cover each with a test: RFC 6238 Appendix B vectors for the verifier, a scan check at the seeding ceremony, and a CI image build.
