# 22. A disposable Pilot releases real data from the repo owner's server

Date: 2026-09-28

## Status

Accepted. Decided by the repo owner while triaging #77 (2026-09-28). Adds the
Pilot beside Production, which #77 still gates; adds `docs/spec.md` §18.15;
rewords #96's "staging deploy" to a running deploy, which the Pilot can be — #96 is not blocked by it. Tickets: #100 (the reference
prefix), #101 (the Pilot).

Amended 2026-10-21 by the repo owner (`dds-sharing-gleam` #12): there may be
more than one Pilot. A second Pilot, the Gleam app, runs under this decision
on the same terms, with the prefix `PLG`. Each Pilot passes the exposure checks
itself before its first real release; no Pilot borrows another's results.

## Context

Production is the department's service on a DDC VM behind the ministry-managed
edge (§17.4), and it does not exist yet. The repo owner needs a running service
now — for #96's copy review, and for a workshop in two to three months whose
participants are real Requesters — and has a server of their own on a Tailscale
network and a domain of their own.

The obvious shape for that is a staging environment: fake upstream, a mail
catcher, reachable only on the tailnet. Each of those was offered and declined,
because the point is real use by real people.

## Decision

**The Pilot is a second, separate service for real Requesters**, run on the
repo owner's server (`personal-test-server`) and open to the internet through a
Cloudflare Tunnel on the owner's domain, with no identity gate in front of it.

- It calls **the real upstream** with the department's token, so its Extracts are
  real case-level DDS surveillance data, and **its Decisions are real releases**
  under the same approval gate. The DDC PDPO is told, as §18.1 was settled.
- Its mail goes through **the real relay**. A mail catcher was declined: once
  Extracts are real, a catcher is a second screen showing Download tokens, which
  §10.8 forbids.
- It **runs beside Production indefinitely** and is never merged into it. Its
  references carry their own prefix (`PLT-2569-0142`) so a number quoted over the
  telephone says which service to look in (#100).
- **Its record is disposable.** No backup: losing the server loses every trace
  of what the Pilot released. Only the Reviewer accounts need to survive, and
  those are re-seeded by the host command.
- **One Reviewer at launch** — the repo owner. The two-Reviewer minimum is
  knowingly unmet until a colleague is seeded; recovery meanwhile is shell access
  to the server.
- It goes public **in English**, and is redeployed when #96 flips it to Thai.
  Updates are by hand on the server (`git pull`, `docker compose up -d --build`).

## Considered options

- **A review-only staging on the fake upstream** — declined: the workshop is
  real use, not a demonstration.
- **Probe-only against the real upstream, extraction switched off** — declined;
  it would need a switch Production must never have.
- **Tailnet-only, or Cloudflare Access / an email gate** — declined: the owner
  wants the workshop's participants to reach it unrestricted. Tailscale Funnel
  was compared and set aside because it cannot serve the owner's domain.
- **Production starts from the Pilot's database**, or the Pilot stops and hands
  an archive to DDC — declined in favour of two separate services.
- **A nightly backup, or a nightly export of Decisions only** — declined: the
  Pilot is disposable.

## Consequences

**The spec's own rules do not all hold on the Pilot, and that is on purpose.**
§3.1's internet-facing reach does; §17.4's ministry edge, infra sign-off and
`moph.go.th` name do not; §18.11's "one disk, no copy" is accepted by the
ministry for Production and by the repo owner, more sharply, for the Pilot.

**A release made from the Pilot may leave no record anywhere.** §12's premise —
that every release can be read back years later — is given up for the Pilot's
releases, not for Production's. The workshop's approvals put the repo owner's
name on releases that the record may not keep.

**Case data and department credentials sit on personally owned hardware and
cross a third party.** Extracts land in the Pilot's MinIO for up to 72 hours;
the upstream token and the relay credentials live in the Pilot's `.env`; and
Cloudflare terminates TLS, so Requester contact details, Reviewer sign-ins and
Extract downloads pass through it in clear.

**#77 is unchanged.** Every check it makes of Production — data stores
unreachable from outside, host firewalling, NTP, `TRUST_PROXY`, the kill switch
— is made of the Pilot too, in #101, because the Pilot faces the same internet.
