# Part 3 — Local Bridge

Status: **provisional** (firms up after Parts 1–2) · Track B · Depends on: Part 2

## Goal

A small Node process running on the developer's machine that the extension can talk
to, so we can do things the browser can't: read source files from disk and (later)
invoke Claude Code. Establishes the **message protocol** (Seam #3).

## Scope

**In:**
- A Node server started via a CLI (e.g. `npx ui2source-bridge` or `node bridge/index.js`).
- Transport for extension ↔ bridge messages (see decision below).
- Message envelope + dispatcher per `architecture.md` §5.
- First handlers: `PING` (health/discovery) and `GET_SNIPPET` (read lines around a
  location). `GET_SNIPPET` is consumed in Part 4 but implemented here.
- A safety boundary: the bridge only reads files under an allowed root (the project
  dir it was launched in) — never arbitrary paths.

**Out:** context assembly (Part 4), Claude hand-off (Part 5).

## Open decision — transport

Pick during this part:

| Option | Pros | Cons |
|--------|------|------|
| **WebSocket** (recommended) | Persistent, bidirectional, clean for future push | Slightly more setup |
| **HTTP fetch to localhost** | Dead simple, stateless | No server-push; CORS + MV3 host permissions to sort |

Recommendation: WebSocket, since Part 5 may want the bridge to push status back.

## Modules (new `bridge/` dir, Track B owns)

| File | Responsibility |
|------|----------------|
| `bridge/index.js` | CLI entry: parse args, resolve project root, start server. |
| `bridge/server.js` | Transport + connection lifecycle. |
| `bridge/protocol.js` | Envelope validation, request/response correlation by `id`. |
| `bridge/handlers/ping.js` | Returns bridge version + project root. |
| `bridge/handlers/getSnippet.js` | Reads `fileName`, returns lines `[line-radius, line+radius]`, path-guarded. |

## Extension-side changes (Track A coordinates)

- `services.bridge.request(msg)` in the content script's service layer, routed
  through the background service worker (content scripts shouldn't hold long-lived
  sockets directly under MV3 — background worker owns the connection).
- Graceful degradation: if the bridge is offline, bridge-dependent actions are
  disabled with a "start the UI2Source bridge" hint.

## Acceptance criteria ("done when")

1. Bridge starts from a project dir and logs its version + root.
2. Extension `PING` returns a response within the correlation protocol.
3. `GET_SNIPPET` returns correct lines for a known file; rejects paths outside the
   allowed root with a clear error.
4. Bridge offline → extension shows the hint, no crashes.

## Learning notes

This is the "how do a browser and a local process safely talk" lesson — plus the
security discipline of sandboxing filesystem access to a project root.
