# dds-sharing

## Reference implementation

The Gleam app ports the TS Pilot at commit `33a9491` (`rawinan-soma/dds-sharing@pilot`), pinned so parity is measured against one snapshot even as that branch moves. Its code is no longer in the tree; read it from this repo's history with `git show 33a9491:<path>` or `git grep <term> 33a9491 -- apps/`, or check out a copy beside the repo with `git worktree add ../dds-sharing-ref 33a9491`. Where the reference and `docs/spec.md` disagree, open a ticket for the repo owner; never settle it silently.

## Agent skills

### Issue tracker

Issues and PRs live on GitHub in `rawinan-soma/dds-sharing-gleam` (`gh`). Issues in `rawinan-soma/dds-sharing` and the closed files under `.scratch/dds-sharing/issues/` are history; ticket numbers dated before 2026-10-21 refer to them. See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-role vocabulary — `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix` — applied as GitHub labels. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context — `CONTEXT.md` (this repo's name for `GLOSSARY.md`) and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
