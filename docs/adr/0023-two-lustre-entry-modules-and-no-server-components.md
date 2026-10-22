# 23. Two Lustre entry modules in one build, and no server components

Date: 2026-10-22

## Status

Accepted. **Amends** [ADR 0003](0003-plain-spa-and-a-collection-path-that-bypasses-it.md)'s
mechanism for the Gleam app, and leaves its guarantee and its rejections intact:
no SSR, no prerendering, one build, and collection never depends on a front-end
bundle. [ADR 0018](0018-the-delivery-link-opens-a-page-and-the-archive-is-its-own-request.md)
is unchanged. Decided with the repo owner on 2026-10-22 (`dds-sharing-gleam` #8),
from the findings of #5.

## Context

ADR 0003 describes one Angular application with a lazy-loaded Reviewer route,
served by NestJS. The Gleam app ports `dds-sharing@pilot` to Lustre on the
client and wisp on the server. Lustre offers three ways to render: client SPAs,
plain server rendering (`element.to_document_string`), and server components,
which keep the view on the server and patch the DOM over a transport.

## Decision

- **The Requester and the Reviewer are two Lustre client SPAs, built as two
  entry modules of one `client` build.** bun emits one bundle per entry and a
  shared chunk. The server serves the Reviewer shell for `/reviewer/*` and the
  Requester shell for every other path outside `/api` and `/d`.
- **Each SPA fetches its first data after it starts, as the pilot does:**
  `GET /api/reference` for the Requester, `GET /api/reviewer/session` for the
  Reviewer. Nothing is embedded in the shell.
- **`/d/<token>` is a wisp handler that renders HTML on the server with no
  `<script>`**, and `/d/<token>/archive` streams as ADR 0018 says.
- **Lustre server components are not used.**

## Consequences

Two entries in one build release together, so they cannot drift. That was ADR
0003's objection to two builds, and it does not apply here. The Requester's
download still carries no Reviewer code, as the lazy route ensured before.

Both SPAs keep the pilot's loading state on first paint. Accepted for parity.

## Alternatives rejected

**Server components.** Interactive use needs a WebSocket, because the SSE and
polling transports cannot send events from the client to the server. A WebSocket
is an edge setting that §17.4 forbids. They also need a client script, so they
cannot serve `/d/<token>` without breaking ADR 0003's guarantee, and every
keystroke would be a round trip.

**One entry for both SPAs.** Simpler wiring, but without code splitting every
Requester downloads the Reviewer code.

**Embedding first data in the shell (state transfer).** Removes a round trip and
the loading state, but departs from the pilot. After sign-in or a 401 the
Reviewer SPA would also need a reload or a second channel to get its new session.
