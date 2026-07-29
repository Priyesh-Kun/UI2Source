# UI2Source

> Click a React component in your browser → jump straight to the source, and hand
> precise context to Claude Code to change it.

UI2Source bridges the gap between what you *see* on a page and where it *lives* in
your codebase. Hover any element on a running React app, click it, and either jump
to the exact `file:line` in VS Code or ship that component's full context to
Claude Code as a ready-to-run prompt.

## Why this exists

This is primarily a **learning project** built by two people. It's a way to get
hands-on with browser-extension internals, React's runtime (the **fiber tree**),
mapping a live DOM node back to source, a local browser↔filesystem bridge, and an
AI hand-off. A secondary goal is keeping the codebase approachable and easy to
split between two people.

## Architecture (at a glance)

The core is a **pipeline with a pluggable seam**. Everything a user does flows
through the same stages; new features slot in as new *actions* without touching the
core. Anything that needs the filesystem or AI goes through a small **local bridge**
(a Node process on your machine), because a browser extension can't read files or
invoke Claude Code on its own.

```
   ┌─────────────────────────────────────────────────────────┐
   │                   BROWSER (extension)                     │
   │                                                           │
   │   [1] Detect      →   [2] Identify    →   [3] Build       │
   │   element under       component via       a Context       │
   │   cursor              fiber tree          Package         │
   │                                                           │
   │                    [4] ACTION REGISTRY  ◄── the seam      │
   │              ┌───────────────┼───────────────┐            │
   │       "Open in VS Code"  "Copy context"  "Send to Claude" │
   │       (+ any future action plugs in here)                 │
   └──────────────────────────────┬────────────────────────────┘
                                   │  (typed messages)
                                   ▼
   ┌─────────────────────────────────────────────────────────┐
   │              LOCAL BRIDGE (Node, on your machine)         │
   │   read file · return snippet · hand off to Claude Code    │
   │   (+ future filesystem/AI capability = one new endpoint)  │
   └─────────────────────────────────────────────────────────┘
```

**Three seams keep it scalable** (design them once, they're nearly free):

1. **Context Package** — one stable schema describing a clicked component. Every
   feature consumes this object.
2. **Action registry** — features are entries `{ id, label, run(context) }`.
   Adding a feature = registering one action; core code never changes.
3. **Message protocol** — a versioned, typed envelope between extension and bridge,
   so the two halves evolve independently.

Full details in [`docs/architecture.md`](./docs/architecture.md).

## Roadmap (build order)

Each part is independently useful and has its own spec in [`docs/specs/`](./docs/specs).

| Part | Name | Delivers | Depends on |
|------|------|----------|------------|
| **1** | Inspector core | Extension + hover overlay + fiber-based component identification | — |
| **2** | Source resolution + editor jump | Click → `vscode://` opens the exact line (first end-to-end tool) | 1 |
| **3** | Local bridge | Node server + message protocol; reads source files | 2 |
| **4** | Context extraction | Assemble the full Context Package (props, snippet, hierarchy) | 3 |
| **5** | Claude Code hand-off | Turn context into a prompt, deliver to Claude Code (the AI payoff) | 4 |

First working milestone = Parts **1 + 2**. Everything after adds stages/actions
behind seams that already exist.

## Repository layout

| Path | Responsibility |
|------|----------------|
| `src/index.js` | Entry point — wires the pipeline together |
| `src/fiber.js` | Walks React's internal fiber tree to identify components |
| `src/inspector.js` | Hover/click logic — "which element did the user pick?" |
| `src/overlay.js` | Draws the highlight box over the targeted element |
| `docs/` | Architecture doc + per-part specs |

## Status

🚧 **Design phase.** Specs are being written; implementation has not started.

## License

TBD
