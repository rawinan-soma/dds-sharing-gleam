# DDS Sharing — Specification

**Status:** complete for implementation. Version 1.4, 2026-09-08 — §9.3, §10.6,
§10.7, §11.4, §12.4 and §15.2 amended by ADRs 0011–0014, four defects in the Alert
and clock model found by driving the lifecycle in a prototype
(`prototype/request-lifecycle`).
One open item, flagged at the end of §19 and tracked as
[#33](https://github.com/rawinan-soma/dds-sharing/issues/33): the worst-case row
volume rests on a Report code no Disease group can reach.
**Source of authority:** every requirement here was decided on an issue of
[Map: DDS surveillance data-sharing service](https://github.com/rawinan-soma/dds-sharing/issues/1).
Where this document and a closed issue disagree, this document wins and the
disagreement is a bug — report it. Section 19 indexes every requirement back to
the ticket that decided it.

**Vocabulary is not defined here.** [`CONTEXT.md`](../CONTEXT.md) is the glossary
and is canonical. Capitalised terms — Requester, Reviewer, Request, Decision,
Extract, Extract archive, Download token, Attempt, Delivery, Probe, Alert,
Re-run, Snapshot, Disease group, Report code — mean exactly what it says they
mean. Read it first.

---

## 1. What this service is

A web application that lets an officer ask for a de-identified,
case-level extract of Thai DDC (กรมควบคุมโรค) **DDS** occupational- and
environmental-disease surveillance data. Every Request is read and decided by a
named human before any data is fetched. An approved Request produces one CSV,
delivered as a zip through a time-limited link in an email, and destroyed 72
hours later.

Audience: officers at DDC and the regional offices (สคร.), working from
ordinary internet connections, reading Thai, and analysing in Excel, R or Python.

Deployment: a single Docker host inside DDC infrastructure, reachable from the
open internet through a ministry-managed edge.

### 1.1 The eight premises this design rests on

These are load-bearing. An implementer who quietly reverses one of them breaks
something that is not local to the code they are editing.

1. **Surveillance data is never stored at rest.** Upstream responses are fetched
   live per Request, projected in memory, and written only after
   de-identification. PostgreSQL holds Requests, parameters, timestamps and audit
   records — never case rows.
2. **De-identification is a strict allowlist**, and it is the safety control the
   whole service rests on. See §6.
3. **A human reads every Request before any data is fetched.** The gate is the
   second control, and it is an *accountability record, not a lawful basis*
   (§18.1).
4. **The system cannot observe whether an email arrived.** This is a premise, not
   a caveat. See §11 and [ADR 0001](adr/0001-email-delivery-is-unobservable.md).
5. **Rate limiting is a load control on the upstream DDC relationship and on this
   server's disk. It is not a data-protection control.** See §13.
6. **"Data does not linger" is a claim about surveillance data only.** The
   patient-derived Extract is destroyed after 72 hours; the officer's
   telephone number is kept for ever. See §12.7 and
   [ADR 0004](adr/0004-personal-data-is-retained-indefinitely.md).
7. **Everything the machinery does absorbs *upstream's* limits, not ours.** Strip
   away the 365-day cap, the OFFSET timeout cliff and the ~3.5 s fixed request
   cost and the extraction pipeline collapses to a controller method. A later
   reader will otherwise assume the complexity is self-inflicted. ⚠️ **And the
   volumes are tiny** — 3,861 rows a year across the whole domain (§5.3). The
   pipeline is shaped by upstream's *call cost and cliffs*, never by data size.
8. **No PDPA §26 lawful basis and no DDC sign-off are on record**, by decision,
   not by oversight. See §18.

### 1.2 Non-goals

Named here so nobody adds them back without reopening the decision.

- **Mechanical identity verification** — email-domain rules, staff-directory
  matching, a `workplace` picklist. The check is human judgement; a mechanical
  layer would be false comfort, not defence in depth.
- **Reviewer editing of a Request before approval.** Approve or reject only; an
  editable Request breaks the audit chain.
- **Live job progress for the Requester.** The flow is fully asynchronous:
  submit → confirmation page → close the tab → email.
- **Ingesting or storing surveillance data at rest.**
- **A gated case-level or tambon-granularity service for outbreak
  investigation.** That is a different service with real authentication and a
  lawful basis per request.
- **Scraping resistance, per-client quotas, or any row cap.**
- **A fake upstream service as a product deliverable.** It is needed as an
  *implementers' dev harness* (§17.3) and is not part of this specification's
  destination.

---

## 2. The workflow

```
Requester fills the form ──► reaches the queue at once, row count pending
   │
   ├─ Probe runs behind, off the submit path (§5.4) ─► probe_performed
   │     (~35 s; nothing waits on it — approve is NOT blocked)
   │     └─ calls exhaust 3 attempts ──────────────► probe_failed
   │           (count reads "failed"; the job starts later with the
   │            disk pre-check skipped, and never waits for ever)
   │
   ▼
Request: pending ──── 24 business hours elapse ──► expired          (terminal)
   │
   ├── Reviewer rejects ────────────────────────► rejected          (terminal)
   │      └─ rejection email, no reason given
   │
   └── Reviewer approves
          │
          ▼
       Extraction: queued → running → ready
          │                    │
          │                    └─ failure ─────► failed  ─► Alert (§14.3)
          │                                          └─ Re-run button (§10.6)
          ▼
       Delivery email carries a Download token (72 h)
          │
          ├── Requester collects ──────────────► collected          (terminal)
          │
          └── no Attempt in 24 business hours ─► collection lapse Alert (§11.4)
                 └── no Attempt by 72 h ───────► expired_uncollected (terminal)
```

**Wall-clock from submit to Extract is hours or days, not minutes.** The pipeline
takes **single-digit minutes at worst** (§7.9) against a human gate measured in
business hours. It was never the dominant term and now is not a term at all.
**Optimising the pipeline cannot improve what a Requester experiences** — weigh
every future proposal there against that sentence.

---

## 3. Access model

### 3.1 Reach

**Internet-facing.** สคร. staff work from ordinary internet connections, not a
ministry VPN, so a network boundary would lock out the intended audience to buy
a control that audience cannot use.

- The **Requester surface is unauthenticated**. Contact fields are an audit
  record, never a credential, and are never verified by the system.
- The **Reviewer surface is authenticated** and rides the same app and the same
  public ingress at `/reviewer`, with no network restriction. Reviewers may need
  to decide from outside the DDC network, and Reviewer unavailability converts
  directly into expired Requests.

### 3.2 What the approval gate is and is not

The gate supplies **a named human accountable for each individual release**. It
does **not** supply a PDPA lawful basis, and the specification says so in §18
precisely because the gate *feels* like it closes the compliance hole, and that
feeling is how the sign-off never gets chased.

The gate raised the de-identification standard's context from *safe to publish*
(the pre-gate, anonymous-scraping threat model) back to **safe to release to a
vetted, named recipient**. It is an **additional** control, not a licence to
widen the allowlist — widening now would spend the safety margin the gate
bought. Widening was offered and declined.

### 3.3 The identity check is human and unaided

The system **presents** the fields and **records** the Decision. It does not
judge, score, or match. A Reviewer reads five contact fields, one of which
(`workplace`) is free text nobody validates, and forms a judgement.

---

## 4. Requests

### 4.1 The parameter surface

Exactly three parameters. Everything else a Requester might have controlled is
deliberately off the surface.

| Parameter | Cardinality | Rules |
|---|---|---|
| **Disease group** | exactly one, required | Chosen from a picker, never typed. A **named family of one or more Report codes** (§4.9), not a raw code. The picker shows the Thai family name; the Requester never sees `201`. |
| **Date range** | one inclusive `from`/`to`, required | Maximum span **365 days**, **rejected, never split** (§4.2). |
| **Area selection** | optional; when set, exactly one | Empty means national. Otherwise **one province** *or* **one health region**. Not a province plus a region, not two regions. |

**Not on the surface, and named here so nobody adds them later:** pagination,
page size, date chunking, column choice, output format, current-address
filtering, any row cap, any date floor.

Three deliberate absences, each with a reason:

- **No row count shown to the Requester at submit.** Two reasons now. Upstream's
  count ignores the area filter, so a Requester filtering to one province could
  be shown a number 70× what they will receive — a wrong number is worse than
  none. And since §5.4 the count does not *exist* at submit: the Probe runs off
  the submit path. *The count is shown to the Reviewer* — same number, different
  audience, different meaning. Do not "fix" the inconsistency.
- **No zero-row gate.** A header-only CSV is a true answer, and a Disease group
  returning zero rows for a quarter is a normal outcome for this audience.
- **No date floor.** Usable history appears to start in 2025 for the one Report
  code profiled, but bounding the picker would hardcode one code's sample as if it
  held for all 25.

### 4.2 The 365-day cap

Upstream enforces it (`HTTP 400`, "Date range must not exceed 1 year (365
days)"). It is a **span** limit, `end - start <= 365 days`, not an absolute floor.

Surfaced, not hidden. Enforced in **two places**: the date picker greys out any
`to` beyond `from + 364 days` (365 days counted inclusively, as its foot says;
amended 2026-10-01, #123, to `docs/design/system.md` "Date field"), and the
server re-checks the cap itself on submit as a guard against direct API calls.
The picker is one day stricter than the cap: a typed `from + 365` is accepted. **The server's message names the cap as upstream's**,
because when a Requester asks why, "the DDC API caps it" is the true answer.

Splitting a wider Request server-side stays rejected, but **its original reason
is gone** — that reason was the Probe running synchronously at submit, and §5.4
moved it off that path. What remains is simpler and sufficient: upstream refuses
the span, the cap is therefore upstream's to explain, and a split Request is one
ask the Reviewer would have to judge as several. Someone wanting 2025-to-date
submits two Requests.

### 4.3 Dates are inclusive to the human

The picker presents `1 Jan – 31 Jan` and the Requester receives 1–31 Jan.

**Upstream's interval is half-open, `[start, end)` — `end_date` is exclusive.**
The API client adds **one day to `end_date` on the wire, and nowhere else**. The
`+1` never appears in the UI, the CSV, the audit record, or the stored Request.
Without it, upstream silently returns 1–30 Jan, and a single-day Request returns
zero rows.

### 4.4 Area selection

**The filter matches `epidem_chw_code`. It never matches `chw_code`.**

It is **the province that reported the case**. It is neither the patient's
registered address (`chw_code`) nor the treating unit's (`isolate_chw_code`,
dropped from the Extract).

> **Amended 2026-10-05 (#96).** On 2026-09-18 this section followed upstream's
> dictionary, *ที่อยู่ รหัสจังหวัด ขณะสำรวจว่าเป็นโรค* (the province the patient was
> living in when the case was surveyed), and declined the reporting-province
> reading. The repo owner reversed that in the Thai copy pass: the copy, the
> glossary and this section now say *reported*. The filter column is unchanged.

Both columns ship in the Extract and both use the same province codes, so a
filter written against the wrong one produces a plausible, well-formed,
**silently wrong** Extract that no gate in this system would catch. This is the
question a สคร. is asking — *"cases reported in my provinces"*, not *"cases
among my registered residents"* — and for EnvOcc groups it matters, because
workers are frequently reported far from where they are registered.

- Filtering is a **post-fetch row predicate**, applied client-side. It does not
  reduce upstream cost: a provincial Request costs the queue exactly what a
  national one does.
- **A region is expanded server-side into its province list before the Request is
  stored.** The stored Request names provinces and never a region, so an old
  Request still means what it meant even if a region boundary is redrawn, and the
  audit record stays truthful without versioning a lookup table.
- **Province selection is one at a time.** This is a UI simplicity choice, not a
  cost one — one province and thirty cost identically.
- `epidem_chw_code` is mandatory in DDS reporting, so the null case does not
  arise. **Count rows where it is absent and raise an operational alert if the
  count is non-zero** rather than silently dropping them: "this cannot happen" is
  exactly the assumption worth instrumenting.

### 4.5 Region vocabulary

**Health region (`เขตสุขภาพ`), 13 regions**, from `docs/provinces.csv`'s
`health_region` column. That is MoPH's health-region geography, and it is what
the spec and the picker use.

Two other 13-way vocabularies exist and neither is this one: สช.'s
`เขตสุขภาพเพื่อประชาชน` (same groupings, different institution) and **สคร.**,
DDC's own regional disease control offices.

> **The UI does not state this.** An earlier version of this section required
> the copy to warn that a สคร. officer reading "เขต 8" might assume it meant their
> office's catchment. Removed by the repo owner 2026-09-18: the page names the
> vocabulary (เขตสุขภาพ) and shows the provinces a region expands to, which is the
> check a reader needs. This section, not the UI, is where the three vocabularies
> are distinguished.

### 4.6 Geography codes

`docs/provinces.csv` (77 rows: `province_id`, `name_th`, `health_region`),
`docs/districts.csv` (929) and `docs/sub_districts.csv` (7,451) are the reference
data. Verified properties an implementer may rely on:

- Province codes occupy **10–96**, uniformly two digits. **There is no leading-zero
  case to normalise** at the province level.
- The hierarchy is **strictly prefix-nested at 2 / 4 / 6 digits**:
  `amp_code[:2] == chw_code`, `tmb_code[:4] == amp_code`. Verified with zero
  orphans and zero prefix violations across all districts and subdistricts.
- Therefore **a province filter is a prefix test, not a join.**
- `chw_code` and `epidem_chw_code` draw on the same code domain. **Upstream sends
  them as JSON numbers (`10`, not `"10"`)** — confirmed 2026-09-04 — so
  **normalise to string before comparing**. A plain cast is sufficient and no
  padding is needed, **because of the bullet above**: the domain starts at `10`,
  so no geography code can lose a leading zero in transit. Verified against the
  reference data — zero codes beginning `0` across all 77 provinces, 929
  districts and 7,451 subdistricts.
- **`group_code` is likewise a bare integer** (`201`, not `"201"`) — confirmed
  2026-09-04. It is sent, never parsed back from a response: the extraction job
  knows which Report code it asked for.

> **`group_code` does not populate `epidem_report_group_id`** — confirmed with
> กองระบาดวิทยา, 2026-09-04, closing an inference drawn from p.18 of the source
> deck. **Nothing in this design depended on it**, and that is not luck: the
> pipeline makes **one call per Report code** (§7.2), so the job already knows
> which code a row belongs to from the request it made. **Do not read the Report
> code back out of a response field.** `epidem_report_group_id` is upstream's own
> grouping, it means something else, and it is not in the allowlist.

### 4.7 Contact fields

Five fields, all free text, none validated, none verified: **name, surname, tel,
email, workplace**.

`workplace` has no picklist. You cannot enumerate every hospital, university and
provincial office that might legitimately ask, and a picklist with an "Other" box
proves nothing. It is an input to human judgement, never a credential.

### 4.8 Duplicate suppression

Reject a submit whose **email and ask** match an unfinished Request: the same
email address (compared trimmed and case-insensitive), the same Disease group, the
same `from` and `to`, and the same area as stored: national, or the same
province list after any region is expanded (so region 13 and the province
กรุงเทพมหานคร are the same ask, which they are).

This catches the page refresh and the double-posted form, which re-send exactly
the same form. Narrowed from an IP key on 2026-09-30 by the repo owner: a สคร.
office is one IP, so the IP key held a whole office behind one person's pending
Request, and it stopped a Requester resubmitting with a corrected email. Matching
on the email is **not** verifying it: the address is still free text that nobody
checks, and this rule reads it only to recognise the same form sent twice. It does
**not** catch an adversary, who changes one character for free. It belongs to the UX section of this
spec, not the security section — naming it a "limit" is how it gets miscounted as
a control. It survives the approval gate unchanged: the gate replaces it as a
*volume* control but not as a double-click guard.

---

### 4.9 Disease groups and Report codes

**A Report code is upstream's unit; a Disease group is the Requester's.** There
are **25 Report codes today** — `201`–`224` plus `501` (Heat Stroke, โรคลมแดด) —
seeded from `docs/research/003-disease-group-codes.md`. **Confirmed 2026-09-04:
the seed stays embedded.** Where a list is supplied it covers **EnvOcc diseases
only**, which is this service's scope anyway (§5.3) — so a supplied list changes
how the seed is *maintained*, never how it is *read at runtime*. **Do not fetch
the classification at boot or per Request.** **The set is neither 24-valued nor contiguous**, and code must
never assume it is: `501` sits far outside the EnvOcc block and arrived after the
first 24 were written down. A **Disease group** is a named family of one or more
of them, classified by DDC's own officers. See ADR 0006.

**The classification is `docs/disease-groups.md`** — ten groups over the 25 Report
codes, seeded from there and from nowhere else.

> ⚠️ **The 25 codes are this service's *scope*, not upstream's whole domain.**
> Probing on 2026-09-02 ([#33](https://github.com/rawinan-soma/dds-sharing/issues/33))
> established that the same endpoint also serves the general D506
> notifiable-disease block — `01` cholera, `02` acute diarrhoea, `03` food
> poisoning, `07`–`09` typhoid, `301`–`303` tuberculosis, `401` animal bite, `502`
> snakebite, `601` hepatitis B, and more nobody has enumerated. **This service
> serves the EnvOcc block only** (`201`–`224` plus `501`), by decision. A
> communicable-disease code appearing in `docs/disease-groups.md` is a scope
> change, not a classification fix. Each group has a **stable id**;
the name and the code list may be revised, the id may not, because stored Requests
and Decision Snapshots reference it.

- The classification is a **partition**: every Report code sits in exactly one
  Disease group, none is left out, and a code that belongs alone is a group of
  one. A code in no
  group is data nobody can ask for; a code in two makes *which Extract did this
  case land in* unanswerable.
- It is **ours, not upstream's** — an editorial act inside a de-identified
  release. It is therefore **published in the Data dictionary** shipped in every
  Extract archive (§8.2 rule 8), never held only as a lookup table in code.
- It is **amendable, and demonstrably so**. 216–224 were added by announcement in
  ธ.ค. 2567 and `501` after that; the list will change again, so the group **is expanded at submit and the expansion
  is stored** (§12.3) — the same treatment a health region gets in §4.4. A
  Re-run refetches the codes the first run fetched, not the codes the group means
  today.
- **The merge is a plain union.** No ICD-10 predicate narrows it: a filter would
  put this service in the business of deciding what counts as the disease, and a
  row it dropped would be indistinguishable from a case never reported.
  `diagnosis_icd10` rides on every row for anyone who wants to narrow afterwards.
- **No de-duplication, because none is possible.** One case carries exactly one
  Report code, so the union is disjoint by construction.
- **No width cap at runtime.** A Request is never rejected for spanning many
  codes; a rejection the Requester could only satisfy by shortening their dates
  is a bad conversation. If a family is too wide to serve, that is the
  classification's problem at design time. **`โรคจากสารกำจัดศัตรูพืช` is the widest
  group — ten Report codes** (`209`–`218`). Its cost is **calls, not rows**:
  reported pesticide volumes in DDS are low, but each Probe call costs ~3.5 s
  whatever comes back, so group width alone sets the floor — **ten calls, ~35 s**
  (§5.4). That is what keeps the Probe asynchronous.
- **Adding a Report code upstream is two edits, not one** — the code list in
  `docs/research/003-disease-group-codes.md` and a group for it in
  `docs/disease-groups.md`. A code in no group is unreachable data and **nothing
  in the system will notice.** §17.1's partition test does **not** close this: it
  checks the classification against the seed, so a code that exists upstream and
  is missing from the *seed* is invisible to it. The control is the periodic
  human re-probe in §17.1, not a build.

---

## 5. The upstream API

`GET https://exchange.ddc.moph.go.th/api/d506/v1/disease-groups`,
`Authorization: Bearer <token>`. One token for the whole service.

**The published field dictionary is unreliable.** Four documented errors: it
describes `page` as a page count (it is a 1-based index), omits the `page_size`
minimum of 20, omits the 365-day range cap, claims `lab_report_result` is
`negative`/`positive` when it is Thai prose, and its own example date range
returns no data. Verified behaviour below supersedes it.

### 5.1 Envelope

```json
{ "status": true, "message": "...", "data": [ /* rows */ ],
  "meta": { "page": 1, "page_size": 100, "total_items": 1458,
            "total_pages": 15, "has_next": true, "has_previous": false } }
```

`meta.total_items` and `meta.total_pages` are known **from the first response**.
The loop terminator is `meta.has_next === false`; the last page is short, and
requesting past the end returns `200` with `data: []`.

Response headers to capture: **`x-request-id`** (quote to DDC support; the one
field here with no substitute) and `x-process-time-ms`.

### 5.2 Parameters and their traps

- `page_size`: **minimum 20** (undocumented), maximum 10,000. Never emit below 20.
- `page`: 1-based index. `page=0` → `422`.
- **An unknown `group_code` returns `200` with `data: []`**, not a `404`. A
  typo'd or stale code is indistinguishable from "no cases this period". This
  bites harder since §4.9: a Disease group expands to several codes, and one
  mistyped member of the family goes missing in silence inside an otherwise
  plausible Extract.
- **Unknown query parameters are silently ignored.** A deliberately bogus
  parameter returned `200` with an unchanged response. So **send only known-good
  parameter names, and assert `meta` echoes what was asked** — a mistyped
  parameter name yields a cheerful `200` with unfiltered data rather than an error.
- **There is no field-projection parameter.** Twelve candidate names were tried;
  all returned identical full rows. De-identification is therefore **ours,
  post-fetch**: plaintext identifiers transit the extractor on every Request
  (never to disk). The filter and project stages are load-bearing, not
  conveniences.

### 5.3 Cost, and why the pipeline looks the way it does

| page_size | rows | server time |
|---|---|---|
| 20 | 20 | 3,487 ms |
| 100 | 100 | 3,499 ms |
| 1,000 | 1,000 | 3,617 ms |
| 10,000 | 1,458 | 3,659 ms |

**~3.5 s fixed cost per request, near-independent of rows returned.** Request
*count* is the cost driver — always use `page_size=10000`.

Cost also climbs with page depth — pagination is `OFFSET`-based:

| page | server time |
|---|---|
| 1 | 6,001 ms |
| 10 | 11,183 ms |
| 20 | 17,102 ms |
| 50 | 33,904 ms |
| 100 | **HTTP 504 after 60 s** |

**There is a ~60 s gateway timeout, so pages past roughly 50 are unreachable.**
⚠️ **Nothing this service can be asked for comes near that cliff, and no
machinery guards it.** It was established against Report code `02` at 1,141,658
rows a year = 115 pages — *"not slow, impossible"*. `02` is **not in this
service's domain**: it is acute diarrhoea in the general D506 notifiable-disease
block, and this service serves the 25 EnvOcc codes only (§4.9,
[#33](https://github.com/rawinan-soma/dds-sharing/issues/33)).

**Every Disease group's entire year fits in a single page.** The largest Request
anyone can submit is 1,952 rows against a 10,000-row page, so the cliff sits ~50×
beyond the widest thing this service serves. It is handled by one loud failure
(§7.2) and by nothing else. **Do not reintroduce date-chunking, adaptive sizing,
or any other apparatus to stay clear of it**, and do not let a later reader infer
a load that has never existed
([ADR 0008](adr/0008-the-pipeline-is-sized-for-one-page.md)).

**Concurrency buys nothing.** Eight concurrent requests all returned `200` but
degraded from ~3.9 s to ~14.3 s each — the upstream serializes. Extraction stays
sequential and the global concurrency budget is 1 (§13.2). Record *why* in any
config comment, so a later operator does not tune it upward expecting throughput.

**Annual row volumes for sizing — all 25 in-scope Report codes**, measured
2026-09-02 over 2025-08-27 → 2026-08-27
([#33](https://github.com/rawinan-soma/dds-sharing/issues/33)):

| Disease group | Report codes | rows/yr |
|---|---|---|
| `air-pollution` | 201 | **1,952** |
| `work-related` | 220 | 970 |
| `heat` | 501 | 401 |
| `environmental-pollution` | 221 | 194 |
| `silicosis` | 202, 203 | 129 |
| `lead` | 208 | 95 |
| `asbestos` | 204–207 | 73 |
| `pesticides` | 209–218 | 28 |
| `confined-space` | 219 | 15 |
| `radiation` | 222–224 | 4 |

**The entire in-scope domain is 3,861 rows a year.** The worst case a Requester
can express — a full-year national `air-pollution` Extract — is **1,952 rows**,
returned in a single `page_size=10000` call. Codes 212, 215, 217 and 224 returned
**zero** over the full year; a Disease group legitimately extracting to nothing is
the normal case here, not a fault (§7.7).

**Out-of-scope volumes, recorded only so nobody re-imports them as sizing.** The
same endpoint serves the general D506 notifiable-disease domain — `02` acute
diarrhoea 1,141,443 · `03` food poisoning 146,559 · `401` animal bite 87,281 ·
`601` hepatitis B 51,517 · `301` TB 12,059. **None of these is reachable through
this service.**

### 5.4 The Probe

**One `page_size=20` upstream call per Report code, over the Request's whole
span**, purely to read exact `meta.total_items`. The Request's total is the sum
across the Disease group's codes.

**Why per code, over the whole span.** A Request's span is capped at 365 days
(§4.2) and upstream's own cap is 365 days, so **the whole span is always one legal
call**. This now mirrors §7.2's fetch exactly — same span, same one-call-per-code
shape — which is the point: the Probe and the run ask upstream the same question,
and the only difference is `page_size`.

An earlier design probed per (code, month) pair, turning the widest group's
full-year Request into ~130 calls — ~7.6 minutes holding the single upstream slot
(§13.2), queued ahead of every extraction job, and spent as often on the reject
path as anywhere. It bought one thing: a submit-time check that no month exceeded
the ~50-page cliff. §7.2 has since removed both the months and the machinery that
watched them.

**Its date range comes from the shared span builder**, the single function that
turns a Request into the half-open range `[from, to + 1 day)` — with the
conversion to upstream's inclusive `end_date` in the API client and nowhere else
(§4.3, §7.2). The extraction job calls the same function for the same Request.
*The Probe must not know how to build a date range*, and neither may the job hold
a second copy of the rule: §7.2's ⚠️ records that two expressions of one range is
exactly how the 3,196-row loss happened. One function, two callers, no way to
disagree.

**It runs off the synchronous submit path.** 35 s is still too long for a form.
Submit returns immediately, the queue item appears at once with its row count
**pending**, and the Probe fills it in behind.

- It fetches **no data for the Extract**.

> ⚠️ **A Reviewer MAY approve before the count lands. Do not "fix" this.** An
> earlier draft disabled approve until it did, on the stated ground that the count
> was "the proportionality signal the gate exists for". **That ground is false.**
> The Reviewer's gate is about **who is asking** — identity, Workplace,
> legitimacy — not about how much they ask for. A Request that needs hours
> completes in hours; **long runtime is not a reason to reject**, and §13.3
> already refuses to gate anything on size for the same reason. Blocking
> approve made a real person wait for a number they are not permitted to act on.

**There is therefore no Probe stalled Alert.** It existed only to rescue a
Request that could not be approved without its count (§10.6). With approve
unblocked a wedged Probe strands nobody: the count stays *pending*, the Decision
proceeds, and correctness rests on the run-time totals (§7.5), which the Probe
never fed.

**Nothing waits on the Probe.** Since [ADR 0008](adr/0008-the-pipeline-is-sized-for-one-page.md)
the disk pre-check is a fixed free-space floor rather than a projection from the
row count (§7.8), so an approved job starts immediately whether the count has
landed or not. The Probe gates neither the *human* nor the *job*.

**It still needs a bounded end, and it has one.** Each call retries on the same
discipline as a fetch (§7.6) — **3 attempts, exponential backoff, 60 s
per-request timeout**. When a code's calls exhaust their attempts the whole Probe
is **abandoned**, recorded as `probe_failed` (§12.4), and the count is displayed
as **failed** rather than *pending*. The only thing lost with it is the zero-row
catch for that Request; the Extract is produced regardless, and correctness rests
on the run-time totals (§7.5), which the Probe never fed.

The count's uses are all informational:

- **The zero-row catch.** §5.2's cheerful-empty-`200` makes a typo'd or stale
  Report code indistinguishable from "no cases this period", and since §4.9 one
  silent member of a ten-code family vanishes inside an otherwise plausible
  Extract. This is the **only** reason the Probe stays pre-Decision rather than
  folding into the job after approval: folding it in would spend nothing on
  rejected Requests, but a Requester would then wait hours to be told their codes
  matched nothing. At 10 calls the reject-path waste is trivial.
- **Accountability.** Recorded as `probe_performed` (§12.4), once per Request,
  carrying the **per-code** counts. A rejected or expired Request spends real
  upstream calls, and the approval gate makes the reject path common — without
  this event that traffic exists in no record anywhere.

**The Probe's total and the run's fetched total will differ, and that is
legitimate.** Hours to days pass between them — the approval gate guarantees it —
and upstream keeps receiving reports for past dates. The difference is
**recorded on `job_completed`, never asserted**: failing on it would fail
correct jobs. It is kept because it is the only witness that would ever show the
Probe's range builder and the job's had diverged, which §7.5 cannot see — that
assert compares a run against itself.

### 5.5 Failure taxonomy

| Condition | Status | Body |
|---|---|---|
| bad token | `401` | `{status, message: "Token invalid"}` |
| `page_size` out of bounds | `422` | `{status, message: "Validation Error", errors:[{field, message}]}` |
| malformed date | `422` | `errors[].field = "start_date"` |
| `end_date` < `start_date` | `422` | `{status, message}` |
| range > 365 days | `400` | `{status, message: "Date range must not exceed 1 year (365 days)"}` |
| `page` far beyond `total_pages` | `400` | `{status, message: "Page too large"}` |
| deep page under load | `504` | gateway timeout at 60 s |

**Two error-body shapes** — `{status, message}` and
`{status, message, errors[]}`. The client must handle both.

No rate-limit headers are advertised and no `429` was ever observed. The failure
mode to fear is **not a throttle** — it is DDC noticing our traffic and revoking
the token, which no retry recovers from.

---

## 6. De-identification

This is the load-bearing safety control. Everything else in the system assumes
its output.

### 6.1 The six rules

Stated as rules, not as field verdicts, so that an upstream schema change cannot
slip a field through.

1. **Strict allowlist.** A field reaches the Extract only by appearing in §6.2.
   Not on the list ⇒ not in the Extract. An **unknown field name** ⇒ operational
   alert (raised in `project`, §7.4). An **absent** field is normal and must
   never alert (§6.5).
2. **No free text, ever.** Coded and delimited-code fields only.
3. **No sub-district geography, and no point coordinates.**
4. **`epidem_`-prefixed twins follow their originals exactly.** Asymmetry is what
   leaks. The twin set is **closed and exhaustively verified**: `address`, `moo`,
   `road`, `chw_code`, `amp_code`, `tmb_code`, plus `epidem_report_guid`,
   `epidem_report_group_code`, `epidem_person_status_id`,
   `epidem_symptom_type_id`. (`epidem_person_status_id` is not a twin — there is
   no `person_status_id`.)
5. **No small-cell suppression.** The Extract is a case-level line list;
   suppression means dropping rows, which breaks the completeness invariant
   (§7.5).
6. **A derived column may read only fields that are themselves kept.** Adding a
   derived column is therefore an **allowlist change, reviewed as one, never a
   pipeline change.** Without this, a computation over a dropped field smuggles
   its information past the allowlist — a "distance from home to treating
   hospital" column derived from the dropped GPS pairs would disclose exactly
   what dropping them prevented.

**Rule 6 binds immediately and it costs something.** Where `birth_date` is null,
`onset_age` is blank, even though upstream ships a populated `age_y`. Falling
back to that field is precisely what rule 6 forbids. The blank is accepted; see
the dev-cycle ask in §17.2.

### 6.2 The Extract's 23 columns

**21 upstream passthrough + 2 derived.** Fixed, in this order, always — see §7.4.

| # | Column | Class | Source |
|---|---|---|---|
| 1 | `epidem_report_guid` | record identity | upstream |
| 2 | `epidem_report_group_code` | disease | upstream |
| 3 | `diagnosis_icd10` | disease | upstream |
| 4 | `diagnosis_icd10_list` | disease | upstream |
| 5 | `birth_date` | person | upstream |
| 6 | `gender` | person | upstream |
| 7 | `prefix` | person | upstream |
| 8 | `nationality` | person | upstream |
| 9 | `occupation` | person | upstream |
| 10 | `marital_status_id` | person | upstream |
| 11 | `chw_code` | geography (registered) | upstream |
| 12 | `amp_code` | geography (registered) | upstream |
| 13 | `epidem_chw_code` | geography (reporting) | upstream |
| 14 | `epidem_health_zone` | geography (reporting) | **derived** from 13 |
| 15 | `epidem_amp_code` | geography (reporting) | upstream |
| 16 | `hospital_code` | facility | upstream |
| 17 | `onset_date` | dates | upstream |
| 18 | `onset_age` | person | **derived** from 17 + 5 |
| 19 | `treated_date` | dates | upstream |
| 20 | `diagnosis_date` | dates | upstream |
| 21 | `death_date` | dates | upstream |
| 22 | `report_datetime` | dates | upstream |
| 23 | `update_datetime` | dates | upstream |

**Column 2 is the Report code the row was fetched under**, admitted on #30. It
was previously dropped as redundant — true only while one Request meant one code.
With a Disease group spanning several (§4.9), it is the only thing telling merged
rows apart, and it discloses nothing: it is what the Requester asked for.

**Ordering rule:** each derived column sits **immediately after its input**, so a
reader scanning the header sees each computed column beside what produced it.
That is why `epidem_health_zone` follows `epidem_chw_code` and `onset_age`
follows `onset_date`.

Finest **named** geography is the district, `amp_code`. See §18.2 for why
"finest named" is not the same as "finest effective".

### 6.3 The derived columns

**`onset_age`** — the case's age in completed years **at `onset_date`**.

Anchoring it on the case rather than on the Request's submission date is a
deliberate reversal of an earlier decision. A Request spans up to 365 days and
the approval gate can add days more, so a submission-anchored age reports an
eleven-month-old case a year older than they were, and reports **the same case
differently in two different Requests**. Anchoring on the case makes two Extracts
comparable and appendable. Recorded as
[ADR 0002](adr/0002-derived-extract-columns-anchored-to-the-case.md).

**`epidem_health_zone`** — the health region of `epidem_chw_code`, via
`docs/provinces.csv`.

**Deliberately not called `health_zone`.** Upstream ships a `health_zone` column
and it means something else: measured over 4,632 rows it agrees with the region
of `isolate_chw_code` on **99.8%** of rows, against 92.7% for `epidem_chw_code`.
It is the *treating unit's* region. Shipping it unchanged would put a column in
the Extract whose meaning silently disagreed with the area filter — a Requester
asking for เขต 8 would receive rows stamped with other regions and no way to see
why. The `epidem_` prefix reuses rule 4's convention to say which address it came
from.

**Absent, malformed and impossible inputs all emit an empty cell.** Never a
sentinel (`-1` / `UNKNOWN` is read as data by an R loader), never a dropped row
(it breaks the completeness assert), never a clamp to 0 (it invents a newborn).
This covers `onset_date` before `birth_date`, a future `birth_date`, and an age
over 120. **Impossible values are counted per job and the count goes in the
record**: three is bad source data, four hundred thousand means the derivation is
broken, and those must be distinguishable.

**One exception, and it is not a blank.** A null `epidem_chw_code` is a gap in the
source; a code that is **not one of the 77** means *our table is stale*. That
raises the scheduler banner (§15.3) rather than silently blanking a column a
regional analyst is about to group by.

### 6.4 The province lookup

A **Postgres table**, seeded by a checked-in migration generated from
`docs/provinces.csv`, which stays canonical. The application role is **read-only**
on it, so a production database disagreeing with the repo is a detectable bug
rather than silent drift.

- **Startup asserts 77 rows and a checksum, and fails fast.** Treat this as a
  boot failure, never a warning: a half-applied seed would blank
  `epidem_health_zone` for every row of a 25-minute job.
- **The job reads the table once at start and holds it.** Not for speed — for
  consistency. A per-row join would let a mid-job edit put two different regions
  for one province inside one Extract, and the fingerprint would then attest to a
  file no single state of the database ever produced.
- **The table's checksum joins the job-completion event** (§12.4). Correct one
  province's `health_region` and an identical Request yields a different
  `csv_sha256`; without the checksum the permanent record shows two hashes for
  one ask and nothing explaining why.

### 6.5 What is dropped, and why

Direct identifiers: `cid`, `first_name`, `last_name`, `mobile_phone` (all arrive
**already encrypted** upstream, apparently salted per record — so `cid` is *not*
a stable person key and repeat-patient detection is impossible from this feed;
they are dropped because there is nothing to gain by shipping ciphertext), and
`passport_no` (**plaintext**, up to 7.9% populated in group `02`).

Sub-district geography and point location: `moo`, `road`, `address`, `tmb_code`
and their `epidem_` twins; `location_gis_latitude/longitude`,
`cluster_latitude/longitude`. **The GPS pairs were dropped on grounds of what
they would carry if populated, not on being empty — and they turned out to be
populated in groups `02` and `03`.** That judgement is confirmed correct rather
than lucky.

Free text (rule 2): `cdeath`, `active_case_finding`, `lab_his_ref_name`, and
**`lab_report_result`** — reversed from an earlier KEEP verdict once its contents
were read: 831 characters in group `02`, 88.2% of values carrying non-code
characters, 29.4% containing Thai script. It is prose, not the
`negative`/`positive` the dictionary promises. Free text is where names leak, and
an 831-character Thai lab narrative is the single most likely place in this
response for a patient's name to appear in plaintext.

Upstream internal keys: `id`, `lab_his_ref_code`.

All clinical fields: `complication`, `organism`, `epidem_person_status_id`,
`epidem_symptom_type_id`, `patient_type`, `vaccinated_status`,
`respirator_status`, `tmlt_code`, `status`, `lab_report_date`. The Extract is
*who, where, when, and what diagnosis*. `death_date` survives as the only outcome
signal; `diagnosis_icd10` and `diagnosis_icd10_list` survive as *disease*, not
clinical.

Superseded by derivation: upstream `age_y`/`age_m`/`age_d` (age to the day, which
reconstructs `birth_date` exactly) and upstream `health_zone` (§6.3).

Redundant: `treated_hospital_code` — measured **identical to `hospital_code` on
99.6–100% of rows** across four groups; it was never a second facility.
`hospital_name` (leaves the Extract with an opaque code; see §6.7).
`isolate_chw_code`, `municipal`, `generation_datetime`. (`epidem_report_group_code`
was on this list until #30 and is now column 2 — see §6.2.)

### 6.6 There is no fixed upstream schema

Per-group key counts run **56–62**; the union across groups is **63**. Absent
keys are *omitted*, not nulled. Worse, the count varies **within** one group by
date range: group `201` returned 50 keys over January and 56 over January–March.

**Therefore the Extract's columns are the allowlist, fixed, with absent fields
written empty — never the observed response keys.** Deriving columns from the
response would make the column set a function of the date range the Requester
happened to ask for, and would give two Report codes of one Request different
columns.
This is the third instance of the same failure shape this design keeps meeting:
well-formed, plausible, silently wrong.

### 6.7 `hospital_code` ships raw, and the reader resolves it elsewhere

Not coarsened, not replaced, not conditionally suppressed. No facility reference
list joins this repo. `facility_type`, an opaque `facility_key`, and row-level
suppression of รพ.สต. codes were each offered and declined — all three require
the MoPH register as a pipeline dependency.

Two facts decided it:

1. **The 5-digit code carries no structure.** It is a running number
   `00001`–`89999`, assigned once and never changed — not when the facility moves,
   not when it is reclassified. Only `99XXX` / `77XXX` are structural, marking
   branch units. **So coarsening by truncation does not exist.**
2. **The register is published openly** by MoPH at
   <https://hcode.moph.go.th/> and as open government data. Withholding a lookup
   protects nothing: anyone holding an Extract can already resolve every code in
   it.

`hospital_code` has exactly one audience — **whoever opens the Extract**. The
Reviewer never sees case rows and there is no facility parameter, so the "a human
making a judgement needs Thai names" argument does not apply here.

**The spec states the resolution route plainly** rather than leaving a bare
5-digit code with nothing said about it: the reader resolves it at
<https://hcode.moph.go.th/>. See §18.2 for the risk this retains.

---

## 7. The extraction pipeline

Four stages: **fetch → filter → project → write.**

Each stage answers a different question — fetch: *which rows exist upstream*;
filter: *which rows*; project: *which columns and what is in them*; write: *what
bytes*. Keeping them separate is not tidiness: it is where rule 6 lives.

### 7.1 Invariant — raw upstream data never lands anywhere

Each page's rows are projected in memory and appended to that Report code's output
before the next page is fetched. **Raw responses are never persisted — not to the
scratch volume, not to logs, not to the audit table.** Only post-allowlist output
ever touches disk.

### 7.2 Fetch: one call per Report code, over the Request's whole span

**One upstream call per Report code in the Disease group** (§4.9), over the
Request's entire date span, at `page_size=10000`. Codes are walked in ascending
order — that order is the Extract's row order (§8.2) and must not be varied for
throughput, which §5.3 shows there is none to gain.

The span comes from the **shared span builder** — `[from, to + 1 day)`, half-open,
derivable from the Request alone with no upstream call. §5.4's Probe calls the
same function, so the Probe and the run cannot disagree about which days they
covered.

- **The span is always one legal call.** A Request is capped at 365 days (§4.2)
  and so is upstream, so the range never needs splitting to be accepted.
- **The pagination loop stays**, and it is not headroom — it is the correct way to
  read a paged endpoint. Walk `while page <= meta.total_pages`. At today's volumes
  it executes once for every Disease group; that is an observation about the data,
  never an assumption the code may make.
- **There is no date-chunking.** Every group's whole year fits in one page (§5.3).
  Monthly tiling was inherited from a design sized against out-of-scope code `02`,
  and its only surviving effect was cost: it turned `pesticides` into **120 calls
  to fetch 28 rows**.
- **Adaptive chunk sizing off `total_items` is rejected**, as it was before, and
  now for a simpler reason: there is nothing to tune.
- If a Report code ever exceeds ~50 pages over its span, that is an
  upstream-volume event that **must fail loudly** — a `504` the retry cannot clear
  (§7.6), then a failed job and an extraction-failure Alert (§10.6). The remedy is
  a human act against the classification, not machinery.

> ⚠️ **Half-open arithmetic lives in the span builder and the API client, and
> nowhere else** (§4.3). Upstream's `end_date` is exclusive; the human's `to` is
> inclusive. The 3,196-row loss in the record came from a second copy of that
> conversion disagreeing with the first. One expression, every caller.

### 7.3 Filter

A post-fetch row predicate on **`epidem_chw_code`** against the Request's stored
province list. Prefix comparison, string-normalised (§4.6). See §4.4 for why this
column and not `chw_code`.

### 7.4 Project

Owns the **fixed 23-column set**, the **fixed column order**, and **both
derivations**.

- Emits exactly those 23 columns in that order; a missing upstream key becomes an
  empty cell.
- Computes `onset_age` and `epidem_health_zone` per §6.3.
- **Raises rule 1's unknown-field alert.** Project already compares observed
  upstream keys against the allowlist; the unknown-*name* check is that same
  comparison read the other way. This keeps rule 1 living in one place.
- **The writer is never given column semantics.** Folding projection into filter,
  or pushing it into the writer, was rejected: give the writer column semantics
  and rule 6 is what gets violated later.

### 7.5 Completeness

**Asserted on rows *received*, per Report code, against that code's
`meta.total_items`.** Never on rows *written* — the area filter legitimately
changes that number, so a rows-written assert would fire on every filtered
Request. Across the code joins, the final CSV's line count must equal the sum of
per-code rows written.

**This assert is not sized for anything and never was.** It costs one integer
comparison and it is the only thing standing between a truncated fetch and a CSV
that looks complete. It stays whatever the volumes do.

**On mismatch: fail the job and publish nothing.** No partial Extract, no link.
Both counts go to the audit record. A truncated CSV that looks complete is worse
than an error — that judgement is the reason this pipeline exists at all instead
of synchronous streaming.

### 7.6 Retry, resume and stall

**The Report code is the atomic unit** — the (code, chunk) pair was, until §7.2
stopped chunking. A 504 is expected, not exceptional.

- A failed code retries **from page 1**, **3 attempts, exponential backoff**.
- Each completed code persists on the scratch volume as a **checkpoint**, so a job
  resuming after a worker restart redoes at most one code. For a one-code group
  that is the whole job — which costs ~3.5 s.
- **No mid-code resume.** Restarting at page 7 assumes the OFFSET window has not
  shifted; a partial walk plus a fresh tail is how a quietly-wrong file ships.
  This survives the loop running once today: the rule is about what happens the
  day it doesn't.
- On retry, the code's `total_items` is compared against the previous attempt.
  **If it moved, the code is discarded and restarted.**
- The job fails only when a code exhausts its attempts.
- **Per-request timeout is 60 s**, matching the gateway.
- **Stall detection, not a duration cap: the job fails if no code completes for
  2 minutes.** The widest group is ten calls at ~3.5 s (§7.9), so 2 minutes is
  already ~30× the expected gap between completions. The former 15 minutes was
  sized against a job believed to run *tens* of minutes; against a 35-second job
  it is not a detector, it is a delay. Lack of progress is the fault signal;
  duration is a legitimate variable, and a hard ceiling stays rejected as dead
  code.
- A stall is killed **automatically, not flagged for a human**, because a stalled
  job holds the single upstream slot and blocks every Request behind it. Waiting
  costs more than being wrong.
- **Killing a stall cancels it.** The call in flight is aborted and no retry
  follows, so a failed job never goes on calling upstream beside the next one,
  and every call it made is counted before `job_failed` is written (§13.6).

### 7.7 Queue mechanics

**BullMQ executes; PostgreSQL is the system of record.**

- A job row is written to Postgres at approval; the BullMQ job carries a
  **reference**, never the authoritative state.
- **Redis runs with AOF persistence.** Redis is not durable by default, and a
  restart on the wrong persistence config drops queued jobs while Postgres still
  says those Requests exist — leaving a Requester waiting for ever on a job in no
  queue.
- **On worker startup, reconcile**: any Postgres job in `queued`/`running` with no
  live BullMQ job is re-enqueued or failed. Without this step, "durable jobs" is
  one config file away from being false. **The reconcile never touches
  `pending`** — an unapproved Request has no work, and its clock is derived
  (§15.1).
- **Global extraction concurrency is 1** (§13.2).

### 7.8 Storage during the job

Each code's output is written to a **local scratch volume**; the finished zip
uploads to MinIO in **one** operation at the end.

This makes *"an object exists in the bucket"* mean exactly *"a complete,
publishable Extract"* — the invariant the Download token and the lifecycle rule
both rest on. A partial upload a bug could hand a link to is the silent-truncation
failure this design already ruled worse than an error.

**Scratch is deleted immediately on successful upload**, code checkpoints
included. Holding it to token expiry would double the disk bound. **Exactly one
copy exists after completion.**

**A job refuses to start when free disk is below a fixed floor of 1 GB** (§14.2).
It is a floor, **not a projection from the Probe's row count**. An archive is tens
of KB (§13.5) and the entire domain is 3,861 rows a year, so a projection would be
arithmetic on a number that is always effectively zero — and it cost far more than
it measured: it was the last thing coupling a job to the Probe, and it dragged a
queue wait, an abandonment path and a *"do not fix this"* warning behind it
([ADR 0008](adr/0008-the-pipeline-is-sized-for-one-page.md)).

The floor still earns its place, because it is not really about the Extract: the
volume also carries PostgreSQL, logs and container images (§13.5). Below 1 GB the
right move is to wait loudly rather than run and die at the upload step.

**An approved job therefore starts immediately**, whether or not the Probe's count
has landed.

### 7.9 Worst case

**The worst case is a full-year national Extract of the widest Disease group, and
it is one call per Report code.** Cost is *call count* alone — §5.3's ~3.5 s fixed
cost, near-independent of rows returned.

| | codes | calls | time | rows |
|---|---|---|---|---|
| **Widest by calls: `pesticides`** | 10 | 10 | **~35 s** | 28 |
| **Widest by rows: `air-pollution`** | 1 | 1 | **~3.5 s** | 1,952 |

So the ceiling is **well under a minute, bought entirely by a group's *width* in
Report codes and not at all by data volume.** A ten-code group costs ten times a
one-code group to serve the smallest Extract in the catalogue — which is why
`pesticides` is called out in `docs/disease-groups.md` as the group to split if
any group ever is.

⚠️ **Two former figures are withdrawn.** *"Full-year group `02`, ~115 shallow
pages, roughly 10–25 minutes"* rested on an out-of-scope code
([#33](https://github.com/rawinan-soma/dds-sharing/issues/33)). *"120 calls ≈ 7
minutes"* was real but self-inflicted — it was monthly chunking's cost, not
upstream's, and §7.2 has removed it. Every *"10–25 minute"* and *"7 minute"*
reference elsewhere in this document inherits both corrections.

**Oversized Requests are never refused.** There is no size gate. This survives
unchanged — it was never a bet on the worst case being small (ADR 0007), and there
is now no volume gate anywhere in the system for it to be the exception to.

---

## 8. The Extract and the Extract archive

### 8.1 One flat CSV, zipped

**One Request = one CSV.** The header is emitted once across the code joins.

**Row order is fetch order: Report code ascending, then upstream's own order
within a code** (§7.2).
So a Disease group of two codes yields all of the first code's rows, then all of
the second's — the file reads as what it is. The Extract fingerprint is a hash of
the bytes as written (§8.4) and must be reproducible, and any date-interleaved
order would mean sorting a million rows that arrived already grouped.

**The zip's original justification — transfer size — no longer holds.** It was
*"~150–200 MB of CSV compresses to roughly 20–30 MB"*, sized on out-of-scope code
`02`. The real worst case is **1,952 rows ≈ 400 KB, compressing to tens of KB**
(§5.3). Downloading that over ordinary สคร. internet is not a problem anyone has.

**The zip stays anyway, on a different and now-primary reason:** the archive
carries **two** files — the Extract and the Data dictionary (§8.2 rule 8) — and a
container is the only way to deliver a CSV alongside the document that explains
its columns. Recorded explicitly because the size argument is the one a later
reader will remember, and it is the one that died.

> ✅ **Excel truncation is no longer a risk — this entry is retired.** It read: *"a
> full-year group `02` Extract is ~1.14 M rows … Excel loads the first 1,048,576
> and reports no error."* `02` is out of scope
> ([#33](https://github.com/rawinan-soma/dds-sharing/issues/33)). The largest
> Extract this service can produce is **1,952 rows — 0.19% of Excel's ceiling**,
> and the whole 25-code domain over a full year is 3,861 rows. Excel opens every
> Extract this service will ever generate, intact. See §18.6.

### 8.2 The eight writer rules

The writer receives a fixed, ordered 23-column row and **carries no column
semantics**.

| # | Rule |
|---|---|
| 1 | **UTF-8 with BOM** |
| 2 | **CRLF line endings, fixed — never `os.linesep`** |
| 3 | Empty cell written **bare**: `a,,c`, never `a,"",c` |
| 4 | **Trim** leading and trailing whitespace on every value |
| 5 | **RFC 4180 minimal quoting** — quote only on `,`, `"`, CR or LF |
| 6 | **Uppercase-normalise `diagnosis_icd10` and `diagnosis_icd10_list`, and nothing else** |
| 7 | **English upstream field names** in the header row |
| 8 | A static **Thai/English Data dictionary** ships in every Extract archive |

**Rule 1.** The only way Excel opens Thai correctly on a double-click, and that is
the audience. The 3-byte BOM is *inside* the file and therefore inside the
fingerprint (§8.4). `pandas.read_csv` without `encoding='utf-8-sig'` yields a
first column literally named `﻿epidem_report_guid` — **that note goes in the Data
dictionary**, because the R/Python reader is exactly the person who will not have
hit it before.

**Rule 2.** Fixed regardless of the container's host OS. A writer emitting
`os.linesep` produces a different fingerprint for the same rows depending on
where it ran, which would make the checksum attest to the machine as much as to
the data. **This is a required test, not a comment** (§17.1).

**Rules 3 + 4.** Project collapses absent-key and `""` into "empty cell"; the
writer emits it bare. **The trim is the load-bearing half** — without it a
whitespace-only upstream value survives to the file and there are *three* null
representations again. Trimming is safe only because every one of the 23 retained
columns is a code, a date or an id; none has meaningful edge whitespace. Had
`address` survived the allowlist, this answer would have been different.

**Rule 6.** ICD-10 is canonically uppercase, so `a150` is upstream data entry
rather than a distinct code — and both `A150` and `a150` were observed in group
`301`. Pass-through means an analyst's `group_by(diagnosis_icd10)` silently splits
one disease into two: a wrong *analysis*, not an ugly file. Applied to
`diagnosis_icd10_list` too, same code system. **Applied to nothing else,
deliberately** — a blanket `.upper()` is how a writer starts inventing data, and
normalisation that changes meaning is an allowlist-grade decision, not a writer
convenience.

**Rule 7.** Thai headers break `df.amp_code` in pandas and `$amp_code` in R. The
stronger reason is naming discipline: the English names are what the allowlist,
this specification, and any column list shown to a privacy officer all use. Three
documents saying the same word about the same column.

**Rule 8.** The Data dictionary answers the concern rule 7 creates — the header is
the only documentation travelling with the file. A **static Thai/English CSV — 23
column rows plus the Disease group classification — checked into the repo and
copied into every archive under a fixed filename**. It carries the classification
because that taxonomy is **ours, not upstream's** (§4.9): a Requester who asked
for ซิลิโคสิส must be able to read which Report codes we took that to mean,
without asking us. It is a property of the service, never of the Request. Free per
job.

### 8.3 Archive naming

`dds-envocc-sharing-{YYYYMMDD}-{HHMMSS}.zip`, **Asia/Bangkok**, from the
Request's **submit** moment. The CSV inside shares the stem with `.csv`; the Data
dictionary has a fixed name.

> ⚠️ **A Re-run carries a `-r2`, `-r3` suffix.** A Re-run makes no new Request and
> no new submit moment, so without a suffix it would produce a second archive —
> different rows, different fingerprint, different Download token — **under the
> same filename, silently replacing the first on the Requester's disk.**
> Anchoring on job completion instead was available and declined.

The filename holds **no reference number**, consistent with §8.4's no-marker rule.
It is recorded on the completion event because it is the string a person reads out
on the telephone.

### 8.4 The Extract fingerprint

**A SHA-256 of the Extract — the CSV as written, before the zip step — computed
in one pass as the writer emits.**

Recorded as
[ADR 0005](adr/0005-the-fingerprint-covers-the-extract-not-the-archive.md).

Why not the archive: a zip embeds a modification time per entry and varies with
compressor version and level, so **the same rows would hash differently on every
run** — and rule 8 added a second, constant entry to the archive. A deterministic
zip was available and declined: it recovers byte-for-byte attestation of the
container only by making reproducibility a standing constraint on a library
nobody will remember it applies to, one dependency upgrade from the silent
failure the fingerprint exists to remove. Two fields were declined too — they
invite an incident reader to check the wrong one.

> ⚠️ **Accepted: we cannot attest the delivered Extract archive byte-for-byte,
> only the Extract inside it.** The archive is transport.

> ⚠️ **The fingerprint attests content, never provenance.** A content hash cannot
> say *which* Request produced an Extract: two Requesters asking the same question
> of the same date range receive identical bytes, and every header-only Extract
> hashes alike. **A match narrows to a *set* of Requests, never to one.** In
> practice the reader also has the date range, the disease group, the column set
> and `token_lookup` to narrow with. **No identifying mark is added to the
> Extract** — a marker would put an identifier back into a file this design spent
> four decisions removing them from, and would vary the bytes it was meant to
> certify.

> ⚠️ **No checksum covers the upload to MinIO.** A re-read before the upload would
> not cover it either; the choice was between two passes and one, not between
> checked and unchecked.

**The fingerprint never leaves the record.** Not in the Delivery email, not on the
download page. The Requester has nothing to compare it against, and publishing it
converts a byte-exactness property into a promise made to an unauthenticated
recipient — a promise §8.2's rules are one library upgrade from breaking.
Interrupted transfers are covered by range requests (§16.2) instead.

**The verification path is a command on the Docker host, not a written
procedure.** It takes a file — an Extract archive or a bare CSV, whichever
arrived — and reports which Request it came from, or reports no match. Same route
as the other host commands. A procedure performed by hand is how a wrong answer
gets made under pressure.

> ⚠️ **The command prints the asymmetry, so nobody has to remember it: a match is
> strong evidence; a mismatch is nearly none.** This audience opens CSVs in Excel,
> and a file opened and re-saved will not match although the data is unchanged.
> Without that line, a mismatch reads as *"this did not come from us"* — the wrong
> answer to the only question the checksum exists for.

---

## 9. Delivery and retention of the Extract

### 9.1 The channel

**Email carries a Download token, never the file.** Size is the obvious reason;
the stronger one is that an attachment has no expiry and no revocation, while the
whole delivery design rests on a bounded lifetime.

The Delivery email points at **`GET /d/<token>` on NestJS**, not at an Angular
route (§16.1). **It renders a small server-side page, and the archive is a second
request, `GET /d/<token>/archive`, made when a person presses the button**
([ADR 0018](adr/0018-the-delivery-link-opens-a-page-and-the-archive-is-its-own-request.md)).
The page view is a lookup, audited but **not an Attempt**; the archive request
counts the Attempt, checks the token, and streams with range-request support. A
dead token redirects to `/link-expired` from either route.

*Until 2026-09-18 the emailed URL streamed the archive directly. That let any
mail-security scanner that opened the link spend an Attempt and receive the
case-level archive before the Requester ever clicked.*

> ⚠️ **`/d/<token>` travels in email, so a live Download token outlives any
> redeployment that moves it. Treat the path as fixed.**

> ⚠️ **Consequence, and it is load-bearing: an Extract stays collectable even if a
> front-end asset fails to load.** Any design that put collection behind the
> Angular bundle would let a completed extraction become unreachable inside its
> 72 hours because of a static asset.

### 9.2 The token

- **Unguessable**, time-limited, **attempt-capped — not single-use.** Single-use
  does not defend against the actual threat (a leaked or forwarded link) and makes
  the outcome worse: whoever opens it first wins, so a leak becomes a lockout of
  the legitimate Requester *on top of* the disclosure.
- **Expiry: 72 hours from job completion, never extended by download, never
  extended by a same-address resend.**
- **An Attempt is one presentation of the token to the archive route**
  (`/d/<token>/archive`), counted at presentation, not at completed transfer —
  counting completed transfers would not bind an attacker who aborts at byte 1.
  Opening the page at `/d/<token>` is audited as a lookup and is not an Attempt
  (ADR 0018).
- **Cap: 10 over the whole 72 h, no rolling window.** Deliberately loose. The cap
  cannot stop a leaked link (one successful download is the entire disclosure);
  its only job is bounding how long a link that reached somewhere public stays
  useful, and 72 h already does most of that. Counting presentations does bite a
  legitimate Requester whose ~30 MB transfer drops, which is why the number is 10
  and not 3.
- **Every presentation is audited — timestamp, IP, user agent, success or
  failure.** *The audit trail is the control; the cap is a backstop.*
- **Failed token lookups throttle per-IP at 20/hour, then a 1-hour block.
  Successful downloads never throttle** — retrying an interrupted transfer must
  always work.

### 9.3 The 72-hour clock

**Anchored on job completion.** The clock's job is bounding **how long the data
sits at rest**; that is a retention property, and anchoring it on anything the
Requester observes would make retention hostage to their attention.

> ⚠️ **Recorded knowingly: job completion is an event the Requester never sees**,
> and it may follow their submit by days. The whole 72 h can elapse unnoticed if
> the Delivery lands in junk. **The remedy is the collection-lapse Alert and a
> telephone call inside the window — not a longer clock.** That remedy works only
> because the lapse trip-wire runs on this same wall clock (§11.4). On the business
> clock it ran on until 2026-09-08, the Alert arrived at or after this expiry, and
> the promise in this sentence was empty.

72 h covers a Friday-evening completion read on Monday.

### 9.4 The expiry page

**One page, one sentence, plus the contact telephone number. Always identical.**

Four states render it: expired token, exhausted attempts, deleted object, and **a
token that never existed**. They are distinguished in the audit record and
nowhere else.

> ⚠️ **No reference number on this page.** An earlier decision put one there so the
> three *known* states would read alike — but a token that never existed has no
> Request and therefore no reference number, and showing the number where we have
> it would tell someone walking the token space **which guesses landed.** The
> per-IP throttle blunts that; the principle forbids it. This costs the legitimate
> Requester nothing: they already hold the reference number, because the Decision
> email carries it.

**No prefill and no resubmit link.** A "resubmit this Request" link in the
Delivery email would be a *second* unauthenticated capability, exposing the
Requester's own contact fields and outliving the Download token it rides beside.
A re-run by the Requester means **resubmit and be reviewed again**, never
*download again*.

### 9.5 Deletion

**Two independent mechanisms, and the application owns the one on the record.**

1. **A scheduled application job deletes the object at token expiry** and writes a
   deletion record: actor, object key, timestamp, outcome. A **startup reconcile**
   sweeps objects whose tokens expired while the application was down.
2. **A MinIO lifecycle rule is the backstop only** — for when that job is broken
   or the box was down. The tick applies it, retrying every pass until it takes;
   until then the pass withholds its heartbeat, so a backstop that is not in
   place reaches the banner rather than a boot-time log line.

A lifecycle rule deletes *silently*. The application would hold a token row
asserting an Extract exists when the object is already gone, and hold no record
that the deletion happened. For a system whose safety argument is that the data
does not linger, "we configured a lifecycle rule" is a weaker claim than it
sounds. **The evidence is the application's deletion record, not the bucket's
configuration.**

> ⚠️ **The bucket lifecycle is also 72 h, so the invariant *lifecycle ≥ token
> expiry* holds by equality with no slack.** That is safe **only** because
> S3/MinIO lifecycle expiration is evaluated in whole days on a periodic scan and
> therefore fires **at or after** the boundary, never early — so a live token can
> never point at an object the backstop removed ahead of schedule. **If the
> lifecycle rule is ever expressed in a unit finer than days, or measured from
> anything other than object creation, that guarantee is gone and the rule must go
> back above the token expiry.** This reasoning belongs next to the number in any
> config, because "72 and 72" looks like a tidy coincidence.

**An object still present 1 hour past its token expiry is a scheduler-class
fault** and raises the banner and `/health` signal (§15.3). Never a silent skip.

---

## 10. The Reviewer surface

Route: **`/reviewer`** — not `/admin`. `CONTEXT.md` rules that word out for a
Reviewer: it conflates the person who approves a data release with the person who
runs the server, and those carry different risk. Secondary benefit: `/admin` is a
constant target of automated scanning, and both the sign-in throttle and the
Reviewer event record would otherwise carry that noise.

**Not linked from the public app, `noindex`, and no security is claimed for the
URL.** Reviewers arrive by bookmark. This is tidiness — keeping a privileged login
out of search results and out of a curious Requester's path. **Say so in the
spec**, so nobody later treats the path as a secret worth protecting.

### 10.1 The queue

**A split queue: list on the left, detail on the right.** Browsing and picking is
kept. A no-list, one-at-a-time hand-off and an approve-gated-on-expanding-identity
variant were both built and rejected as friction that buys engagement it cannot
verify.

**The queue does not auto-refresh** (§10.5).

### 10.2 The review screen

Shows, and only shows:

- The **five contact fields** (name, surname, tel, email, workplace).
- The **Request parameters in human terms** — Disease group *name*, inclusive
  dates, area *name*. Never codes. The Report codes the group expanded to sit
  **beneath the name**, available but not the headline: the name is what is being
  judged, the expansion is what makes the Decision legible years later (§12.3).
- The **Probe row count** — or **"pending"** while the Probe is still running, or
  **"failed"** if it was abandoned (§5.4) — as a **single summed number**. A
  Decision waits on none of the three (§5.4):
  size is not a ground for rejection. The per-code breakdown is on
  `probe_performed` (§12.4) and is deliberately not the headline; the sum is the
  whole informational signal, and ten numbers on a screen read as something to
  judge.
- **Submit time and time remaining** on the business-hours clock, shown so a
  Reviewer feels the clock without the queue reading as an alarm.
- **How many Requests are ahead of this one**, and nothing more precise (§13.3).

**Prior-Request history was offered and declined.** The Reviewer never sees case
rows, and there is no facility parameter.

> **Requirement, not styling: the decision buttons sit BELOW the identity fields
> and the ask.** Approve must not be reachable without passing what is being
> judged. This is load-bearing because the Reviewer's name goes permanently onto
> the release and the human check is this system's second control. It is the
> **weak** form — it costs a scroll, not a click, and a Reviewer determined to
> rubber-stamp still can. A hard gate was available and declined.

### 10.3 The Decision

**What the Reviewer judges.** Two of the five contact fields carry the judgement:
the **name** and the **Workplace**. The Reviewer asks one question of them —
*does this person exist today, and do they work at the Workplace they named?*
That question is the whole test. Size is never a second one
([ADR 0007](adr/0007-the-reviewers-gate-is-identity-not-size.md)).

**Uncertainty is not a rejection. It is a telephone call.** Where the Reviewer
cannot answer that question from what is on screen, they **contact the Requester**
on the supplied number **before** deciding either way. Rejecting the uncertain
case is not the safe default: it silently converts *"I could not tell"* into
*"this person is not who they say"*, and the Requester is never told which.

> **The call is not recorded** (decided 2026-09-04). No event enters §12.4's
> catalogue and no field appears on the Reviewer's screen. **The cost, stated
> rather than discovered:** in exactly the cases where the gate worked hardest,
> the record shows a Decision and a **Snapshot** with no trace of what resolved
> the doubt — a reader years later cannot tell a telephoned Decision from a
> glanced-at one. Accepted because the alternative is a field a Reviewer fills in
> under time pressure, whose accuracy nothing checks, and which would read as
> evidence.

> **Stated so nobody mistakes the call's strength: telephoning confirms that the
> contact details reach a person who answers to the name. It cannot confirm the
> Workplace, because the only source being asked is the person making the
> claim.** The call raises the cost of a false identity; it does not close it.
> This is the same **weak form** as the button placement in §10.2 — and it is the
> human half of the control §18.4 accepts in place of a credential.

**Approve or reject. Nothing else.** A Reviewer cannot modify a Request: an
editable Request breaks the audit chain, because what was approved would no longer
be what was asked and the record could not say which the human actually judged.
Too broad? Reject and let them resubmit narrower.

- **Approve** releases the extraction job into the queue.
- **Reject requires a mandatory internal note**, never shown to the Requester and
  never sent anywhere.
- **The rejection email gives no reason** — "not approved; contact … if you
  believe this is an error". A Requester-visible reason field is a trap: it
  invites the Reviewer to write something that becomes a disclosure or a
  negotiation. Silence about *whether* a Request was refused would guarantee
  resubmission loops, so the refusal itself is stated; only the reason is not.
  **The no-reason rule itself is not stated to the Requester.** The form's
  up-front notice was cut on 2026-09-30 (§16.3), and the rejection email's
  sentence saying no reason is given was cut on 2026-10-05 (#96): the email
  states the refusal and gives no reason, without announcing that it will not.
- The Decision copies a **Snapshot** of what the Reviewer had on screen (§12.3).

### 10.4 Expiry beats a late Decision

**The Decision handler re-derives elapsed business hours before it inserts, and
refuses if the Request is past the threshold.** Otherwise "expired" would mean
"expired unless a tick was slow".

The refusal is not a bare error. The Reviewer just spent real attention on that
Request, so the screen says plainly that it expired while they were reading, and
**the `expired` event payload records that a Decision was attempted and refused.**
That is a rare event worth having in the trail: it is the signal that the
24-hour window is too tight for the Reviewers actually staffing it.

### 10.5 Sessions

**1-hour sliding idle timeout inside a 6-hour absolute ceiling from login.** The
slide handles the walked-away-from-an-unlocked-laptop case; the ceiling bounds a
stolen cookie. **The ceiling always wins and is never extended** — a ceiling with
an exception is not a ceiling.

**Only user-initiated requests extend the session.** This is why the queue does
not auto-refresh: a polling screen resets the idle timer for ever, and an idle
timeout that never fires is not a timeout. Mail is the notification channel, so
the screen does not need to be live.

- **Ceiling fires** → hard cut, redirect to sign-in with a return-to URL back to
  the Request they were on. Re-login lands them on the same screen with the
  Request still pending: ~15 seconds lost.
- **Warning at T-5 minutes** — a bottom-left toast, not a modal or a banner.
- **The idle timeout warns too**, at 55 minutes idle, with one action,
  **ยังใช้งานอยู่**. Pressing it is a user-initiated request, so it extends the
  idle window exactly as any other request does; nothing about the ceiling
  changes. Without it a Reviewer reading a long dossier is signed out without
  notice and loses a half-typed internal note (WCAG 2.2.1; added 2026-09-18).
- Both warnings are `role="alert"` so a screen reader hears them.
- **The mandatory internal note is never persisted client-side.** It is retyped,
  because a shared สคร. desktop is the wrong place for internal notes to linger.

> ⚠️ **An in-flight Decision is never replayed after re-authentication.** A submit
> arriving on a dead session is rejected outright; the Request stays pending and
> the Reviewer must click approve **again**, deliberately, in the new session.
> Auto-replay would put a Reviewer's name permanently on a release for a click
> made in a session that had already ended. It happens inside a live authenticated
> session or it did not happen.

**Concurrent sessions allowed, capped at 3 per Reviewer, oldest evicted** — a
hygiene bound, not a control.

**Sessions and login-throttle state live in Postgres, not Redis.** Deactivation
becomes a query rather than a cache-invalidation problem, a Redis flush cannot
resurrect state that matters, and a throttle a `docker compose restart` clears is
a throttle an attacker can wait out. Volume is a handful of rows a week.

**Cookie:** `httpOnly`, `SameSite=Lax`, and **`Secure` on by default, disabled
only by an explicit development config flag.** There is no TLS before production,
so the insecure setting must be opted into and can never be reached by silent
degradation.

**CSRF:** double-submit token on every state-changing `/reviewer` post, on top of
`SameSite=Lax`. Justified because the thing protected is a one-click irreversible
release of case-level personal data with a named human's identity attached. The
justification is the **irreversibility and the sensitivity**, not the volume —
the largest Extract is ~1,952 rows (§5.3), and a smaller release is not a safer
one.

### 10.6 Alerts on the queue

An **Alert** is a **must-clear queue item**, never a passive list — the queue does
not auto-refresh, so a passive list is a list nobody looks at. It is cleared only
by naming an outcome **from a closed set, never free text**: the count of each
outcome is the only measure this service has of how often its silent failures
actually happen.

Three kinds:

| Alert | Raised by | Assigned to | Cleared by | Outcomes |
|---|---|---|---|---|
| **Send abandoned** | 5 failed send tries (§11.3) | approving Reviewer | that Reviewer | (as collection lapse) |
| **Collection lapse** | 24 wall-clock hours, zero Attempts, raised in business hours (§11.4) | **the approving Reviewer, by name** | that Reviewer, or `system` on late collection | reached the Requester / could not reach the Requester / no action needed |
| **Extraction failure** | a job reaching `failed` (§14.3) | the approving Reviewer | **any Reviewer** | `contacted_requester` / `abandoned`, or `re_ran` written by `system` (§10.7) |

**Why collection lapse is assigned strictly by name:** the action is *phone the
Requester you personally vouched for*, and that Reviewer already formed a
judgement about this person and has the telephone number in front of them.

**Assignment and clearing are two different things
([ADR 0013](adr/0013-deactivating-a-reviewer-widens-their-alerts.md)).**
**Deactivating a Reviewer widens their open Alerts to any active Reviewer, and
never rewrites the assignment.** A Reviewer is never removed, only deactivated,
so without this a by-name Alert outlives the only person permitted to clear it —
a must-clear queue item nobody can clear, which is the one thing an Alert may
never be. The name of the Reviewer who vouched stays where it is; only the
eligible set grows, and `collection_lapse_cleared` carries **both** the assigned
and the clearing Reviewer so the two populations stay separable. The widening is
derived from `deactivated_at` at read time — **no event announces it, and no type
is added to the closed catalogue.** Blocking deactivation until the Alerts are
cleared was rejected: deactivation is frequently not voluntary, and that would
keep an account live at the moment the organisation most wants it shut.

> ⚠️ **There is deliberately no "Probe stalled" Alert, and adding one is a
> regression.** One was specified while approve was blocked on the Probe's count;
> §5.4 removed that block, and with it the only person a wedged Probe could
> strand. A stalled Probe now leaves a Request approvable with a *pending* count
> and, at worst, an approved job queued behind ~35 s of work — not a human in
> front of a screen. An Alert must be a **must-clear queue item**, and this one
> would be a must-clear item for a condition nobody is harmed by.
>
> **What replaced it is a terminal state, not an Alert**: §5.4 abandons a Probe
> whose calls exhaust their retries, so the *job* it gates is released rather than
> queued for ever. That is the half the removal originally left open, and it is
> machinery, not a queue item — nobody has to clear it.

**Why an extraction-failure Alert may be cleared by anyone:** the action is often
just "re-run", and two reachable people is the real availability unit — with a
two-person team the assigned Reviewer is on leave a material fraction of the time,
and an alert only one person can clear is an alert that waits for them. **The
clearing Reviewer is recorded separately from the assigned one.**

**Pressing Re-run defers an extraction-failure Alert; it does not clear it**
([ADR 0014](adr/0014-a-re-run-defers-its-alert-rather-than-clearing-it.md)). The
Alert stays on the queue marked as re-running. A **completed** re-run clears it as
actor **`system`**, outcome `re_ran`; a **failed** one leaves it open with a second
attempt recorded, and raises no second Alert. **One Alert per broken promise, not
one per job.** `re_ran` was never an outcome — it is an action whose outcome is
not yet known — and a Reviewer who cleared with it before the second job settled
put one `re_ran` and one open item on the record for a single unkept promise. This
is the same rule as the late collection two paragraphs below: **the actor who
resolves a thing is the one who actually resolved it.**

**A late collection clears its lapse as actor `system`, never `reviewer`.** No one
gets credit for a call they did not make, and the lapse count must stay honest.

### 10.7 Re-run

A **Re-run** is a second extraction of an already-approved Request, started by a
Reviewer pressing a button.

**A Re-run is NOT a new Decision.** Same Requester, same parameters, same
judgement already snapshotted. Re-judging would put two Decisions on the record
for one release, and a reader years later could not tell which authorised what.
The chain must read *approved once, extracted twice*.

A Re-run:

- produces a **fresh Extract, fresh Download token, fresh 72 h clock** (a new
  object, so completion-anchored retention applies from the new completion);
- **revokes the previous Download token — at *ready*, never at the press of the
  button** ([ADR 0012](adr/0012-a-re-run-revokes-the-previous-download-token.md));
- **does not re-Probe** — the row count is already on `probe_performed` (§12.4),
  and re-probing would spend upstream budget to re-learn a known number;
- carries the **original Decision's id**;
- takes the next `-rN` filename suffix (§8.3);
- **defers rather than clears its extraction-failure Alert** (§10.6).

**One Request never has two collectable Extracts.** §10.7 previously fixed what a
Re-run creates and was silent on what it retires, so a Re-run over a *successful*
first run left two live tokens on two clocks — and the link the Requester was most
likely to click was the one most likely to be dead. Revoking at **ready** rather
than at **queued** is what makes the button free: a failed re-run leaves the
original still collectable, so nothing is destroyed until something better exists.
The revocation is written by the job, so `download_token_revoked` carries a
`system` actor — and since
[ADR 0017](adr/0017-a-reviewer-never-corrects-a-requesters-email-address.md) that
is the **only** actor it ever carries: a Reviewer cannot revoke a token.

**It is a button, never automatic.** Chunk-atomic retry is already exhausted by
the time a job is `failed`, so a self-retry mostly burns another ~25 minutes
against an unchanged cause — and under `N=1` it blocks every Request behind it. A
human who can see *why* it failed decides whether re-running is pointless.

*(This never collides with the startup reconcile, which re-enqueues jobs left
`running` by a crash. Those never reached `failed` and never exhausted anything.)*

### 10.8 Resend, and the boundary it holds

**A Reviewer can resend the Delivery. A Reviewer can NEVER see the Download
token.** Revealing it would put the capability in a second place and make the
Reviewer a channel for case-level data — precisely the boundary the approval gate
exists to hold.

- **Resend to the same address**: free, audited, and **never moves the 72 h
  clock**. The token is never extended by use, and a resend is not use.
- **What is resent is the sent Delivery itself** (decided with the repo owner
  2026-09-24, #74). The raw token is stored nowhere but in the rendered message,
  so a sent Delivery's BullMQ job is **kept in Redis for 72 hours** — the life of
  the token it carries — where every other kind is dropped once sent. A resend
  copies that message into a new send, so the link, the address and the clock
  are exactly what they were; it takes the **current** link's Delivery, never a
  superseded one's. If Redis no longer holds it, the resend is refused and the
  remedy is a Re-run. **The audit is the second `mail_sent` `{kind: delivery}`**;
  who pressed resend is not recorded, and the catalogue gains no type for it,
  because a resend changes nothing a Decision was about. Storing the raw token
  in Postgres, even encrypted, was declined. **Accepted:** for those 72 hours the
  link is readable to whoever can read Redis — the operator, and Bull Board
  (§14.4), which is localhost-only and never on the Reviewer surface — as a
  failed send's already was.
> ⚠️ **A Reviewer never corrects a Requester's email address. The resend control
> takes no address field.**
> [ADR 0017](adr/0017-a-reviewer-never-corrects-a-requesters-email-address.md)
> reverses this section's former corrected-address branch, which issued a fresh
> token to a new address and called itself a new Decision. That guard was a
> formality performed by the person it was meant to check, and the address it
> released to was never verified by this service in the first place (§16.4).
> **A Reviewer decides who receives data, never where it goes.**

A Requester who mistyped their own address has **ended their Request**, exactly as
one who missed their 72 hours has
([ADR 0016](adr/0016-a-lapsed-download-token-ends-the-request.md)). They resubmit
and are reviewed again. The remedy is `requester_email_warning` on the form, which
is the only place a Requester is told that a typo tells nobody — that notice is now
the entire remedy, and it is load-bearing.

*(The mirror with §10.7 collapses, and reads better for it: neither a Re-run nor a
resend is a new Decision, because neither can change anything a Decision was
about.)*

### 10.9 The in-flight list

*Restored 2026-09-18 from #74, which states it in full. This section existed
before the repository was reverted and was cited by #74, ADR 0015 and ADR 0016
while absent from `main`.*

**In flight = approved, and not yet terminal.** Terminal is `collected`,
`expired_uncollected`, or an extraction failure a Reviewer cleared as `abandoned`.
Rejected and expired-undecided Requests are never in flight: nothing remains to be
done to them, and they leave the queue at the Decision.

**Membership is derived at read time, never stored.** No `in_flight` column and no
entering or leaving events; the catalogue stays closed. A Request is on the list
because of what is true about it, not because something wrote it there. **The
definition of *terminal* lives in exactly one place in the code.**

**It is everyone's Requests.** The approving Reviewer's name is on each row as
accountability, not permission; any active Reviewer may act on any of them
([ADR 0013](adr/0013-deactivating-a-reviewer-widens-their-alerts.md)).

**The screen shows the five live contact fields**, read from the Request and never
from the Snapshot, for as long as the Request is in flight
([ADR 0015](adr/0015-the-record-is-contact-free-the-screen-is-not.md)).

**Actions are gated by what is physically possible, not by policy:**

| Extraction state | Row reads | Available |
|---|---|---|
| `queued`, `running` | *extracting — nothing to do until it finishes* | nothing |
| `ready` | time left on the Download token | Re-run, resend |
| `failed` | *extraction failed* | Re-run, and the Alert's clearing outcomes |

Each row shows the wall-clock time left on the Download token. **Sort by submit
order and nothing more urgent**: the Alert section is the only part of this surface
allowed to shout. The list does not auto-refresh (§10.5). **An open Alert
suppresses its Request's in-flight row** (§10.6), so a Request appears exactly once
on the surface.

A lapsed Download token ends the Request, and no Reviewer action revives it
([ADR 0016](adr/0016-a-lapsed-download-token-ends-the-request.md)).

### 10.10 Looking up a Request by its reference

*Decided with the repo owner 2026-09-18.*

**A Reviewer can find any Request, finished or not, by its exact reference number,
and read it.** The confirmation, the collection page, the rejection email and the
expiry page all tell a Requester to telephone and quote their reference; before
this, the Reviewer who answered could not find a finished Request, because
finished Requests leave the surface.

- **Exact reference only.** Not a search by name, email, workplace or telephone.
  §10.2 declined prior-Request history on the review screen, and a search by person
  is that history by another route.
- **Read-only.** A lookup carries no actions. A Request that is still on the
  surface opens in its zone as usual; a terminal one opens as a record.
- **A terminal Request shows the record, never the contact fields**
  ([ADR 0015](adr/0015-the-record-is-contact-free-the-screen-is-not.md)): the ask,
  the Snapshot's `workplace` and row count, the Decision and its Reviewer, each
  Extract and its link state, and the event trail. Contact details leave the
  Reviewer surface when a Request stops being in flight, and the lookup must not
  become the way round that.
- **A lookup writes no event.** It is a read, and no read is an event anywhere in
  the catalogue. Decided with the repo owner 2026-09-18; §12.4 is unchanged.

---

## 11. Email

### 11.1 The premise

> **The system cannot observe whether an email arrived.**

State it in those words. It is a premise, not a footnote.
[ADR 0001](adr/0001-email-delivery-is-unobservable.md).

Why: **every recipient is outside the ministry's mail domains.** Confirmed
2026-09-04 — **no user of this service, Reviewers included, reads mail at
`moph.go.th`**; they are on public providers. So the earlier framing of an
*"internal"* Reviewer hop and an *"external"* Requester hop was wrong: **there is
no internal leg. Every message this service sends crosses to a public mail
provider** and is filtered by that provider's rules, not by any arrangement inside
DDC. Bounces, meanwhile, **return to `envocc@ddc.mail.go.th`, a mailbox this
application does not own**.

> **Junk filing was withdrawn as the leading example, 2026-09-04.** This section
> used to argue that the relay's **non-government domain**
> (`mailrelay.uc-workd.com` under an `@ddc.mail.go.th` sender identity, §11.2) was
> *"exactly the SPF/DKIM/DMARC shape that fails silently as junk"*. **The repo
> owner rules otherwise: the relay is in official organisational use and its mail
> is not filed as spam.** That ruling is recorded as given.
>
> ⚠️ **Read it against the paragraph above, which was confirmed the same day.**
> The relay's standing inside DDC is a fact about the **sender**; with every
> recipient on a public provider, **none of the traffic is judged by anyone who
> knows that.** The ruling and the all-external recipient list were given
> together and pull in opposite directions; both are recorded, neither is
> reconciled, and nothing was tested — see §18.13.
>
> **The premise above is unaffected either way.** Unobservability is not a claim
> about spam: even a relay that never junks anything cannot tell this application
> that a message reached a human. Every mechanism built on the premise — the
> Collection lapse Alert, `expired_uncollected`, the operator banner — stands
> unchanged, and **§11.1 is not weakened by the ruling; it is now carried by the
> recipient list instead.**

**An application-owned bounce mailbox was offered and declined.** It is a new
moving part and a new deployment dependency, and it still misses the dominant
failure — a message filed silently as junk. **Do not "fix" this by adding one
without reopening the decision.**

### 11.2 Configuration

```
SMTP_HOST=mailrelay.uc-workd.com
SMTP_PORT=            # supplied at dev cycle — submission port, confirm the number
SMTP_STARTTLS=true
SMTP_SECURE=false     # explicit STARTTLS on submission, not implicit TLS
SMTP_USER=envocc@ddc.mail.go.th
SMTP_PASS=            # supplied at dev cycle
SMTP_FROM=noreply-dds-sharing@ddc.mail.go.th
FRONTEND_URL=         # supplied at dev cycle — absolute, never derived from Host
```

Sender identity is `noreply-dds-sharing@ddc.mail.go.th`. **The From address must be
on `ddc.mail.go.th`**, whose SPF record includes `_spf.uc-workd.com`; it need not
be the login's own mailbox. A From domain that does not authorise the relay is
accepted by the relay (`250 … queued`) and then silently dropped by Gmail —
found on the Pilot 2026-10-05 with `noreply@dds-sharing.ddc.go.th`, which has no
SPF record at all. Rate limits were not asked about and
are not a design risk: volume is single-digit messages per Request. **Confirm the
relay hostname verbatim before it lands in config** — `uc-workd` is close enough
to a typo to be worth one deliberate check.

**Mailpit covers development.**

### 11.3 Send failure

**The relay refuses the message, or the SMTP conversation fails.** Observable
within seconds. **Our fault** — configuration or the relay's health, never the
recipient's.

- **Retry 5 times over roughly 1 hour** on the 60 s scheduler pass, each try
  written as its own event.
- On the fifth failure the send is **abandoned** and raises a must-clear Reviewer
  Alert for that Request.
- ⚠️ **Two or more concurrent send failures raise the operator banner and the
  `mail` health component.** *One failure is a Requester's problem; two at once
  is an outage.* **The per-Request Alert is still raised during an outage**
  (decided 2026-09-24, #73): the Requester still has no link, and someone must
  still telephone them — a suppressed Alert is a call nobody is prompted to
  make, and its link expires unseen. This line formerly said the banner
  replaced the Alerts.

**The class covers every email the system sends**, but handling differs by kind:

| Email kind | On repeated failure |
|---|---|
| **Delivery** (carries the Download token) | must-clear Reviewer Alert (**Send abandoned**, §10.6); also the only kind with a collection lapse |
| **Rejection** | **no Alert** — the Request is terminal at the Decision and has no approving Reviewer to assign one to (§10.9) |
| **Extraction failure** | **no second Alert** — the Request's open extraction-failure Alert already asks for the same contact (§10.6) |
| **Reviewer queue notification** | ⚠️ **operator banner on the FIRST failure**, not the fifth |

*Rejection and extraction-failure rows narrowed 2026-09-24 (#73): both
formerly read "must-clear Reviewer Alert". An abandoned send of either is
still on the record as `mail_send_abandoned`.*

**Why the queue notification is different, and why it is the sharpest failure in
the system:** there is no Reviewer alert available, because the whole point is
that no Reviewer is looking at the queue. A silent queue notification means **the
approval gate has no trigger and the Request expires at 24 business hours through
nobody's fault.**

**There is no receipt email on submit.** Offered and declined; the Requester gets
a confirmation *page* instead (§16.3). **Accepted cost, recorded knowingly: bounce
detection is lost.** A mistyped address now surfaces only after a Reviewer has
spent time and an extraction job has run. Mitigated by §11.4, not eliminated —
and by a warning on the email field that the address is not checked. Since
2026-10-05 (#96) nothing tells the Requester that a typo goes unnoticed.

### 11.4 Collection lapse

**The Delivery was accepted by the relay, and 24 *wall-clock* hours later the
Requester has made no Attempt on the Download token.** Never observable, only
inferred. Silence is the only signal, and silence is ambiguous — junk folder, or
annual leave.

**Two clocks, doing two different jobs** ([ADR 0011](adr/0011-the-collection-lapse-trip-wire-runs-on-wall-clock-time.md)):

- **The trip-wire is wall-clock**, because the Download token it is warning about
  is wall-clock (§9.3). **A clock that stops cannot warn you about one that does
  not.**
- **The Alert is raised in business hours** — a trip-wire firing outside them waits
  for the next opening at 08:30 (§15.2). An Extract delivered Friday at 16:00 is
  not a failure on Saturday afternoon, and a queue full of weekend noise is a queue
  nobody reads.

> ⚠️ **This was 24 *business* hours until 2026-09-08, and that was a defect, not a
> tuning choice.** The business clock advances 8 hours a working day, so **24
> business hours are exactly 72 wall-clock hours** on a week with no weekend in it
> — the very life of the token. A Monday 09:00 Delivery raised its Alert at
> Thursday 09:00, the same instant the token expired; a Friday 15:00 Delivery
> raised it **48 hours after the Extract had been deleted**. Lowering the threshold
> does not help: 8 business hours from Friday 15:00 still lands on Monday 15:00.
> **Do not re-unify the two clocks.**

- **A trip-wire whose next opening falls after the token has already expired
  raises nothing** (decided 2026-09-23, #72). That is every Delivery from
  Thursday's 16:30 close to Friday's 08:30 opening: it trips after Friday's
  close, the queue next opens Monday 08:30, and its token has died over the
  weekend. The link is dead and nothing revives it (ADR 0016), so the Alert
  would have no action; `expired_uncollected` records the Request.
- **Waiting for the 72 h token expiry was the alternative and it is useless** — it
  fires as the window closes, leaving no time to telephone. The old business-hours
  trip-wire was worse than that alternative, not better.
- **What the Reviewer now gets**: 48 hours to telephone on a Monday Delivery, and
  6.5 hours on a Friday-afternoon one. The second is thin, and it is the first
  arrangement under which that case gets any window at all.
- **The false-positive rate is accepted deliberately, and is now higher.** A
  Reviewer will sometimes telephone a Requester who was merely slow. That costs one
  call. The alternative costs a completed extraction, an upstream slot, and a
  Request that must be resubmitted and re-reviewed.

### 11.5 `expired_uncollected` is a distinct terminal state

Without it, a Request would end identically whether the Requester collected the
Extract or never saw the email — opposite outcomes, one a success and one a wasted
extraction and a wasted upstream slot.

**This is the only number that measures whether email is working.**

---

## 12. The audit record

**A permanent, append-only event record, with the Request row as its projection.**

### 12.1 Who reads it

Four readers, all confirmed in scope:

| Reader | Horizon | Question |
|---|---|---|
| **Reviewer** | live | what is pending, and what am I judging |
| **Operator** | days | what is failing, and is someone sweeping tokens |
| **Incident** | months | an Extract surfaced where it should not — who collected it, did it come from here |
| **Accountability** | years | who authorised this release |

**The record must satisfy the longest-lived reader**, and it therefore carries
fields (IP, user agent) that only the short-lived readers use. Accepted openly.

### 12.2 Shape and enforcement

- **Both a state row and an event stream.** The Request row carries current state
  and the queue reads it directly; `request_event` carries history. **The Request
  row is a cache of the stream, not an independent truth.** A pure projection was
  rejected — the queue screen would be a fold over events on every page load.
- **Events are immutable, with no exception.** A correction is a new event citing
  the prior one — including a Reviewer's mistyped internal note, which becomes a
  `note_amended` event rather than an edit. A correctable audit record is not an
  audit record.
- **Enforced by database roles, not by convention.** The application role holds
  `INSERT`/`SELECT` on the event tables and **`DELETE` nowhere**, and no role
  holds `UPDATE` on the event tables or on `request_contact`. This is the one
  arrangement under which *"the running application cannot rewrite history"* is a
  fact rather than a promise. Convention-only was rejected explicitly: the record
  is permanent, so a bad `UPDATE` in year three would be undetectable.
- **Discriminated actor.** `actor_type ∈ requester | reviewer | system |
  anonymous`. `reviewer_id` is set only for `reviewer`; IP and user agent only for
  the unauthenticated kinds. A single nullable actor blob was rejected because
  "which human did this" would then be a query with a plausible wrong answer
  available.
- **Ordered by `bigserial`, timestamped `timestamptz` in UTC.** Two events share a
  timestamp routinely — `approved` and `job_queued` land in one transaction;
  Attempts collide under a token sweep. **The sequence answers *in what order*;
  the timestamp answers *when*.** UTC storage with ICT rendering is load-bearing,
  not hygiene: the business-hours clock is defined in ICT, and a naive local
  timestamp would make it unauditable across a server timezone change.
- **Closed event catalogue**, `jsonb` payload with a documented per-type shape
  (Drizzle enum plus a discriminated union in TypeScript). **Adding a type is a
  migration *and* a spec change.** Deliberate friction: an open-ended
  type-plus-payload column is exactly how a third body of personal data gets
  created by accident.
- **Two timestamps on every late-materialised event**: **`occurred_at`** (the
  moment the predicate became true — computed, and the legally meaningful one) and
  **`recorded_at`** (the insert). Normally within a minute; **when they diverge,
  that divergence *is* the outage record.**

### 12.3 Tables

| Table | Holds | Notes |
|---|---|---|
| `request` | parameters, state, reference number | **carries no identifying data** |
| `request_contact` | name, surname, tel, email, `workplace` | split out |
| `request_event` | the append-only stream | |
| `token_lookup` | every presentation of a Download token | `request_id` **nullable**; **token prefix only** |
| `reviewer` | `username`, `display_name`, `email`, `deactivated_at`, password hash, TOTP secret | |
| `reviewer_event` | the append-only Reviewer stream | a login belongs to no Request |
| `reviewer_session`, login-throttle | operational state | **genuinely deletable — see §15.4** |
| `province` | the seeded lookup (§6.4) | app role read-only |

**Why `request_contact` is split out.** It originally existed to make erasure a
single `DELETE`. Erasure is gone and the split was **kept on a different
justification**: the Reviewer queue is the only reader that needs those fields and
it reads one Request at a time, so the join is cheap — while every other query
(abuse patterns, service-promise measurement, what-was-released) touches **no
personal data at all**.

**Why `token_lookup` is separate.** An unknown token resolves to no Request and so
cannot be a child event of one. A *successful* presentation is written to **both**
tables — mirrored as a `download_attempted` event — because the two readers ask
different questions: accountability asks *"who collected this Extract?"*, abuse
asks *"is one IP sweeping the token space?"*, and the second query must not have
to filter out the first. **Token prefix only, never the full presented token** —
logging it in full would put working credentials in a permanent trail.

**Human form and upstream form are both stored, in different places.** The
`request` row stores the ask **as the human made it** — inclusive dates, the
Disease group's name, and the two expansions it and a region resolved to: the
**Report code list** (§4.9) and the province list. **The expansions are
authoritative**, because both taxonomies are amendable and a Re-run must refetch
what the first run fetched, not what the names mean today. That is what the Reviewer judged and what
the Snapshot copies. **Each Report code's fetch writes its own event** carrying
the exact `group_code`, half-open `start_date`/`end_date`, page count, and the
upstream **`x-request-id`**.

**The Snapshot** copies the Disease group name **over the Report codes it expanded
to**, date range, Area selection, Probe row count and `workplace` — what the
Reviewer had on screen. Since §5.4 the count may be **`pending`** or **`failed`**
there, because approve does not wait on it; the Snapshot records what was on
screen, not what was eventually learned. It does **not** copy the
contact fields. A Reviewer cannot modify a Request, so content cannot drift; the
Snapshot exists to make the Decision legible on its own years later.

### 12.4 The event catalogue

**`request_event`** — closed. Adding a type is a migration and a spec change.

*Request lifecycle*

| Type | Actor | Notes |
|---|---|---|
| `submitted` | `requester` | carries IP, user agent |
| `probe_performed` | `system` | Report codes probed, calls made (**every call, retries included**, §13.6), the probed span, **per-code and total `total_items`**, upstream `x-request-id`s. **Fires when the Probe finishes — after submit, and usually but not necessarily before any Decision**, since approve no longer waits on it (§5.4) — this is what makes the reject path's upstream traffic accountable |
| `probe_failed` | `system` | a Report code's Probe calls exhausted their 3 attempts (§5.4). Carries the code, the relay of upstream errors and their `x-request-id`s, and the calls the whole Probe spent, the codes before this one included (§13.6). **Terminal for the Probe** — the count never lands. It gates nothing: the job runs regardless (§7.8), and only the zero-row catch is lost |
| `approved` | `reviewer` | carries the Snapshot |
| `rejected` | `reviewer` | carries the Snapshot and the **mandatory internal note** |
| `note_amended` | `reviewer` | cites the event it corrects |
| `expired` | `system` | `{notified_at, business_hours_elapsed, reviewer_accounts_active, decision_attempted_and_refused}`. `notified_at` is when the first queue notification was accepted by the relay, **null if none ever was** — the record of an expiry "through nobody's fault" (§11.3) |

*Extraction lifecycle* — **enumerated explicitly here**, because it was previously
described only in prose while a mail kind already pointed at it. An implementer
reading the ticket record alone would find `job_queued` and nothing else.

| Type | Actor | Notes |
|---|---|---|
| `job_queued` | `system` | |
| `job_deferred_low_disk` | `system` | free space below the 1 GB floor (§7.8) |
| `job_started` | `system` | |
| `code_fetched` | `system` | one per Report code: exact upstream params, page count, calls made (every page of every attempt, §13.6), `x-request-id`, rows received vs `total_items` |
| `job_completed` | `system` | the two-group payload below, plus the **Probe-vs-run drift**: the Probe's per-code totals against the run's. **Recorded, never asserted** (§5.4) — real drift between Probe and run is expected and legitimate |
| `job_failed` | `system` | **cause**: `upstream_5xx` / `auth_expiry` / `completeness_mismatch` / `stall` / `internal`, plus the upstream `x-request-id` and the calls spent on the code it failed on, which no `code_fetched` carries (§13.6). **Operator-facing only** |
| `extraction_alert_raised` | `system` | |
| `extraction_alert_cleared` | `system` \| `reviewer` | `reviewer` names `contacted_requester`/`abandoned`; **`system` writes `re_ran` when a deferred re-run completes** (§10.6). Carries **both** the assigned and the clearing Reviewer, and the count of re-run attempts |
| `extraction_rerun_queued` | `reviewer` | carries the **original Decision's id** |

*Delivery and collection*

| Type | Actor | Notes |
|---|---|---|
| `mail_sent` | `system` | `{kind: delivery \| queue_notification \| rejection \| extraction_failure, to, relay_response}` |
| `mail_send_failed` | `system` | try number, relay error |
| `mail_send_abandoned` | `system` | fifth try failed — or the queued send was lost from Redis, which cannot be rebuilt because the rendered Delivery is the only place its raw Download token existed (§15.3) |
| `delivery_alert_raised` | `system` | send abandoned |
| `download_attempted` | `anonymous` | mirrored from `token_lookup` |
| `collection_lapse_raised` | `system` | **24 wall-clock hours, zero Attempts**, raised at the next business-hours opening (§11.4). Carries the wall-clock hours elapsed, so a trip-wire that fired on time is distinguishable from one whose Alert waited for Monday |
| `collection_lapse_cleared` | `system` \| `reviewer` | `system` = collected late; `reviewer` carries the closed three-value outcome, and **both** the assigned and the clearing Reviewer — they differ when the assigned Reviewer was deactivated (§10.6) |
| `download_token_revoked` | `system` | **A Re-run whose new Extract is ready** (§10.7), naming the run that superseded it. **The only cause** — a Reviewer cannot revoke a token (ADR 0017) |
| `expired_uncollected` | `system` | **terminal state** |
| `object_deleted` | `system` | actor, object key, timestamp, outcome |

> **`mail_bounced` does not exist and must never be added.** Bounces return to a
> mailbox the application does not own, and reading it was declined. **A type that
> can never be written is a lie in the schema** — a future reader would assume
> bounces are covered.

**`job_completed` payload — two groups, deliberately not merged:**

| Group | Fields |
|---|---|
| **Extract fingerprint** — *what was released* | `row_count`, `column_count`, `csv_bytes`, `zip_bytes`, `csv_sha256` |
| **Reference data** — *what made it* | `provinces_checksum`, `data_dictionary_checksum` |

Plus the **Extract archive filename**, and the **count of impossible derivation
inputs** (§6.3). Both sizes are kept and named distinctly: the Extract's size is
the data's size; the archive's is what the disk bound and the edge argument reason
about.

**`reviewer_event`** — `login_succeeded`, `login_failed`, `logged_out`,
`session_expired`, `password_changed`, `seeded`, `totp_enrolled`, `deactivated`,
`password_reset`, `totp_reset`.

- **Actors and payloads** (settled with #61): `login_succeeded`, `logged_out`,
  `session_expired`, `password_changed` and `totp_enrolled` have a `reviewer`
  actor. `login_failed` has an `anonymous` actor, because the login did not
  succeed, and carries `{username, failed_factor, totp_clock_drift}` —
  `failed_factor` is which of the checks failed (§17.5: the record keeps it, the
  screen never says). `seeded` and `deactivated`
  are written by a host command and so have a `system` actor and name no one
  (ADR 0020). `seeded` has an empty payload; `deactivated` carries only
  `{force}`, whether `--force` overrode the two-Reviewer floor.
  `password_reset` and `totp_reset` are the host password reset and TOTP
  re-enrolment (§17.5), also `system`, and carry `{username, sessions_ended}`:
  a `system` actor has no reviewer id, and a reset must be on the record the
  moment it runs — not only when the Reviewer next signs in, which may be never.
  The Reviewer's own follow-up still writes `password_changed` or
  `totp_enrolled`.
- Failed logins carry IP and user agent and **never the submitted password or
  TOTP code** — the pattern is the signal, not the credential.
- **A `login_failed` whose code was valid one or two TOTP steps ago is recorded
  distinctly.** That is host clock drift, not an attack, and distinguishing it is
  what makes §16.4's NTP failure diagnosable rather than mysterious.
- **Nothing about Requests is ever added here.** Clearing an Alert belongs to a
  Request, so it is a `request_event` with a `reviewer` actor. `reviewer_event` is
  for things a Reviewer did that belong to no Request.

### 12.5 The reference number

**A display label; the UUID is the key.** `REQ-2569-0142` in shape, stamped at
submit and accepted by every lookup surface, but foreign keys use the UUID.
**The prefix is configuration** (`REFERENCE_PREFIX`, `REQ` by default): each
service that keeps its own record stamps its own, and each Pilot has its own —
`PLT` for the TS Pilot, `PLG` for the Gleam Pilot
([ADR 0022](adr/0022-a-disposable-pilot-releases-real-data-from-the-repo-owners-server.md)),
so a number quoted over the telephone names the service that holds it. Its
exact format — Buddhist-era year, what the counter resets on, whether it must be
unguessable — is free to change, because a key that is not finished being designed
should not be load-bearing in a record that can never be migrated by deletion. It
is quoted over the telephone, so it does not need to be unguessable.
**Settled 2026-09-04: this section is the answer.** The shape above is the format,
and its freedom to change is the decision — not a gap waiting on one. What the
counter resets on is an implementer's choice, constrained only by the shape.

**It first appears on the confirmation page**, and there is no receipt email, so a
Requester who closes that tab loses it until the Decision email arrives. Nothing
breaks, but state it plainly rather than let an implementer discover it.

### 12.6 Expiry as evidence

The `expired` event's payload makes the 24-business-hour service promise
**measurable rather than anecdotal**. It is free at write time.

> ⚠️ **It is performance data about named staff, on a permanent record.** See
> §12.7.

### 12.7 Retention: indefinite, for four bodies of personal data

**Nothing is ever deleted.** Decided three times; the third time with a reason, a
lawful basis, and two places a human is told.
[ADR 0004](adr/0004-personal-data-is-retained-indefinitely.md).

**The ground on record is auditing and traceability of data releases.** The lawful
basis is **legal obligation and legitimate interest**. *Consent was rejected on
the merits, not on convenience*: a Requester withdrawing it could erase the record
of a release that actually happened, and consent that cannot be withdrawn is not
consent.

**Four bodies, not two:**

1. **Contact data** — `request_contact`. DDC and สคร. staff.
2. **Network data** — IP and user agent on `request_event`, and every
   `token_lookup` row.
3. **Accountability data** — the Decision chain, the Snapshot, and the Reviewer
   display name kept resolvable for ever.
4. **Staff performance data** — named here as a class for the first time:
   `expired{business_hours_elapsed, reviewer_accounts_active}`, the login and
   failed-login stream, `collection_lapse_cleared`, `extraction_alert_cleared`.

> ⚠️ **The hardest case was pressed and kept: `token_lookup` rows matching no
> Request.** These are IP addresses of anonymous strangers who presented a bad
> token — no release to audit, no Request to trace. Kept because **a token-guessing
> sweep is only ever visible in hindsight**, so evidence deleted on a one-year
> clock is evidence missing from the only investigation that would want it. The
> cost, stated openly: **the system's longest-lived body of personal data is about
> people who never used it and are not staff.**

**Consequences, taken rather than softened:**

- **No retention job, no deletion role, no `retention` component on `/health`.**
- The **`DELETE`-nowhere guarantee stands untouched and now has an argument**:
  every retention scheme considered required a hole in it.
- **"Data does not linger" is formally a claim about surveillance data only.** Any
  sentence saying it without that qualifier is wrong.

> **The contrast, carried verbatim and not paraphrased: *the patient-derived
> Extract is destroyed after 72 hours; the officer's telephone number is
> kept for ever.***

### 12.8 Removed — see [ADR 0019](adr/0019-a-requesters-contact-details-are-never-removed.md).

### 12.9 Where the two populations are told

A rule nobody hears is a compliance artefact, not a policy.

- **Requester — at submit.** One Thai sentence — `requester_retention_notice` in
  the copy catalogue (§16.3) — beside the email-typo warning, above the form and
  not in a footer. It carries three things:
  **what** is kept (contact details and the record of the Request), that it is
  kept **indefinitely**, and **why** (audit and traceability of data releases).
  Naming the reason is what makes it read as a policy rather than a leak.
- ⚠️ **Reviewer — at seeding and at first login, NOT in this specification.** Four
  separate decisions deferred this to "the spec states it plainly", and **the spec
  is read by implementers, not by Reviewers** — who never see a submit form and so
  have no surface where the telling would happen, while holding the *most* data of
  anyone. So the seeding CLI, which already prints a one-time password and a
  terminal QR, **also prints what is recorded about them and that it is
  permanent**, and **first login shows it once** alongside the forced password
  change. It must say: every sign-in and failed sign-in with IP address and user
  agent, response times against the 24-business-hour promise, every Alert cleared
  and uncleared, and the display name on every Decision — all kept indefinitely,
  for audit and traceability.

---

## 13. Load, abuse, and what they do not protect

### 13.1 The named Non-goals clause

Carry this into the spec's own words, where a reviewer will see it:

> Rate limiting slows casual scraping and protects the upstream DDC relationship
> and this server's disk. It is **not** a data-protection control. The
> de-identification allowlist and the approval gate are the controls standing
> between this service and disclosure.

The risk of silence is specific: a DDC reviewer sees "rate limiting" under a
security heading and banks a safeguard that is not there.

### 13.2 Global extraction concurrency, N = 1

**Keyed on nothing. Not a conservative guess — what the measurements force.**
Eight concurrent upstream calls degraded to ~14.3 s each with zero throughput
gained (§5.3). Any N > 1 makes the service slower for everyone while increasing
the chance DDC notices us. `N` is configurable; **record why it is 1** so a future
operator does not tune it upward expecting throughput.

**Per-client quotas are rejected outright.** A 1000× cost spread between Requests
makes counting submissions meaningless — one full-year `201` Request is a single
~3.5 s call; one full-year `02` Request is ~115 calls. And both candidate keys
rotate for free: IP via any phone hotspot, and the audit email is never verified —
using it as a control key is precisely how a later reader comes to assume it *is*
verified.

### 13.3 There is no queue admission control

**Nothing is refused, deferred, or projected on grounds of size.** A Request is
approved or rejected by a Reviewer on the identity of the person asking
([ADR 0007](adr/0007-the-reviewers-gate-is-identity-not-size.md)), and then it
runs.

This section used to hold a projected-drain estimate — *"this will start in ~40
min and take ~20"* — shown to the Reviewer as advisory. It is **removed**, for two
reasons that compound:

- **The Reviewer may not act on it.** ADR 0007 ruled that long runtime is not a
  ground for rejection. A number the only person seeing it is forbidden to use is
  not advice, it is decoration on the one screen this design depends on being
  read.
- **There is nothing left to project.** Since §7.2 a job is one call per Report
  code: the widest Disease group in the catalogue is **ten calls, ~35 seconds**
  (§7.9). The formula's inputs — chunk counts, page counts, a 4% error bound —
  described a system that no longer exists.

**What the Reviewer sees instead is queue position** — how many Requests are ahead
of this one (§10.2). It is an honest statement about the queue rather than a
prediction about upstream, and at N=1 concurrency (§13.2) with sub-minute jobs it
is the only part anyone could have used.

> ⚠️ **Do not reintroduce a size gate here.** The earlier hard 2-hour refusal at
> submit existed because nobody was watching; a human now is. And a hard refusal
> would let a Reviewer approve a Request the system then rejects — **two gates
> disagreeing, which is worse than either.** That reasoning survives the
> simplification intact.

### 13.4 Download endpoint

**Failed/unknown token lookups only: 20 per IP per hour, then a 1-hour block.**
This is a deliberate exception to "keyed on nothing", justified by being about
token guessing rather than load. **Successful downloads are never throttled.**

**Every failed lookup is an audit row. The block is not the useful output — the
pattern is.**

### 13.5 Disk

```
worst-case stored bytes  =  (retention window / worst-case job duration) × archive size
                         =  (72 h / 7 min) × ~50 KB  ≈  31 MB
```

> **The counterintuitive inversion belongs in the spec: the faster the extractor
> gets, the more disk it needs.** The formula is the durable part; the number has
> now moved twice — first when retention dropped from 7 days to 72 h, and again
> when [#33](https://github.com/rawinan-soma/dds-sharing/issues/33) established
> that the archive is tens of KB rather than ~30 MB. **The old ~5 GB bound was
> sized on out-of-scope code `02` and is withdrawn.**

> ⚠️ **Disk is no longer a real constraint, and the thresholds below are worth
> keeping anyway.** ~31 MB of extracts will not fill anything; but `/health`'s
> `disk` component watches the *volume*, which also carries PostgreSQL, logs and
> the container images. Keep the check, and do not justify it with the extract
> figure.

- **Warn at 75%, unhealthy at 90%**, on `/health`'s `disk` component.
- **The responder is the service owner with shell access, never a Reviewer.** A
  Reviewer cannot resize a volume; alerting them would be pure noise on the one
  screen this design depends on being read.
- **No eviction policy.** Deleting a completed Extract inside its token window
  would hand the Requester a valid link to a file that is gone — trading a loud
  failure for a silent one. Size the volume for the bound, alarm on the high-water
  mark, and let §7.8's 1 GB refuse-to-start floor be the backpressure.

### 13.6 Upstream traffic accounting

DDC issues one bearer token for the whole service, and the failure mode is **DDC
noticing and revoking us**, which no retry recovers from. "How much traffic are
you sending?" must be answerable.

- `code_fetched` covers running jobs; **`probe_performed` covers submits,
  including those that are rejected or expire** (§12.4). `probe_failed` covers the
  calls an abandoned Probe spent before giving up — traffic spent either way, and
  the retries make it more than a successful Probe's, not less. `job_failed`
  covers the calls a failed job spent on the code it failed on.
- **Each carries `callsMade`: every call that reached upstream, retries
  included** — answered; cut off while in flight, by the 60 s timeout or a
  stall; or dropped by the other side after it was sent. A call that failed
  before it left the host (a DNS failure, a refused connection, a failed TLS
  handshake) is not counted: DDC never saw it. The retries this counts pile up
  exactly when DDC's server struggles, which is when DDC is likeliest to ask
  (decided 2026-09-27, #99).
- **The count is not a floor, and it has one known error.** A stall's cancel
  can land while a call is still connecting, and that call is counted though
  it never arrived. It errs high, never low, and at most by one per stall. A
  `job_failed` carries only the calls no `code_fetched` already did, so no
  call is counted twice.
- **A CLI report on the Docker host** counts upstream calls over a date range,
  split by Probe and fetch. Not a dashboard and not an endpoint: this question
  gets asked by a human a handful of times a year, and a dashboard nobody opens is
  a cost with no reader.

---

## 14. Observability

### 14.1 One `/health` document

**Unauthenticated, with named components: `scheduler`, `extraction`, `disk`,
`mail`.** Non-200 if any component is unhealthy, with per-component status in the
body. **`/health/scheduler` is kept as an alias.**

One surface is one thing to ask DDC infra to watch. Four paths means three of them
never get watched.

> **Stated rather than hidden: an aggregate non-200 cannot distinguish a dead
> scheduler from a 90%-full disk.** Any uptime checker worth configuring reads the
> body.

> ⚠️ **`/health` is unauthenticated and leaks that this service's disk is filling
> or its extractions are failing.** Accepted: the alternative is an authenticated
> health endpoint no external checker can watch, which is the same as not having
> one. **The document carries statuses only — never counts, never Request data.**

| Component | Unhealthy when |
|---|---|
| `scheduler` | heartbeat stale > 5 min (§15.3), or an object still present 1 h past token expiry, or an unrecognised province code (§6.3) |
| `extraction` | **two consecutive** extraction failures; a success resets the count |
| `disk` | ≥ 90% used (warn at 75%) |
| `mail` | **two concurrent** send failures |

**Why two, twice:** *one failure is a Requester's problem; two is an outage.* One
principle, reused verbatim, rather than two invented numbers. **Consecutive, not a
rate** — with `N=1` concurrency and jobs measured in tens of minutes, a windowed
rate has too few samples to mean anything. If a single failure reddened `/health`,
the endpoint would be red routinely and would stop meaning anything.

### 14.2 Two watchers, told different things

A failed extraction is simultaneously a **technical fault** only shell access can
fix and a **broken promise to a named person** only a Reviewer will contact.
Assigning both halves to one watcher is how this goes wrong: tell only the
operator and the Requester is never contacted; tell only the Reviewer and the
alert becomes wallpaper, because **a Reviewer cannot fix a failed extraction.**

- **Operator** → `/health` (§14.1) and the Reviewer-queue banner for
  scheduler-class faults.
- **Approving Reviewer** → the must-clear Alert (§10.6).

### 14.3 The Requester is emailed on failure

The Requester submitted, was told to expect an email, and closed the tab —
whether the job takes thirty seconds or thirty minutes. **Silent death is the
worst outcome for this audience.** The Requester sees **one undifferentiated
failure** — they cannot act on "504 on code 214 of the group". The cause split
lives in `job_failed` and is operator-facing only.

### 14.4 Bull Board

**Ships, bound to localhost, reached by SSH port-forward from the Docker host.**

- **Not published through the public ingress** — it has no auth of its own and
  would ride the single origin.
- **Not behind Reviewer auth**, for a sharper reason: **a Reviewer's entire job is
  to never see case data**, and putting a raw job inspector on their surface
  leaves them one unlucky error payload away from it.
- **It is a debugging tool carrying no watcher obligation.** The watching duty sits
  entirely on `/health` and the Reviewer queue.

### 14.5 Application logs

**Application logs are kept for 72 hours, then deleted — the same clock as the
Extract.** They are operator-facing, they live on the Docker host, and they are
**not** part of the audit record: nothing in §12 may be reconstructed from them,
so deleting them takes nothing §12.7 promises to keep.

**No case data ever reaches a log line.** That ban is the control; the 72-hour
lifetime is the backstop behind it. Concretely:

- The pipeline stages of §7 log **counts, Report codes, page numbers and elapsed
  time** — never a row, and never a field of a row. A projection error logs the
  **row index**, never the row.
- The upstream client logs **status, Report code, page number and
  `meta.total_items`** — **never a response body.** An error carrying a body is
  logged with the body **removed, not truncated**: a truncated body is still case
  data, and it reads as though somebody had thought about it.
- A **BullMQ job payload carries the Request id and nothing fetched**, so a failed
  job sitting in Redis holds no rows.

> **Why the ban and the clock both.** The ban is the control, because a log line
> has none of the three things that make the 72-hour rule trustworthy elsewhere:
> the allowlist decides what may exist (§6), the destruction rule ends it (§9.3),
> and the **Extract fingerprint** records what left (§8.3). A log has no
> allowlist, no fingerprint and no audit event. **The matching clock is defence in
> depth** — if some unhandled path ever breaks the ban, the breach expires on the
> same schedule as the data it copied instead of outliving it. Aligning the two
> numbers is not what makes this safe, and nobody should later read the matching
> numbers as the reason.

> ⚠️ **The accepted cost, recorded so it is not rediscovered: a fault that fires
> on a Friday has no logs by Monday.** `/health` reddens only after **two
> consecutive** extraction failures (§14.1), and this is a one-operator service
> (SRS §6.3 R13). Diagnosis therefore depends on the operator looking inside three
> days, or on the audit record alone — which holds the **fact** of the failure
> (`job_failed`, §12.4) but never its cause.

---

## 15. Scheduled work

### 15.1 Expiry is derived, not scheduled

Both expiry rules are **predicates computed at read time**, never scheduled state
changes:

- **Request expiry at 24 business hours** — the Reviewer queue computes elapsed
  business hours when it renders. A Request past the threshold is simply not
  actionable.
- **Download token expiry at 72 h** — the download endpoint checks
  `now > expires_at` on every request.

**So a dead scheduler cannot un-expire a Request or keep an Extract reachable.**
Only two jobs genuinely need a timer: **physical object deletion** (a file nobody
asks for is never noticed) and **stall detection** (a stuck job never triggers a
read). The tick's remaining work on expiry is *materialising an event row for a
fact that is already true* — a late tick produces a late row, not a wrong outcome.

### 15.2 The business-hours clock

**Mon–Fri 08:30–16:30 ICT, and nothing else stops it.** The clock only advances
inside those windows, so a 02:00 Sunday submit starts counting at 08:30 Monday.
**There is no public-holiday list**
([ADR 0021](adr/0021-the-business-hours-clock-has-no-holiday-list.md)).

> **Accepted cost: a public holiday counts as a working day.** A Request that
> arrives before a holiday has less real Reviewer attention inside its 24 business
> hours than its window says, and over a long holiday it can expire with nobody at
> their desk. The earlier design subtracted a checked-in holiday list and claimed
> a stale list could only make expiry more generous. That was false: a stale list
> *omits* holidays, which shortens the window. The list was removed rather than
> maintained by hand every year for a property it could not deliver.

**The same clock serves two callers, and it answers a different question for
each.** Request expiry (§10.4) asks it *how much attention time has elapsed* — the
24-hour window is a promise about Reviewer availability, so business hours are the
measurement itself. The collection lapse (§11.4) asks it only *is anyone there to
be told?*, using it to schedule when an Alert may be raised; **the silence it
reports is measured in wall-clock hours**, because the Download token it warns
about expires in wall-clock hours.

> ⚠️ **Do not re-unify these two uses.** Measuring the lapse in business hours is
> the natural-looking simplification and it is a defect: 24 business hours are
> exactly 72 wall-clock hours on a weekend-free week, so the Alert lands on the
> token's expiry at best and two days past it at worst
> ([ADR 0011](adr/0011-the-collection-lapse-trip-wire-runs-on-wall-clock-time.md)).

### 15.3 The tick

**`@nestjs/schedule`, every 60 s**, querying Postgres for due work and enqueueing
real BullMQ jobs to do it. **Redis executes; Postgres is the truth.** BullMQ
repeatable jobs were rejected: a schedule living only in Redis is a schedule a
Redis loss silently cancels.

- **Stateless and disposable.** "Due" is a query, not an event, so a restart needs
  no catch-up logic — the next pass picks up everything outstanding. **There is no
  missed-window class of bug.**
- **One pass, several queries, one Postgres advisory lock.** The lock is taken each
  pass, so single execution is *enforced* rather than remembered and the deployment
  shape stays free — scaling the app container later cannot silently double every
  deletion. Separate cadences per job were rejected: every job here is "find due
  work", and four schedules mean four heartbeats and four ways to be half-alive.
- **The startup reconcile is the same pass with no lower bound on "due"** — not a
  separate code path. It touches exactly two things: approved Requests whose
  extraction was unfinished — `running`, or `queued` with no live BullMQ job
  (§7.7) — when the process died (re-enqueue; code-atomic retry makes this
  safe), and expired Download tokens whose objects still exist (delete).
  **It never touches `pending`.**
- Every "cleanup" the tick performs on the event tables is expressed as an
  **insert**, per §12.2.
- **What is enqueued and what runs in the pass** (decided 2026-09-23, #72).
  Extraction re-enqueues and mail send-retries go to BullMQ. Object deletion,
  materialising `expired` and `expired_uncollected`, the collection lapse and
  pruning run **inline under the lock** — single execution is then the lock's
  guarantee rather than a job-id's. Each MinIO call times out at 30 s, so a hung
  call fails its job instead of stalling the pass, and **a pass with a failed
  job writes no heartbeat**: a half-alive tick reaches the banner like a dead
  one. A failed delete writes no `object_deleted` — the record holds settled
  outcomes only; the failure is in the log, the missing heartbeat and the
  1-hour overdue signal.

**The work on the pass:** object deletion at token expiry · stall detection ·
materialising `expired` · due mail send-retries · Deliveries past 24 **wall-clock**
hours with zero Attempts · pruning §15.4's tables.

**Liveness: the tick writes a heartbeat row every pass; stale after 5 minutes**
(five missed passes — unambiguous).
In-band alerting is circular, so one fact feeds two consumers:

- **A Thai banner on the Reviewer queue**, stating plainly that automatic
  processing has stopped and what that means for their work — not an error code.
  This is the guaranteed reader: the one screen a named, accountable human opens
  daily, and the person whose Requests are the ones going stale.
- **`/health`** (§14.1), so DDC infra or any uptime checker can watch it without a
  login.

An external dead-man's-switch service was rejected: an outbound internet
dependency on a ministry host, and a new vendor in the compliance conversation.
**The design deliberately never *requires* email.**

### 15.4 The one place `DELETE` is correct

`reviewer_session` and the login-throttle table are **operational state, not event
tables.** They carry no accountability value and *should* be genuinely deletable.

> ⚠️ **The pruning job needs `DELETE` on exactly those two tables and on nothing
> else.** Draw the role boundary around it, or this becomes the reason the
> application role gets a broad `DELETE` grant that §12.2's whole enforcement
> argument rests on it not having.

---

## 16. The application

### 16.1 Shape

**A plain Angular SPA, one build, served by NestJS from the same origin as the
API.** One container, one port, one public route for DDC infra to register, no
CORS. `docker compose down` remains the whole kill switch. The static handler
excludes the API prefix so it cannot swallow API responses.
[ADR 0003](adr/0003-plain-spa-and-a-collection-path-that-bypasses-it.md).

**No SSR**, rejected on all three of its benefits: search indexing does not apply
(a Requester is given the address by DDC), there is no per-request server data on
the form (the Disease group list is a fixed ten-value seed from
`docs/disease-groups.md`; the confirmation page
is client state), and a faster first paint precedes a wait of up to 24 business
hours. The cost — a second runtime on the host that the kill switch would have to
account for, plus hydration as a failure mode — is not worth it.

**No prerendering**, rejected separately: a prerendered *form* is a picture of a
form until the bundle hydrates.

**One build, not two.** Separating bundles is not an access control, and lazy
loading already keeps the Reviewer code out of the first download.

**Screen widths** (decided with the repo owner 2026-09-18). The public surface —
the form, the check page, confirmation, collection and expiry — works down to a
360px phone, because a Requester in the field may have nothing else and the
Delivery email is usually opened on one. **The Reviewer surface is desktop-only,
from 1024px.** Below that it shows one sentence asking for a computer. A Decision
puts the Reviewer's name on a release permanently and rests on a telephone call
made while reading the dossier; a phone between meetings is the setting where that
becomes a rubber stamp.

### 16.2 Routes

| Path | Served by | Note |
|---|---|---|
| `/` | Angular | the Request form, one scrolling page |
| `/submitted` | Angular | confirmation. **Client state only — nothing in the address** |
| `/link-expired` | Angular | §9.4's one sentence. Reached only by redirect |
| `/reviewer/...` | Angular | sign-in, queue, one Request. Sign-in accepts a return-to address |
| `/api/...` | NestJS | |
| `/d/<token>` | NestJS | The collection page, server-rendered, no Angular. A lookup, not an Attempt. **Fixed — travels in email** |
| `/d/<token>/archive` | NestJS | The archive, with range requests. **Counts the Attempt** (ADR 0018) |
| `/health`, `/health/scheduler` | NestJS | unauthenticated |

English words, lower case, **no language prefix** (§16.3 leaves nothing to
prefix).

**`/submitted` holds nothing in the address**, because a confirmation page
addressable by reference number would be a second unauthenticated capability
exposing a Requester's own ask — exactly what §9.4 refused for the resubmit link.

**The download endpoint supports range requests** (`Accept-Ranges: bytes`,
honouring `Range`). No single request has to last long, so a proxy read-timeout
cannot kill a transfer mid-flight and a สคร. connection dropping at 80% resumes
rather than restarting.

**The base URL is explicit configuration, never derived from the `Host` header.**
Behind an edge this project does not control, inbound headers are not trustworthy,
and the download link in the Delivery email is an absolute URL — deriving it from
`Host` is how a poisoned link reaches a Requester's inbox.

### 16.3 Language and copy

**Thai is the only language shown to a person.** No language prefix in any
address, and no separate answer for the Reviewer surface. English survives in the
Extract's column headers, with the Thai/English Data dictionary in every archive,
and — since [ADR 0010](adr/0010-copy-is-authored-in-english-and-translated-before-production.md)
— in a source catalogue that is maintained but **never served**. **English is in
the file, never on the screen.**

**The copy is authored in English and translated to Thai before production.**
[ADR 0010](adr/0010-copy-is-authored-in-english-and-translated-before-production.md)
carries the reasoning, and the reversal it makes of this section's former ruling
that `messages/en.json` was deliberately not maintained. In short: the message
catalogue **cannot carry a description field** — the inlang SDK data model has no
metadata column, and a sibling `_note` key would compile into a shipped message
function — so a second locale file is the only way the toolchain will hold an
English statement of what each sentence is *for*. That statement is what the
translator works from.

**Paraglide message-format**, with `project.inlang/settings.json` at the repo root:

| | Development | Production |
|---|---|---|
| `baseLocale` | `"en"` | `"th"` |
| `locales` | `["en", "th"]` | `["th"]` |
| `strategy` | `["cookie", "baseLocale"]` | `["cookie", "baseLocale"]` |

`messages/en.json` is **checked in and permanently maintained**: a copy change
touches both catalogues in the same commit. During development an incomplete
`th.json` degrades to readable English; **after the flip it degrades to the raw
message key**, which is why §17.1 carries a gate and why the keys are descriptive.
**The `url` strategy is never added** — its default patterns leave the base locale
unprefixed and prefix every other, so `baseLocale` would decide which language owns
`/`, reintroducing the prefix this section forbids as a side effect of a change
nobody would review as a copy decision.

**One catalogue covers both surfaces** — the Angular screens and the four NestJS
emails (Delivery, Reviewer queue notification, rejection, extraction failure). The
rejection email is the sharpest case for this: §10.3 gives it **no reason**, so its
wording is a decision, and outside the catalogue it changes as a template edit
nobody reviews as one. The former figure of *122 strings* predates the emails being
in scope and is **not** a count to check against; §17.1's gate compares the two
catalogues to each other.

Its job is **governance of the copy, not translation**: a change to a sentence
must appear as a change to these files, so a reviewer can see that a decision moved.
Sentences inside Angular templates make a copy change look like a template change,
and nobody reviews that as a decision change. `@angular/localize` was rejected
because it extracts strings *out of* templates, leaving the template as the source;
a hand-rolled typed module was rejected because the artifact stops being a
*document* domain people can read and check.

**Paraglide has no Angular support.** Its framework adapters are React, Solid,
Svelte and Vue, and no getting-started guide covers Angular, so the bundler plugins
are out of reach. The compiler runs as a pnpm `prebuild` and `prestart` step —
`paraglide-js compile --project ./project.inlang --outdir ./src/paraglide
--emit-ts-declarations` — with `--watch` as a second dev process. `src/paraglide`
is generated and gitignored, and **is deleted at the flip**: a stale outdir
surviving a `baseLocale` change renders Thai screens in English, which reads as an
incomplete translation rather than as a build fault.

> ⚠️ **Accepted cost: the Thai copy is unreviewable by domain readers until the
> staging deploy**, with the screens already built around the strings. This is the
> exact inverse of the cost this section formerly accepted. Mitigated by reviewing
> on the running UI rather than in JSON, and by **naming** the reviewers — the repo
> owner and colleagues — rather than hoping for them. §18.14.

> **The copy is normative; the appearance is not.** This is the exact inverse of
> §16.4's ruling on styling. A rule ("the rejection gives no reason") lets an
> implementer write a new sentence, and **the sentence *is* the decision.** Copy
> changes only by reopening the decision that put it there. **The Thai in
> `messages/th.json` is what ships**; the English beside it records what the
> sentence is *for* and is not itself a promise to anyone.

Load-bearing strings, each carrying a decision that exists nowhere else. **This
table names keys and the decisions they carry; the catalogues hold the prose.**
The Thai is authored at translation time and is deliberately not quoted here — a
third copy in this document would drift against two it cannot be checked against
(ADR 0010).

| Key | Carries |
|---|---|
| `requester_email_warning` | the only place a Requester is told the address is not checked; the email field's tooltip |
| `requester_retention_notice` | §12.9; the consent block's body |
| `requester_epidem_area_label` | the `epidem_chw_code` vs `chw_code` trap, made visible at the point of choosing — the address that answers *"cases reported in my provinces"*; under the Area switch in the map pane |
| `error_span_too_long_detail` | the 365-day cap, stated when a longer range is refused |

> **Amended 2026-09-30 to the locked Requester screens** (#123). Three keys were
> cut from the catalogue: `requester_gate_notice` (the approval gate, stated
> first), `requester_no_reason_notice` (the no-reason rejection, said up front),
> and `requester_span_cap_notice` (the cap as a standing notice; the calendar
> cannot offer a longer range, and the refusal above states the cap).
> `docs/design/brief.md` §5 records the same.

> **Amended 2026-10-05 in the Thai copy pass** (#96). `reviewer_no_email_edit_detail`
> was cut. The in-flight list still states the absence of an email correction
> (ADR 0017) in its heading, `reviewer_no_email_edit_heading`, but no longer gives
> the reason on screen. `email_rejection_no_reason` no longer says that no reason
> is given; it keeps only that a new request may be sent and is unaffected (§10.3).
> `requester_email_warning` keeps that the address is not checked, and no longer
> says that a typo goes unnoticed and the file will not arrive.
> `error_span_too_long_detail` no longer attributes the cap to upstream or says
> that the range is not split for the Requester; it states the cap as the
> system's. The behaviour is unchanged: a longer range is refused, never split.
> Six keys whose Thai was left empty were cut from both catalogues and their
> templates: `reviewer_signin_no_lockout`, `reviewer_alert_closed_set_note`,
> `email_queue_notification_note`, `reviewer_lookup_readonly`,
> `reviewer_lookup_events_note` and `requester_epidem_area_detail`.
> `reviewer_probe_failed_note` was cut: a failed row count shows the same note
> as a counted one, `reviewer_probe_note`.

### 16.4 The Requester page

**A single scrolling page**, not a wizard and not a two-column live preview. In
order: the title, the de-identification block, then the map-first split — the
Area on a map beside a form card holding the Disease group, the dates and the
contact fields — then the requirement checklist, the PDPA consent, and submit.

> **Amended 2026-09-30 to the locked Requester screens** (`docs/design/system.md`,
> ticket #123). The order used to open with an approval-gate notice; the repo
> owner cut it, with the 24-hour line and the no-reason notice, when the design
> was locked, and screen 1 no longer states the gate. The 24-hour promise is on
> the confirmation. The PDPA consent must be ticked before the check page opens;
> it is a client-side acknowledgement and is not posted.

> **Requirement, not styling: the de-identification block is open, above the form,
> not collapsed.** A Requester who never opens it receives a CSV with no names in
> it and files it as broken. *What you will and will not get is visible before any
> field is filled in, without interaction.* Since 2026-09-30 the block no longer
> carries a *no names is the correct result* line; its *not received* column,
> which is not styled as an error, carries that.

**Submit goes to a check page first**, decided with the repo owner 2026-09-18. It
restates the ask and the contact details, and shows the email address at the size
of a headline. (The warning that it cannot be changed after sending was cut from
the locked screen on 2026-09-30.) The
Requester sends from there or goes back to edit with everything kept. Since
[ADR 0017](adr/0017-a-reviewer-never-corrects-a-requesters-email-address.md) no one
can correct an address after submit, so this page is the last moment a typo can be
caught — and it is the only one.

> **A typo discovered after submit cannot be fixed in place.** The Requester
> cannot edit, and a Reviewer cannot correct the address. The Request runs its
> course, and if approved the Delivery goes to the address as typed. Accepted
> knowingly by the repo owner, 2026-09-18. Since 2026-09-30 the Requester **can**
> send a new Request with the corrected address: duplicate suppression (§4.8)
> matches on the email, so a different address is a different form.

**The confirmation page** carries the reference number, a restatement of the ask,
the 24-business-hour service promise, and the telephone number. It must read as
*you are done*, not as *something went wrong*.

> ⚠️ **This prototype settled structure, ordering and copy only. The repo owner
> supplies a wireframe during the dev cycle.** Visual design, spacing, typography
> and component choice are **not** settled here. Carry the two ordering rules
> (this one and §10.2's) as requirements, and treat the wireframe as the source of
> visual layout — an implementer must not read the prototype's styling as
> normative. Prototype:
> [`prototype/requester-reviewer-ui`](https://github.com/rawinan-soma/dds-sharing/tree/prototype/requester-reviewer-ui).
>
> The wireframe arrived: the design locked on 2026-09-30 in
> `docs/design/system.md`, drawn on the Lunagraph page "Requester form:
> prototypes", is the source of visual layout, and the Requester surface is
> built from it (#123).

**A worked example for acceptance testing:** seed the Reviewer queue with a
request that is genuinely hard to judge — an "independent researcher" on a
`gmail.com` address, beside a plainly legitimate DDC officer asking for a
full-year national `air-pollution` Extract (the largest Request that exists,
§7.9), a สคร. request, a hospital request, and one for `radiation` returning zero
rows. **A review screen is only judgeable against a request that is hard to
judge.**

---

## 17. Deployment and dev-cycle

### 17.1 Required tests

- **Reproducibility of the fingerprint.** Write the same rows twice on the CI
  host, assert one checksum, plus a fixture assertion that the bytes begin with
  the BOM and use CRLF. **Without this test, §8.2's rule 2 is a comment** — and
  the whole reason the fingerprint moved off the zip is that a non-reproducible
  hash went unread for weeks.
- **`diagnosis_icd10_list` quoting.** It is the only plausibly quotable column
  left, and **its delimiter is a comma** — confirmed 2026-09-04. **So every
  multi-code value in that column quotes**, and the CSV writer's quoting rule
  (§8.2 r5) is exercised on real data rather than theoretically. Pin it with a
  fixture: a row whose `diagnosis_icd10_list` holds two codes must round-trip
  through the writer as one quoted field. **A delimiter surprise splits a field
  into two and shifts every later column on that row — a corruption §7.5 counts
  rows, not columns, and so would not catch.**
- **The span builder is the only date arithmetic.** Assert that the Probe and the
  extraction job, given the same Request, produce byte-identical `start_date` and
  `end_date` — and that an inclusive `to` of 31 Dec yields an exclusive
  `end_date` of 1 Jan. This is the test that would have caught the 3,196-row loss
  (§7.2).
- **Completeness assert fires.** A Report code whose received count disagrees with
  `total_items` must fail the job and publish nothing.
- **The classification partitions the code list.** Assert that the groups in
  `docs/disease-groups.md` cover every Report code in the seed
  (`docs/research/003-disease-group-codes.md`) exactly once — none missing, none
  repeated. The check is three lines.

  > ⚠️ **This test compares the classification against the *seed*, so it is
  > structurally blind to a code that exists **upstream** and is missing from the
  > seed.** It previously claimed to be *"the only thing that will ever notice a
  > new upstream code arriving with no group"*; that claim was false and
  > [#33](https://github.com/rawinan-soma/dds-sharing/issues/33) is exactly the
  > case it could not see. **Nothing automated closes that gap**, and nothing
  > should pretend to: discovering a new upstream code means probing codes nobody
  > has enumerated, against a live token, in a test that would either hit
  > production upstream on every CI run or be permanently skipped. The honest
  > control is the periodic re-probe below, done by a human.
- **Periodic upstream domain re-probe — a documented operator task, not a test.**
  Once a year, or on any DDC announcement of new `รหัสรายงานโรค`, re-run
  [#33](https://github.com/rawinan-soma/dds-sharing/issues/33)'s probe: `page_size=20`,
  full-year span, one call per candidate code, reading `meta.total_items` only and
  **writing no response data to disk**. An unknown code returns `200` with
  `data: []` (§5.2), so absence and emptiness are indistinguishable — which is why
  this needs a human comparing against a DDC announcement, not a green build.
- **No case data in a log.** Run a job against the fake upstream harness (§17.3)
  with a **sentinel string planted in every fetched field**, capture everything
  the process writes to its log, and assert the sentinel never appears. Exercise
  it on the harness's **500 mid-loop, truncated page and auth-expiry** paths —
  those are exactly where an error handler reaches for the response body.
  **Without this test, §14.5's ban is a comment.**
- **The copy catalogues agree, and the Thai is Thai.** Assert that the key set of
  `messages/th.json` is identical to the key set of `messages/en.json`, and that
  **every value in `th.json` contains at least one character in U+0E00–U+0E7F**,
  against a short checked-in exemption list (the telephone number, `DDS`, email
  addresses). **Without this test, §16.3's maintenance rule is a comment** — and
  after the flip it is the *only* thing binding the two files, because `en.json`
  leaves `locales` and the compiler stops reading it. Paraglide has no strict mode,
  emits no warning for a missing message, and inlang's lint rules were removed in
  CLI v3, so a key missing from `th.json` reaches a Requester's screen as the raw
  key ([ADR 0010](adr/0010-copy-is-authored-in-english-and-translated-before-production.md)). **It must be green before the flip commit**, not after.
- **Large-download smoke test through the real ministry edge** — see §17.4.

### 17.2 Dev-cycle asks

| Ask | Why |
|---|---|
| ~~Measure the **`birth_date` null rate**~~ | **CLOSED 2026-09-04: estimated below 1% by the repo owner, which is not material.** The blank stands and `age_y` is **not** re-admitted to the allowlist. ⚠️ The figure is an **estimate, not a measurement** — if a real extract shows a material rate, the answer is still re-admitting `age_y` **as an allowlist change**, never a quiet fallback inside the derivation |
| ~~Confirm `diagnosis_icd10_list`'s **delimiter**~~ | **CLOSED 2026-09-04: it is a comma.** Every multi-code value quotes; see §17.1 |
| `SMTP_PORT`, `SMTP_PASS`, `FRONTEND_URL` | §11.2 |
| ~~**Three test sends**~~ | **CLOSED 2026-09-04 without being made.** The repo owner rules that the relay is in official organisational use and its mail is not filed as spam. Recorded as an accepted risk — §18.13 |
| ~~Confirm the relay hostname **verbatim**~~ | **Folded into the config supply above.** `uc-workd` is still close enough to a typo to be worth reading twice when `.env` is written — but nothing now tests it, §18.13 |
| The **wireframe** | §16.4 |

### 17.3 Build artefacts this spec requires

- The **static Thai/English Data dictionary CSV**, checked in, copied into every
  archive under a fixed filename. Its content is a build-time task, not a decision.
- The **province seed migration**, generated from `docs/provinces.csv`.
- **Host CLI commands**: Reviewer seeding / password reset / TOTP re-enrolment /
  deactivation (§17.5), the fingerprint verification command (§8.4), and the
  upstream traffic report (§13.6). None of them records who ran it
  ([ADR 0020](adr/0020-a-host-command-does-not-record-who-ran-it.md)).
- **A fake upstream dev harness.** Out of this spec's scope as a decision, but
  required to test §7.6 at all: it must expose a **500 mid-loop, a slow page, a
  truncated page, an auth expiry mid-job, and a `total_items` that shifts between
  attempts** — that last one is what tests the retry guard, and no fixture can
  produce it. **Standing constraint: no real patient data ever seeds it.**

### 17.4 The boundary, and who owns it

**The edge is ministry-managed and is not a dependency.** The design was made
robust to the worst plausible edge, so no setting on a proxy this project does not
administer is a precondition for the system working.

| Party | Owns |
|---|---|
| **DDC infra team** | the VM, and registering the public route through the ministry-managed edge. Its timeouts, buffering and body caps are shared ministry policy and are **not assumed tunable** |
| **Service owner** | the VM's Docker host, the application, everything from the TCP connection inward, and sign-off on what is published |

**The edge is irrelevant, and the reason is now simply size.** The largest archive
this service can produce is **tens of KB** (§5.3, §8.1) — below any plausible proxy
body cap, buffer or timeout. Range-request support on the download endpoint and an
explicitly configured base URL are retained, both cheap and both still correct.

⚠️ *The former argument — "~20–30 MB on the wire, never the ~200 MB Extract" — was
sized on out-of-scope code `02` and is withdrawn
([#33](https://github.com/rawinan-soma/dds-sharing/issues/33)). The conclusion
survives; only the margin changed, from comfortable to overwhelming.*

**Kill switch: `docker compose down` on the VM — minutes, not the edge team's
queue.** This is the escape hatch for §18's residual risk. Removing the *route* is
the edge team's and is slower, but is not needed to stop serving data.

**Infra is told what this publishes.** The VM request states that the service is
internet-facing and serves case-level (de-identified) DDS surveillance data. A VM
granted under "internal tool" assumptions is a mismatch that surfaces at the worst
moment, and the person granting it carries part of §18's question knowingly.

**Requested, and verified after the fact — not preconditions:**

- A public DNS name under `moph.go.th` and a **ministry-issued TLS certificate**,
  both requested from and renewed by the ministry. **Production only** — no public
  hostname or ministry certificate exists during development, which is why the
  base URL is configuration.
- A route from the edge to the VM's app port, and **only** that port.
- **Confirmation, tested from outside, that Postgres, Redis, MinIO and the worker
  are unreachable.** MinIO is the sharp one: a bucket reachable directly would
  bypass the Download token and the download audit entirely.
- ⚠️ **Internal reachability is a separate and larger question than internet
  reachability.** Being unreachable *through the edge* is automatic; being
  unreachable from any DDC desktop is not. **Host-level firewalling is required,
  not just edge routing.**
- ⚠️ **An end-to-end download smoke test as a first-deploy gate** — the route, the
  token and the archive through the real ministry edge, which is the last thing
  testable since no public route exists before production. **Exercise it
  deliberately on first deploy rather than discovering it through a Requester.**
  *No longer a **large**-download test: at tens of KB there is nothing to stress.
  The gate is about the path existing, not the payload surviving.*
- ⚠️ **NTP sync on the Docker host.** TOTP is clock-dependent and there is no email
  reset path, so **drift beyond ~30 seconds locks out every Reviewer
  simultaneously**, and the only fix is shell access to the machine that is
  broken. The ±1-step window absorbs ordinary drift; nothing absorbs an unsynced
  clock. NTP is doubly required because the business-hours clock and the
  UTC-stored/ICT-rendered timestamps are unauditable across a drifting clock.
- **The province seed migration must run, and its startup assert must be treated
  as a boot failure, not a warning** (§6.4).

### 17.5 Reviewer accounts

- **Username + password + TOTP (Google Authenticator).** Per-Reviewer **named**
  accounts, **minimum two**, seeded by a CLI command on the Docker host.
- **Password: 12–20 characters**, at least one uppercase, one digit, one special
  character, hashed with **argon2id**. **No expiry, no rotation.** Rules are
  enforced **server-side in one place**, hit by both the CLI and the change form.

  > ⚠️ **The 20-character ceiling is deliberate, not an oversight.** It rules out
  > passphrases and truncates what a password manager would generate, which makes
  > the **ceiling — not the floor — the binding constraint on password strength.**
  > It was chosen knowingly with a TOTP second factor in place. Do not "fix" it
  > without reopening the decision.

- **Seeding ceremony.** The CLI generates a random compliant password and prints
  it **once**, alongside a terminal QR for the TOTP secret — the Reviewer is
  present or on a call. It also prints §12.9's retention notice. **First login
  forces a password change**, and the account cannot approve anything until one
  TOTP code has confirmed enrolment. **A seeded-but-unconfirmed account is inert.**
- **Self-service password change while authenticated**, requiring the current
  password plus a fresh TOTP code. Not a bypass — it needs a live session and both
  factors — and without it, "I think someone saw me type it" has no answer short
  of reaching the host.
- **No self-service email reset, ever.** That would let the Requester's
  unverified-email world reach the privileged surface. Password reset and TOTP
  re-enrolment go through the CLI, which requires shell access — the correct bar.

> ⚠️ **No lockout at all — throttling only.** Exponential backoff **per account and
> per IP**, capping around 30 seconds, **state in Postgres so it survives a
> restart**. A used TOTP code cannot be replayed; the validation window is ±1 step.
>
> Recorded because this contradicts the reflex: **a lockout an anonymous internet
> stranger can trigger against a named account *is* the denial of service.** With
> two Reviewers, a business-hours expiry clock, and Reviewer unavailability
> converting into expired Requests, an aggressive lockout attacks availability far
> more reliably than it defends the password. At one attempt per 30 seconds an
> attacker holding the correct password still cannot brute-force six digits.

**Password and TOTP are submitted on ONE form and checked together**, and failed
sign-ins return **one generic message** (`ข้อมูลเข้าสู่ระบบไม่ถูกต้อง`). A two-step
form tells an attacker when the password is right, which is the signal that makes
attacking the second factor worthwhile. **The audit record keeps which factor
failed; the screen does not.**

> ⚠️ **No TOTP recovery codes.** They are a written-down second-factor bypass for a
> data-release surface, and the artifact most likely to end up photographed or in a
> notes app. **The second Reviewer is the recovery mechanism** — that is what the
> two-account minimum was always for. **The minimum is two *reachable people*, not
> two rows in a table.** Losing both phones at once requires shell access.
>
> **Enforced, not documented: the CLI refuses to deactivate below two active
> Reviewers**, overridable only by an explicit `--force` that prints what it is
> breaking. A minimum living only in prose is gone on the day it is needed.

**Deactivation is `deactivated_at`, never a row deletion.** It invalidates live
sessions immediately (a Postgres query) and is itself a recorded event — recording
what happened to the Reviewer, never who ran the command
([ADR 0020](adr/0020-a-host-command-does-not-record-who-ran-it.md)). Shell access
to the Docker host is the bar; no argument, environment variable or host login
name is read or recorded. **The display name stays on every Decision, permanently**
— `display_name` must be the person's real name, and the CLI prompts for it
deliberately rather than deriving it from the username, because it is unerasable.

**`reviewer.email` is for queue notification only** — never for password reset.

---

## 18. Accepted risks

Every item here was put explicitly, weighed, and adopted. **None is an oversight.**
A specification that hides its trade-offs cannot be reviewed by whoever owns the
DDC data agreement, and this section exists so it can be.

### 18.1 The lawful basis is legal obligation; the PDPO was consulted — REVISED 2026-09-04

**The repo owner has consulted the DDC PDPO, and the position is that the release
rests on legal obligation** — the department's own duty to run occupational and
environmental disease surveillance under
**พ.ร.บ.ควบคุมโรคจากการประกอบอาชีพและโรคสิ่งแวดล้อม พ.ศ. 2562**. This **supersedes
the previous entry**, which said no basis existed and none had been sought.

**Two things it does not do, stated so the difference is not lost:**

- **It does not put a written ruling in this repository.** The consultation
  happened; **no artefact of it is checked in** — no dated memo, no named PDPO
  officer, no cited section. A basis nobody can produce on request is
  indistinguishable, to a later reader or an auditor, from one that was never
  obtained. **The gap is now the record, not the position.**
- **It does not make the approval gate a lawful basis.** The gate remains an
  **accountability record**. That distinction was the sharpest thing the old §18.1
  said, and it survives the revision intact — *precisely because the gate feels
  like it closes the compliance hole, and that feeling is how the paperwork never
  gets chased.*

**What is unchanged:** the de-identification argument the position rests on —
`tmb_code` and `epidem_tmb_code` are dropped, so the repo owner's reading is that
the Extract is non-personal data. **The risks that reading carries are §18.2 and
§18.3, and neither is softened by the consultation.** §18.2 in particular says the
finest *named* geography is not the finest *effective* geography, which is an
argument about whether the de-identification claim holds at all.

**Reversal path is unchanged:** the kill switch (§17.4), then a fresh effort
against a redrawn scope.

### 18.2 The Extract's finest *named* geography is not its finest *effective* geography

> **The column a privacy officer named was removed; the precision she objected to
> substantially remains, for a minority of rows, by a public route she may not have
> been shown.**

`hospital_code` is retained. A **รพ.สต.** (โรงพยาบาลส่งเสริมสุขภาพตำบล) serves
exactly one subdistrict, so for any case one reported, the reporting facility
identifies the tambon as surely as the dropped `tmb_code` did — **and the MoPH
facility register that makes this readable is published openly** (§6.7). No special
access is required.

**The narrowness cuts both ways.** The share of such rows is small — occupational
and environmental disease is diagnosed where diagnostic capacity is, which is
hospitals. But **small groups are precisely where re-identification bites**, and
full `birth_date` sits beside them. The profile changed from *many rows, moderate
exposure* to **few rows, high exposure each** — not to *no risk*. A spec reporting
only the first half of that sentence would be misleading.

**PDPA identifies data as personal when a person can be identified indirectly**,
not only directly. This and §18.3 are the indirect route. Nobody with authority on
that question has been asked, by decision.

### 18.3 Full `birth_date` is retained beside a derived age that already replaces it

`onset_age` ships in every Extract, so the substitute exists and is unused.
**DOB coarsening has been offered and declined four times.**

`{amp_code, gender, birth_date, onset_date}` is a live quasi-identifier. Against
district-level geography this sits near Sweeney's {place, sex, DOB} uniqueness
rate — a real improvement over the tambon-level {ZIP, sex, DOB} rate, and still
not anonymous.

**The realistic adversary is the employer.** These are EnvOcc groups — pesticide
poisoning *ขณะทำงานเพื่อรายได้*, pneumoconiosis, asbestos mesothelioma, lead,
radiation. The party with the strongest motive to re-identify holds the auxiliary
data (workers' dates of birth, district of residence, treating facility) that
converts these rows to names with near-certainty. Group `210` covers poisoning in
*เรือนจ้าง/หอพัก* — worker dormitories. **This data is the evidentiary basis for
compensation under พ.ร.บ.ควบคุมโรคจากการประกอบอาชีพและโรคสิ่งแวดล้อม พ.ศ. 2562.**

**Mitigations that remain available and unused**, each a single configuration
point: coarsening `birth_date` to `onset_age` alone, coarsening geography to
province, and a two-tier open/gated split.

### 18.4 Upstream is authenticated; this service is not

Every documented path into the upstream DDC system that holds DDS is MoPH
account / Provider ID RBAC — **no anonymous access**. The Requester surface here
has no login. As specified, this service is a weaker door onto records derived
from the same source, with a human gate in front of it rather than a credential.

### 18.5 Excel silently corrupts a leading-zero `hospital_code` — NARROWED 2026-09-04

⚠️ **This risk was stated far too widely, and its headline example does not
exist.** It named `chw_code`, `amp_code`, `epidem_chw_code` and `epidem_amp_code`
as *"zero-padded numeric strings"* and offered `01` → `1` as the corruption.
**There is no province `01`.** The domain runs **10–96** and the hierarchy is
prefix-nested at 2 / 4 / 6 digits, so no geography code in this Extract can begin
with a zero — §4.6 said so all along, and this section contradicted it.
Re-verified 2026-09-04 against the reference data: **zero codes beginning `0`
across 77 provinces, 929 districts and 7,451 subdistricts.**

**What survives is `hospital_code` alone.** It is a facility code, it is not
prefix-nested, and **this repository deliberately holds no facility list** (§6.7,
X14) — so whether any in-scope `hospital_code` begins with a zero is **not
verifiable from this repo**, and is not asserted here either way. If one does,
Excel-on-double-click parses it as a number and quoting does not prevent it;
Excel's type inference ignores quotes.

> **Why the correction is recorded rather than applied silently.** This is the
> **second** Excel risk in this register found to rest on an unmeasured premise —
> §18.6 was the first. Both were accepted because the mechanism sounded right, and
> in both cases nobody checked the data that was already in the repository. **The
> reasoning that accepts these is the thing to distrust, not the individual
> claim.**

Two repairs were considered and rejected: **`="01"` formula-escaping** corrupts the
file for pandas in order to fix it for Excel, and puts a formula-injection vector
into a file handed to strangers; **shipping `.xlsx`** contradicts the one-flat-CSV
decision and cannot hold the worst case anyway — the format's own ceiling is the
same 1,048,576 rows. *(That last objection no longer bites — no Extract comes near a
million rows, §18.6 — but the first two stand on their own and the one-flat-CSV
decision is unchanged.)*

> **Note the interaction with the audience: the BOM exists to get these users into
> Excel, and Excel is where the corruption happens.** We are optimising for the
> tool that breaks the data, knowingly, because the alternative is mojibake for
> everyone.

### 18.6 ~~Excel silently truncates above 1,048,576 rows~~ — RETIRED 2026-09-02

**This risk does not exist and never did.** It read: *"see §8.1 — a full-year group
`02` Extract loses roughly 95,000 rows with no error."* Report code `02` is out of
this service's scope ([#33](https://github.com/rawinan-soma/dds-sharing/issues/33));
the largest Extract this service can produce is **1,952 rows**, and the entire
25-code domain over a full year is **3,861** (§5.3).

Kept as a struck entry rather than deleted, because the reasoning that accepted it
— *"anyone pulling a 1.14 M-row Extract is working in R or Python"* — was a
judgement about the audience, and someone will make it again. **It was accepted on
a volume nobody had measured.** The lesson is the entry; the risk is gone.

### 18.7 The record cannot say which Request produced an Extract

See §8.4. A content hash narrows to a *set*. **No identifying mark is added.**

### 18.8 Email delivery is unobservable

See §11.1. The sharpest instance: a failed **Reviewer queue notification** means
the approval gate has no trigger and a Request expires at 24 business hours
**through nobody's fault**. That is why it banners on the first failure.

### 18.9 The audit record is permanent personal data, including about people who never used the service

See §12.7. Four bodies, kept indefinitely, on a legal-obligation and
legitimate-interest basis. The longest-lived body is `token_lookup` rows matching
no Request — IP addresses of anonymous strangers.

**"Data does not linger" is true of the Extract and false of everything else.**

### 18.10 The record measures named staff, permanently

Four distinct instances, treated as one class in §12.7:
`expired{business_hours_elapsed, reviewer_accounts_active}`, the login and
failed-login stream, `collection_lapse_cleared` (a permanent record that a named
Reviewer did *not* chase a lapse), and `extraction_alert_cleared` (which
Reviewer's approvals failed and how fast they responded).

**They are told, at seeding and first login** (§12.9) — not in this document,
which they will never read.

### 18.11 The record exists on one disk, and there is no copy of it

**Backup, restore and disaster recovery were put explicitly and declined**
(2026-09-04). The audit record is permanent and append-only (§12.7) and is **the
thing this project was built to create** — the charter's first problem is that
today *"there is no record of who released what to whom"*. It lives in one
PostgreSQL, on one Docker host (§17.4).

**A disk failure destroys every Decision, Snapshot and Extract fingerprint the
service has ever made.** No **Reviewer**'s accountability survives it, and nothing
reconstructs it: the **Extracts** were deleted at 72 hours by design, and upstream
holds no copy of what was released from here.

> **What makes this exposure unusual is its shape.** Everything else in this
> design that could be lost was made disposable on purpose — the **Extract
> archive** at 72 hours (§9.3), Redis by AOF replay (§7.7). **The record is the
> one thing that was made permanent, and it is the one thing with no second
> copy.** Accepted, and recorded here rather than left as an assumption that a VM
> snapshot exists somewhere.

**Reversal path:** a scheduled `pg_dump` and one tested restore. Neither is in
scope. See SRS §1.3.2 X17 and §6.3 R17.

### 18.12 There is no stated accessibility target, and no way back out of a bad release

Two further declines of the same date, recorded so they are not mistaken for
oversights:

- **No accessibility conformance target** (TWCAG / WCAG) is claimed for either
  surface. This is **not** deferred to the wireframe (§16.4) — it is declined, so
  nobody reopens it as a styling detail. The audience is government officers on
  ordinary connections, and some of them will not be able to use this service.
  *(2026-09-18: the design is built to WCAG 2.1 AA as a working standard, still
  with no claim; `docs/design/accessibility.md` is the audit.)*
- **A Reviewer who zooms past about 125% is locked out.** The Reviewer surface is
  desktop-only from 1024 CSS px (§16.1), and browser zoom shrinks CSS pixels: at
  125% a 1280px laptop is 1024 wide, at 150% it is 853, and the Reviewer sees
  *open this on a computer* on a computer. Stacking the layout below 1024 instead
  of blocking was offered and declined by the repo owner, 2026-09-18. This fails
  WCAG 1.4.4 and 1.4.10, and it is accepted knowingly.
- **No redeploy, rollback or dependency-patching procedure.** §17.4 gates the
  **first** deploy thoroughly and says nothing about the second. A bad release is
  backed out by whatever the operator improvises, and the service is
  internet-facing with no patching cadence on record. §17.4's `N=1` worker plus
  the §7.7 reconcile means an in-flight job survives a restart — that is a
  property inherited from durability work, not a deployment design.

### 18.13 Delivery is believed rather than demonstrated


**The three test sends were not made.** §17.2 asked for one message to a Reviewer
mailbox, one to an external Gmail address and one to the bounce destination, each
confirming *where it landed*. **On 2026-09-04 the repo owner closed the question
on organisational knowledge instead: the relay is in official use and its mail is
not filed as spam.**

⚠️ **The same day it was also confirmed that no user of this service reads mail
at `moph.go.th`.** That removes the one leg where organisational standing would
have carried weight: **100% of this service's mail now crosses to public
providers**, which apply their own filtering and have never seen this sender. The
closing argument and the recipient list were given together; **the argument covers
none of the remaining traffic.** Recorded, not reconciled — the decision to close
stands as the repo owner's.

**What is therefore not on record:**

- Whether `ddc.mail.go.th`'s **SPF record authorises `mailrelay.uc-workd.com`**,
  and whether **DKIM signs for the sender domain**. One `dig` would have answered
  both. Gmail evaluates these independently of any arrangement inside DDC.
- Whether a **Reviewer's own mailbox** accepts the queue notification. Since no
  Reviewer is on `moph.go.th`, this is a public-provider delivery like any other,
  with the failure at §18.8.
- Whether the **relay hostname is correct verbatim** — `uc-workd` was flagged as
  close enough to a typo to warrant one deliberate check. That check now rides on
  the config being supplied correctly (§11.2) with nothing behind it.
- That **any message from this service has ever arrived anywhere.** The first
  real Delivery will be the first test.

> **The cost is bounded and already has machinery behind it.** A Delivery that is
> junked, or never sent because the hostname is wrong, produces a **Download
> token** that expires unused in 72 hours — invisible to the system by premise
> (§11.1), surfacing only as a **Collection lapse** Alert after 24 business
> hours (§10.6). **The failure is recoverable and someone is told; it is simply
> told late, and told about a Requester who already waited.**

> **The sharper instance is the Reviewer queue notification**, not the Delivery:
> if that message is junked, the approval gate has no trigger and a Request
> expires at 24 business hours through nobody's fault (§18.8). **That path has no
> Requester to notice it.**

**Reversal is one email.** Nothing here is designed around the belief; if a send
is ever made and lands wrong, this section is deleted rather than worked around.

### 18.14 Smaller accepted costs, recorded so they are not rediscovered

- **`/health` is unauthenticated and leaks service state** (§14.1).
- **No checksum covers the upload to MinIO** (§8.4).
- **The 72-hour clock can elapse unnoticed** because job completion is an event the
  Requester never sees (§9.3).
- **Bounce detection is lost** because there is no receipt email (§11.3).
- **The Thai copy is unreviewable by domain readers until the staging deploy**,
  with the screens already built around the strings (§16.3, [ADR 0010](adr/0010-copy-is-authored-in-english-and-translated-before-production.md)). This
  replaced the inverse cost — an implementer unable to read a Thai-only catalogue —
  which no longer applies.
- **The Requester loses the reference number if they close the confirmation tab**,
  until the Decision email arrives (§12.5).
- **`cid` is not a stable person key**, so repeat-patient detection and
  de-duplication are impossible from this feed (§6.5).
- **A public holiday counts as business time** — the clock skips weekends only
  (§15.2, ADR 0021).

### 18.15 The Pilot releases real data from personally owned hardware, and keeps no record it must

The **Pilot** ([ADR 0022](adr/0022-a-disposable-pilot-releases-real-data-from-the-repo-owners-server.md))
is a second service beside Production: the repo owner's own server, open to the
internet through a Cloudflare Tunnel on the owner's domain, calling the real
upstream and releasing real Extracts under the same approval gate. Accepted by
the repo owner 2026-09-28, with these costs stated rather than discovered:

- **Case-level DDS surveillance data is fetched onto personally owned hardware**,
  outside the department's infrastructure, and kept there for up to 72 hours per
  Extract. §18.1's basis was argued for a department-run service; the PDPO is to
  be told of this one the same way.
- **Cloudflare terminates TLS**, so Requester contact details, Reviewer sign-ins
  and Extract downloads pass through a third party in clear.
- **The department's upstream token and relay credentials live on that server.**
- **The Pilot's record is disposable.** There is no backup and no export: a lost
  disk loses every Decision the Pilot made, and a release from the Pilot may leave
  no trace anywhere. §12's promise that a release can be read back years later
  holds for Production only.
- **One Reviewer at launch.** The two-Reviewer minimum is unmet until a colleague
  is seeded; recovery meanwhile is shell access to the server.

**Reversal:** `docker compose down` on the Pilot's server, then deleting its
volumes — which, the record being disposable, is also the whole of retiring it.

---

## 19. Where each requirement was decided

| Area | Section | Ticket |
|---|---|---|
| Upstream API behaviour, timings, failure taxonomy | §5 | [#4](https://github.com/rawinan-soma/dds-sharing/issues/4) |
| Disease group codes | §4.1 | [#3](https://github.com/rawinan-soma/dds-sharing/issues/3), `docs/research/003-disease-group-codes.md` |
| Disease group as a family of Report codes; the async Probe; column 2 | §4.9, §5.4, §6.2, §7.2, §8.1 | [#30](https://github.com/rawinan-soma/dds-sharing/issues/30), [ADR 0006](adr/0006-a-disease-group-is-a-family-of-report-codes.md) |
| Network reach; de-identification as the standing control | §3.1 | [#13](https://github.com/rawinan-soma/dds-sharing/issues/13) |
| The allowlist and its rules 1–5 | §6.1, §6.5 | [#2](https://github.com/rawinan-soma/dds-sharing/issues/2) |
| The 63-field inventory; no fixed upstream schema | §6.6 | [#14](https://github.com/rawinan-soma/dds-sharing/issues/14) |
| Geographic granularity after the tambon ruling | §6.2 | [#21](https://github.com/rawinan-soma/dds-sharing/issues/21) |
| `hospital_code` raw, no facility list | §6.7, §18.2 | [#23](https://github.com/rawinan-soma/dds-sharing/issues/23) |
| Rule 6, the `project` stage, both derived columns, the province table | §6.1, §6.3, §6.4, §7.4 | [#24](https://github.com/rawinan-soma/dds-sharing/issues/24), [ADR 0002](adr/0002-derived-extract-columns-anchored-to-the-case.md) |
| Request parameter surface, the 365-day cap, `epidem_chw_code` | §4 | [#7](https://github.com/rawinan-soma/dds-sharing/issues/7) |
| Region vocabulary and the province table | §4.5, §4.6 | [#15](https://github.com/rawinan-soma/dds-sharing/issues/15) |
| Rate limiting, `N=1`, the Probe, disk formula, Non-goals | §13 | [#5](https://github.com/rawinan-soma/dds-sharing/issues/5) |
| Probe granularity; the Reviewer's gate is identity, not size; no Probe stalled Alert | §5.4, §10.2, §10.6, §12.3, §13.3 | [#31](https://github.com/rawinan-soma/dds-sharing/issues/31), [ADR 0007](adr/0007-the-reviewers-gate-is-identity-not-size.md) |
| Extraction pipeline, retry, completeness, stall | §7 | [#8](https://github.com/rawinan-soma/dds-sharing/issues/8), superseded in part by [ADR 0008](adr/0008-the-pipeline-is-sized-for-one-page.md) |
| CSV writer's eight rules, Data dictionary, Excel leading zeros (narrowed to `hospital_code` 2026-09-04) | §8.2, §18.5 | [#25](https://github.com/rawinan-soma/dds-sharing/issues/25) |
| The fingerprint, archive naming, verification command | §8.3, §8.4 | [#29](https://github.com/rawinan-soma/dds-sharing/issues/29), [ADR 0005](adr/0005-the-fingerprint-covers-the-extract-not-the-archive.md) |
| Delivery, the token, the 72 h clock, deletion, the expiry page | §9 | [#9](https://github.com/rawinan-soma/dds-sharing/issues/9) |
| Audit record shape, tables, roles, catalogue | §12 | [#10](https://github.com/rawinan-soma/dds-sharing/issues/10) |
| Reviewer accounts and sessions | §10.5, §17.5 | [#18](https://github.com/rawinan-soma/dds-sharing/issues/18) |
| Scheduled work, derived expiry, heartbeat | §15 | [#20](https://github.com/rawinan-soma/dds-sharing/issues/20) |
| Mail configuration and the unobservability premise | §11.1, §11.2 | [#17](https://github.com/rawinan-soma/dds-sharing/issues/17) |
| Send failure vs collection lapse, Alerts, resend rules | §11.3–§11.5, §10.6, §10.8 | [#19](https://github.com/rawinan-soma/dds-sharing/issues/19), [ADR 0001](adr/0001-email-delivery-is-unobservable.md), [ADR 0017](adr/0017-a-reviewer-never-corrects-a-requesters-email-address.md) |
| A Reviewer sees contact details, the record does not | §10.9, §12.3 | [ADR 0015](adr/0015-the-record-is-contact-free-the-screen-is-not.md) |
| A lapsed Download token is terminal; no grace window | §9.3, §9.4, §10.9, §11.5 | [ADR 0016](adr/0016-a-lapsed-download-token-ends-the-request.md) |
| `/health`, two watchers, Re-run, Bull Board, disk thresholds | §10.7, §14 | [#27](https://github.com/rawinan-soma/dds-sharing/issues/27) |
| Retention of personal data | §12.7–§12.9 | [#28](https://github.com/rawinan-soma/dds-sharing/issues/28), [ADR 0004](adr/0004-personal-data-is-retained-indefinitely.md) |
| Ingress boundary, ownership, kill switch, deployment requests | §17.4 | [#16](https://github.com/rawinan-soma/dds-sharing/issues/16) |
| SPA shape, routes, `/reviewer`, Thai-only, Paraglide | §16.1–§16.3 | [#26](https://github.com/rawinan-soma/dds-sharing/issues/26), [ADR 0003](adr/0003-plain-spa-and-a-collection-path-that-bypasses-it.md), [ADR 0010](adr/0010-copy-is-authored-in-english-and-translated-before-production.md) |
| UI structure, ordering rules, copy as deliverable | §10.2, §16.4 | [#11](https://github.com/rawinan-soma/dds-sharing/issues/11) |
| PDPA position ruled out of scope; the five carried risks | §18.1–§18.4 | [#22](https://github.com/rawinan-soma/dds-sharing/issues/22) |
| Fake upstream harness requirements | §17.3 | [#6](https://github.com/rawinan-soma/dds-sharing/issues/6) |
| The Probe's bounded end — retries, `probe_failed`, the skipped disk pre-check | §5.4, §7.8, §10.2, §10.6, §12.3, §12.4, §13.6 | audit of this document against all 30 tickets, 2026-09-02 |
| Scope is the 25 EnvOcc codes; the whole sizing model re-anchored on measured volumes | §4.9, §5.3, §7.2, §7.9, §8.1, §13.3, §13.5, §16.4, §17.1, §17.4, §18.5, §18.6 | [#33](https://github.com/rawinan-soma/dds-sharing/issues/33) |
| §7 sized for one page: no chunking, no disk projection, no drain estimate; the shared span builder | §5.3, §5.4, §7.2, §7.5, §7.6, §7.8, §7.9, §8.1, §10.2, §12.3, §12.4, §13.3, §13.6, §14.3, §17.1 | [#34](https://github.com/rawinan-soma/dds-sharing/issues/34), [ADR 0008](adr/0008-the-pipeline-is-sized-for-one-page.md) |
| The Reviewer's identity test, and the telephone call on uncertainty | §10.3, SRS FR-09, `CONTEXT.md` *Decision* | repo owner, 2026-09-04 |
| Application logs: no case data, ever; 72-hour lifetime as the backstop | §14.5, §17.1, SRS NFR-34 | repo owner, 2026-09-04 |
| No database backup; no accessibility target; no rollback procedure; no in-app amendment of the classification | §18.11, §18.12, SRS X17–X20, R17 | repo owner, 2026-09-04 |

> ✅ **The worst-case row volume is now settled, and it is small.** All 25 in-scope
> Report codes were probed over a full year on 2026-09-02
> ([#33](https://github.com/rawinan-soma/dds-sharing/issues/33)): **3,861 rows for
> the entire domain, 1,952 for the largest single Request.** The former model —
> ~1.14 M rows from Report code `02` — was measured on a code in the general D506
> notifiable-disease block, **outside this service's scope**, and every figure
> resting on it has been withdrawn: §7.9's 10–25 minutes, §13.5's ~5 GB, §17.4's
> 20–30 MB edge argument, §18.6's Excel truncation.
>
> **§7 has since been simplified to match, and that question is now closed.**
> [#34](https://github.com/rawinan-soma/dds-sharing/issues/34) removed date-chunking,
> the disk projection and the drain estimate, leaving **one upstream call per
> Report code**: the widest Disease group is ten calls and ~35 seconds. What
> survives — the pagination loop, the completeness assert, the retry discipline —
> survives on **correctness**, never on load. The boundary agreement that chunking
> used to provide is now held by a shared **span builder** that the Probe and the
> job both call. See [ADR 0008](adr/0008-the-pipeline-is-sized-for-one-page.md).
>
> ⚠️ **The standing constraint outlives the cleanup: do not justify any of this
> apparatus by load, and do not let a later reader infer a load that has never
> existed.**
