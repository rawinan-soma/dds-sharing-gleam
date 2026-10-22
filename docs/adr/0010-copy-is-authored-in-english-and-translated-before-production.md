# 10. The copy is authored in English and translated before production

Date: 2026-09-04

## Status

*Amended by [ADR 0024](0024-copy-through-g18n-behind-a-strict-wrapper.md): in
the Gleam app, copy goes through `g18n` behind a wrapper that fails on a missing
key, and the Paraglide consequences below do not apply. The decision is
unchanged.*

Accepted. **Reverses** `docs/spec.md` §16.3's ruling that `messages/en.json` is
deliberately not maintained, and rewrites that section. Amends §12.9, §17.1 and
§18.14. Leaves [ADR 0003](0003-plain-spa-and-a-collection-path-that-bypasses-it.md)
untouched — the SPA shape and the Thai-only screen are unaffected. Decided with
the repo owner on 2026-09-04.

## Context

§16.3 ruled Thai-only and single-locale — `baseLocale: "th"`, `locales: ["th"]`,
`messages/en.json` deliberately absent — on the ground that several Thai sentences
are **the only record of a decision this design made in prose**, and that a second
language means writing every promise twice, with the two versions drifting into
two different promises. Its accepted cost was that an implementer who does not
read Thai cannot read the catalogue, *"mitigated by descriptive English keys, not
eliminated."*

Two facts were not available when that was decided.

**The catalogue cannot carry English context.** The inlang message format has no
description or comment field, and this is not a gap in the plugin: the inlang SDK
data model is three tables — `{id, declarations}`, `{id, bundleId, locale,
selectors}`, `{id, messageId, matches, pattern}` — with no metadata column
anywhere, so no plugin could round-trip one. Comments in source formats (Apple
`.xcstrings`, i18next `_comment`) are dropped on import, and a sibling key such as
`greeting_note` is not inert metadata — `importFiles` treats every non-`$schema`
key as a message, so it compiles into a real `m.greeting_note()` and ships in the
bundle. **"Descriptive English keys" is therefore not a mitigation that can be
strengthened. It is the ceiling.** The only per-string English the toolchain will
carry is a second locale file.

**The copy is authored by an agent, and agent-written Thai reads badly.** English
is where the agent is competent; Thai is where a human must be. §16.3 assumed the
catalogue would be hand-authored in Thai from the start, which serialises every
screen in the build behind one person's writing.

## Decision

**Copy is authored in English into `messages/en.json`, translated into
`messages/th.json` by the repo owner and named colleagues, and the locale flips to
Thai before production. Only Thai is ever served.**

- **Development**: `baseLocale: "en"`, `locales: ["en", "th"]`. Thai lands
  progressively and an incomplete `th.json` degrades to readable English.
- **Production**: `baseLocale: "th"`, `locales: ["th"]`,
  `strategy: ["cookie", "baseLocale"]`.
- `messages/en.json` is **checked in and permanently maintained**. A copy change
  touches both files in the same commit.
- The catalogue covers the Angular surfaces **and** the four NestJS emails.

**The reversal rests on one narrowing, and nothing else.** §16.3's two-promises
argument was about ***served*** languages. With `locales: ["th"]` at production and
no `url` strategy, English has no route to a screen — no language prefix, no query
parameter, no switcher, nothing a cookie can select. A drift between `en.json` and
`th.json` is a **documentation defect**, not a broken promise to a Requester. That
is a different and much smaller hazard than the one §16.3 refused, and it is the
only ground on which this ADR overrules it.

### Considered options

- **Keep §16.3 as it stands** and hand-author Thai from the start. Rejected: it
  blocks every screen on one person's writing, and the agent cannot proceed
  without a string to place.
- **Author English *values* into `messages/th.json` and overwrite them in place**,
  never changing `settings.json`. **Strictly safer** — no `baseLocale` flip, so no
  fallback inversion and no raw-key exposure, and the completeness gate becomes a
  script that cannot silently pass. Rejected for one reason: the English does not
  survive the overwrite, and the English *is* the deliverable here. It is the
  right answer for anyone whose problem is only that the agent needs a placeholder.
- **Two catalogues and a real flip.** Chosen, with the exposure below accepted and
  gated.

### The six load-bearing sentences

§16.3's table quoted four Thai sentences and §12.9 a fifth. They are **re-authored
in English and translated fresh**, not copied forward. Copying them verbatim was
put and declined: the round trip is a real fidelity risk, but keeping the spec's
Thai normative alongside a maintained `en.json` and a shipped `th.json` puts three
copies of every promise in the repo, and three-way drift is worse than the two-way
drift already accepted above. §16.3 and §12.9 therefore stop quoting Thai prose and
name message keys instead.

## Consequences

- **The failure mode inverts at the flip, and gets worse.** Today a key missing
  from `th.json` renders English. After the flip `fallbackMap["th"]` is
  `undefined`, so a missing key renders **the raw message key** — `calm_green_otter`
  on a Requester's screen — silently, at compile time, with no diagnostic. The
  degradation goes from *readable wrong language* to *developer identifier leaked
  to a user*.
- **Nothing in the toolchain catches it.** There is no strict mode and no
  compiler warning; the CLI has exactly two commands, `compile` and `init`; and
  inlang's lint rules were removed in CLI v3 pending a reintroduction that has not
  happened. §17.1 carries the gate because no dependency will.
- **`en.json` goes untooled at the flip.** Dropping `en` from `locales` means the
  compiler never reads `messages/en.json` again — not compiled, not validated,
  invisible. **The §17.1 parity check is therefore not a safety net on top of the
  toolchain; it is the only thing binding the two files together**, and the
  "permanently maintained" rule above rests entirely on it.
- **The `url` strategy must never be added.** Default `urlPatterns` leave the base
  locale unprefixed and prefix every other one, so adding `url` would make
  `baseLocale` decide which language owns `/` — reintroducing the language prefix
  §16.3 forbade, as a side effect of a strategy change nobody would review as a
  copy decision. `strategy: ["cookie", "baseLocale"]` involves no URL at all.
- **The accepted cost inverts** (§18.14). It is no longer the implementer who
  cannot read the catalogue; it is the Thai-reading domain reviewer who cannot
  check the copy until the staging deploy, with the screens already built around
  the strings. Mitigated by reviewing on the running UI rather than in JSON, and
  by naming the reviewers rather than hoping for them.
- **The flip is a scheduled event with other people's calendars in it**, not a
  task done the night before launch. It is its own ticket, blocked by every UI and
  email ticket, and it carries the staging deploy and the colleague review.
- **Angular is unsupported and undocumented by Paraglide.** Its getting-started
  guides cover SvelteKit, TanStack, React Router, Next.js, Astro, Vite and vanilla;
  the repo's `framework/` directory holds react, solid, svelte and vue only. The
  `paraglide-js compile` CLI prebuild step is the only route with primary sources
  behind it — the bundler plugins are not reachable from Angular's builder without
  a third-party custom builder that has reported problems with unplugin packages.
  This does not threaten the decision; it fixes the route.
- **Delete `src/paraglide` at the flip.** `cleanOutdir` is accepted-but-ignored
  when set in `paraglide.config.*`, so a stale outdir can survive a `baseLocale`
  change. Its symptom is Thai screens still rendering English, which reads as an
  incomplete translation rather than as a build fault — the worst kind of
  misdirection at exactly the moment the flip is being reviewed.
