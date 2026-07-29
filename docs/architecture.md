# UI2Source — Architecture

Status: **design phase** · Last updated: 2026-07-30

This is the master design doc. Each part in the roadmap has its own spec in
[`specs/`](./specs); this document defines the **shared contracts** those parts
build against so the two of us can work in parallel without collisions.

---

## 1. Goal & non-goals

**Goal.** Click a React element in the browser and either (a) jump to its exact
`file:line` in VS Code, or (b) hand its full context to Claude Code as a prompt.

**Non-goals (for now — YAGNI):**

- No support for production/minified sites. We target **React apps running in dev
  mode** (where source info is available). This is a deliberate simplification.
- No framework support beyond React (no Vue/Svelte/Angular).
- No build-time Babel plugin. We read source info at **runtime** from the fiber tree.
- No in-browser code editing. We *hand off* to VS Code / Claude Code; we don't
  become an editor.

---

## 2. The pipeline

Every user interaction flows through the same four stages. This uniformity is what
makes the system extensible.

```
[1] Detect  →  [2] Identify  →  [3] Build Context  →  [4] Dispatch Action
 (inspector)     (fiber)          (context)             (action registry)
```

1. **Detect** — track the DOM element under the cursor; draw the overlay.
2. **Identify** — from that DOM node, find the React fiber and extract component
   identity + source info.
3. **Build Context** — assemble a **Context Package** (see §3).
4. **Dispatch Action** — run whichever registered action the user chose.

Stages 1–2 live entirely in the browser. Stage 3 is partly browser (props, fiber
data) and partly bridge (source snippet from disk). Stage 4 actions may call the
bridge.

---

## 3. Seam #1 — The Context Package (shared schema)

The single object every feature consumes. **This is the most important contract in
the project** — get it stable and everything else is additive.

```jsonc
{
  "version": 1,                       // bump on breaking changes
  "capturedAt": "<ISO timestamp>",

  "component": {
    "name": "Button",                 // fiber component name
    "displayName": "Button"           // React displayName if set, else name
  },

  "source": {                         // null if unavailable (e.g. host div)
    "fileName": "/abs/path/src/Button.jsx",
    "lineNumber": 42,
    "columnNumber": 7
  },

  "dom": {
    "tagName": "button",
    "id": "submit-btn",
    "classes": ["btn", "btn-primary"],
    "textPreview": "Submit"           // first ~80 chars of textContent
  },

  "props": { /* JSON-serializable subset of the fiber's props */ },

  "hierarchy": ["App", "CheckoutPage", "Toolbar", "Button"],  // ancestor component names

  "snippet": {                        // filled in by the bridge (Part 4); null until then
    "startLine": 38,
    "endLine": 52,
    "text": "…source lines…"
  }
}
```

**Rules:**
- Producers must never crash on missing data — every field except `version` and
  `capturedAt` may be absent/null.
- Consumers must treat any field as optional and degrade gracefully.
- Additive changes (new optional fields) do **not** bump `version`. Removing or
  retyping a field does.

---

## 4. Seam #2 — The Action Registry

Features are plugins, not hardcoded branches.

```js
/**
 * @typedef {Object} Action
 * @property {string}   id            unique, e.g. "open-in-vscode"
 * @property {string}   label         shown in UI, e.g. "Open in VS Code"
 * @property {(ctx: ContextPackage) => boolean} [availableWhen]  default: always
 * @property {(ctx: ContextPackage, services: Services) => Promise<void>} run
 */
```

- The registry is a simple list actions are appended to at startup.
- `services` gives an action what it needs without importing globals: e.g.
  `services.bridge.request(msg)`, `services.openUrl(url)`, `services.toast(text)`.
- **Adding a feature = writing one Action and registering it.** Core pipeline code
  never changes.

Planned actions by part: `open-in-vscode` (Part 2), `copy-context` (Part 4),
`send-to-claude` (Part 5).

---

## 5. Seam #3 — The Message Protocol (extension ↔ bridge)

A versioned, typed envelope. Transport decided in Part 3 (likely WebSocket or
`fetch` to `localhost`), but the envelope is transport-independent.

```jsonc
// Request
{ "v": 1, "id": "<uuid>", "type": "GET_SNIPPET", "payload": { "fileName": "…", "lineNumber": 42, "radius": 7 } }

// Response
{ "v": 1, "id": "<uuid>", "ok": true,  "payload": { "startLine": 35, "endLine": 49, "text": "…" } }
{ "v": 1, "id": "<uuid>", "ok": false, "error": { "code": "ENOENT", "message": "…" } }
```

Message types grow over time; each is a new `type` + payload shape, never a
reshape of the envelope:

| Type | Direction | Introduced | Purpose |
|------|-----------|-----------|---------|
| `PING` | ext → bridge | Part 3 | health check / discovery |
| `GET_SNIPPET` | ext → bridge | Part 4 | read source lines around a location |
| `OPEN_EDITOR` | ext → bridge | Part 3 (optional) | open file in VS Code from the bridge |
| `SEND_TO_CLAUDE` | ext → bridge | Part 5 | deliver a prompt to Claude Code |

(Note: Part 2's editor jump uses a `vscode://` URL directly from the browser and
does **not** need the bridge. The bridge becomes necessary at Part 3+.)

---

## 6. Tech stack (decisions & rationale)

| Decision | Choice | Why |
|----------|--------|-----|
| Extension manifest | **Manifest V3** | Required by Chrome for new extensions. |
| Target browser | **Chrome** (MVP) | One target keeps it simple; both devs use it. |
| Language | **JavaScript + JSDoc** | Matches existing `src/*.js`; keeps the learning curve gentle. JSDoc gives us light typing on the contracts. Migrating to TS later is easy. |
| Bundler | **esbuild** | Tiny config, fast, bundles content script + background into `build/`. |
| Bridge runtime | **Node.js** | Standard; direct filesystem + child_process access for Claude Code. |
| Editor deep link | `vscode://file/<abs>:<line>:<col>` | Both devs use VS Code. |

Open decisions deferred to their parts: bridge transport (WebSocket vs HTTP) → Part 3;
exact Claude Code hand-off mechanism → Part 5.

---

## 7. Two-person work split

The contracts above let us work in parallel. **Step 0 (do together):** agree on
§3, §4, §5 and stub them so both tracks compile. Then:

**Track A — Browser (owner: Dev A)**
- Part 1 (Inspector core), Part 2 (editor jump), browser side of Part 4.
- Owns: `src/inspector.js`, `src/overlay.js`, `src/fiber.js`, `src/index.js`,
  action registry, manifest, esbuild config.

**Track B — Bridge & AI (owner: Dev B)**
- Part 3 (Local bridge), Part 5 (Claude Code hand-off), bridge side of Part 4.
- Owns: `bridge/` (new dir), message-protocol handlers, snippet reader, Claude
  hand-off.

**Integration points (agree before diving in):** the Context Package schema (§3)
and the message protocol (§5). As long as both sides honor these, Track A can mock
the bridge and Track B can test with fixture messages — neither blocks the other.
Parts 1–2 (Track A) and a stubbed Part 3 (Track B) can proceed simultaneously from
day one.

---

## 8. Definition of done (whole project)

On any real React dev app: hover highlights components, clicking opens the exact
line in VS Code, and a "Send to Claude Code" action produces a prompt containing
the component's name, `file:line`, source snippet, props, and hierarchy.
