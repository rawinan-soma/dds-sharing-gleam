# 3. A plain SPA, and a collection path that bypasses it

Date: 2026-09-02

## Status

*Amended by [ADR 0018](0018-the-delivery-link-opens-a-page-and-the-archive-is-its-own-request.md):
`/d/<token>` now renders a server-side page and the archive is its own request.
The guarantee below is unchanged.*

*Amended by [ADR 0023](0023-two-lustre-entry-modules-and-no-server-components.md):
in the Gleam app, the Requester and the Reviewer are two Lustre entry modules of
one build, and `/d/<token>` is rendered by wisp. The guarantee and the rejections
below are unchanged.*

*Amended for [#14](https://github.com/rawinan-soma/dds-sharing-gleam/issues/14)
(2026-10-22): `/link-expired` is rendered by the server with no script, as the
pilot does, not by the SPA. Every dead-token redirect lands on it, so it belongs
to the collection path and keeps that path free of the front-end bundle.*

Accepted. Amends ADR-0001's surroundings only insofar as it fixes where the Delivery's link points.

## Context

One Angular application carries both surfaces — the public Request form and the
authenticated Reviewer surface — on one public ingress (#13, #16). The rendering
choice was open: server-side rendering, prerendering, one build or two.

Server-side rendering buys three things. Search indexing does not apply: a
Requester is given the address by DDC and never finds it through a search
engine. Per-request server data does not exist on the form — the disease group
list is a fixed 24-value seed (#3) and the confirmation page shows what the
Requester just typed. Faster first paint is real, but it precedes a wait of up
to 24 business hours for a Decision.

Prerendering the form buys less still. A prerendered form is a picture of a form
until the bundle hydrates, and it costs a build that must run the application in
Node, so browser-only code fails the build rather than the page.

Separately, #9 counts an Attempt **at presentation** of a Download token, and
throttles failed token lookups per IP. That work is unavoidably server work: the
server must inspect the token, count the Attempt, and then either stream or
refuse. It was never front-end work.

## Decision

The front end is a **plain SPA**. One build, no server-side rendering, no
prerendering. NestJS serves the built files from the same origin as the API,
excluding the API prefix from the SPA fallback.

The **Delivery email points at `GET /d/<token>` on NestJS**, not at an Angular
route. NestJS counts the Attempt, then streams the zip with range requests, or
redirects to `/link-expired`, which NestJS also renders.

## Consequences

No second runtime joins the Docker host, so `docker compose down` remains the
whole kill switch (#16), and there is one container, one port, and one public
route for DDC infra to register.

**An Extract stays collectable even if a front-end asset fails to load.** Under
any rendering choice that put collection behind the Angular bundle, a completed
extraction could become unreachable inside its 72 hours because of a static
asset. That is the strongest reason for this pair, and it is why the two halves
are one decision rather than two.

`/d/<token>` is hard to change later: the address travels in email, so a live
Download token outlives any redeployment that moves it. Treat the path as fixed.

The failure page is an Angular route rather than a NestJS template, so every
Thai sentence stays in one catalogue (#26).

First paint on the form is slower than it would be with prerendering. Accepted.

## Alternatives rejected

**Angular SSR with a Node server.** Adds a runtime to the host and hydration as
a failure mode, for benefits none of which apply here.

**Prerendered public routes.** A build-time constraint bought with a visual head
start on a page that is unusable until hydration.

**Two builds, public and Reviewer.** Two artifacts that must be released in step
and can drift. Separating bundles is not an access control — the API enforces
the Reviewer's session — and lazy loading already keeps the Reviewer code out of
the first download.
