# 24. Copy through g18n behind a strict wrapper

Date: 2026-10-22

## Status

Accepted. **Amends** [ADR 0010](0010-copy-is-authored-in-english-and-translated-before-production.md)'s
Consequences for the Gleam app and leaves its decision intact: copy is authored
in English, translated into Thai by the repo owner and named colleagues, only
Thai is served, and `messages/en.json` is permanently maintained. Decided with
the repo owner on 2026-10-22 (`dds-sharing-gleam` #11), from the findings of #5.

## Context

The pilot reads `messages/` two ways. Angular calls Paraglide's compiled
`m.key()` functions, so a missing key is a compile error. NestJS reads the JSON
at runtime (`copy-catalogue.ts`) because Paraglide has no Node adapter, and
throws on a missing key. ADR 0010's Consequences — the raw-key leak after the
flip, the `url` strategy ban, deleting `src/paraglide`, `en.json` going
untooled — are all about Paraglide, which the Gleam app does not use. The flip
itself has already happened: the pilot serves `th` only.

## Decision

- **Copy goes through `g18n`, with `g18n_dev` generating
  `shared/src/shared/translations.gleam` as a prebuild step.** Code looks keys
  up by string, as the pilot's server does. The one module serves both SPAs,
  the four emails, `/d/<token>` and `/link-expired`.
- **`messages/{en,th}.json` stay at the repo root, unchanged.**
  `shared/src/shared/translations` is a symlink to `messages/`, because
  `g18n_dev` reads only `src/<project>/translations/`.
- **`shared/copy` wraps `g18n` and fails loudly on a missing key, naming it**,
  as `copy-catalogue.ts` does. `g18n` on its own returns the raw key.
- **A gleeunit test collects every key literal in `client/` and `server/` and
  fails on one absent from `th.json`**, in place of the compile error Paraglide
  gave the pilot's client.
- **Only `th_translator()` is served.** The pilot's two catalogue checks —
  identical key sets in `en.json` and `th.json`, and every `th.json` value Thai
  except the checked-in exemptions — are ported to gleeunit.
- An unmatched `{param}` is left in place, as the pilot does.

## Consequences

ADR 0010's flip hazards do not exist here: there is no `baseLocale` fallback to
invert, no URL strategy to add, and no stale outdir, since the generated module
is rebuilt from `messages/` every build. `en.json` is compiled into a translator
that nothing serves; the two ported checks still bind it to `th.json`.

A key built at runtime from parts escapes the scanning test. Keys are written
as whole literals.

The build depends on a symlink, which a Windows checkout may not honour. Nobody
builds on Windows.

## Alternatives rejected

**A generated module with one typed function per key.** Stricter — a missing
key or a wrong param is a compile error — but it departs from how the pilot
reads copy, and it is our own generator to maintain where a library exists.

**Porting `copy-catalogue.ts` as our own runtime reader.** The closest port of
the server, but it puts a hand-written reader where `g18n` already does the
lookup and substitution.

**`kielet` (gettext).** It would replace the inlang JSON format that ADR 0010
and the translators work in.

**Moving the files into `shared/src/`, or copying them there at build time.**
Moving changes a path the spec and ADR 0010 name; copying leaves a duplicate
that can go stale.
