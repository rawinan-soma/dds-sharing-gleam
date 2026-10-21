# Issue tracker: GitHub

Issues and PRs for this repo live on GitHub, `rawinan-soma/dds-sharing-gleam`, the repo `git remote -v` names. Use the `gh` CLI for all operations; it infers the repo from the remote. The build's spec is `docs/spec.md`.

## History

- **Until 2026-10-21 issues lived in `rawinan-soma/dds-sharing`** (#1–#151). That repo is history: read it with `gh issue view <NN> -R rawinan-soma/dds-sharing --comments`; never create issues there. A ticket number in a commit, ADR or PR dated before 2026-10-21 means a `dds-sharing` number, not a `dds-sharing-gleam` one.
- Before that, issues #1–#98 were on GitHub. On 2026-09-21 the open ones moved to markdown files under `.scratch/dds-sharing/issues/`; on 2026-09-30 the tracker moved back to GitHub.
- **The closed files in `.scratch/dds-sharing/issues/` are history.** Read them; never add to them.
- **Ticket numbers 99–105 in commits, ADRs and PRs dated before 2026-09-30 mean those files, not GitHub numbers.** GitHub numbers are shared with PRs, so on `dds-sharing` `#99`–`#102` are PRs. The open ones moved:

  | File | `dds-sharing` |
  |---|---|
  | 77 | #77 (reopened) |
  | 96 | #96 (reopened) |
  | 101 | #122 |
  | 103 | #123 |
  | 104 | #124 |
  | 105 | #125 |

  99, 100 and 102 were closed and exist only as files.

## Conventions

- Branches are named `ticket-<NN>`, after the GitHub issue number.
- The first line of an issue body is `Blocked by: #NN, #NN` when it has open blockers. Omit it when none.
- The body is `## What to build` prose followed by an `## Acceptance criteria` checklist (`- [ ]` lines). `pr-clearance` grades against the checklist.
- The triage role is a label (see `triage-labels.md`); the category is `enhancement` or `bug`.
- A PR closes its issue with `Closes #NN`.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

## When a skill says "publish to the issue tracker"

Create a GitHub issue: `gh issue create --title "..." --body-file <file> --label <role>`.

## When a skill says "fetch the relevant ticket"

`gh issue view <NN> --comments`.

## When a skill says "apply a label"

`gh issue edit <NN> --add-label <role>`, removing the previous state role with `--remove-label`.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a GitHub issue labelled `wayfinder:map`; each **child** is an issue labelled `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`) whose body starts `Map: #NN`.

- **Blocking**: a `Blocked by: #NN, #NN` first line. A child is unblocked when every issue it lists is closed.
- **Frontier**: open, unblocked, unassigned children of the map; lowest number wins.
- **Claim**: assign yourself (`gh issue edit <NN> --add-assignee @me`) before any work.
- **Resolve**: comment the answer under an `## Answer` heading, close the issue, then add a gist and link to the map issue's Decisions-so-far.
