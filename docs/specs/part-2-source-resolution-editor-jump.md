# Part 2 — Source Resolution + Editor Jump

Status: **ready to build** · Track A (browser) · Depends on: Part 1

## Goal

Turn a picked component into an exact source location and open it in VS Code. This
is the **first end-to-end tool**: click an element → VS Code jumps to the line.

## Scope

**In:**
- Extract `file:line:col` from the fiber's dev-mode source info.
- Introduce the **Action Registry** (Seam #2) with its first action: `open-in-vscode`.
- On pick (click while in inspect mode), build a minimal Context Package and run the
  default action.
- Fire the `vscode://file/<abs-path>:<line>:<col>` deep link.

**Out:** bridge, snippet reading, props, multiple actions UI (single default action
is fine for now).

## How source info is obtained

- React dev builds compiled with the JSX-source transform attach source info to
  fibers. Check, in order:
  - `fiber._debugSource` → `{ fileName, lineNumber, columnNumber }` (older React), and
  - the newer equivalent exposed on the element/owner in current React dev builds.
- `fiber.js` gains `getSourceForFiber(fiber)` returning `{ fileName, lineNumber,
  columnNumber } | null`. If null (e.g. production build, missing transform), the
  action is unavailable and we show a clear message ("no source info — is this a dev
  build?").

> Both Vite and CRA enable the JSX-source transform in dev by default, so on our
> target apps this info is present. Confirm empirically early — this is the main risk
> in Part 2.

## Action Registry (Seam #2) — first cut

- Implement the registry per `architecture.md` §4.
- Register `open-in-vscode`:
  ```js
  {
    id: 'open-in-vscode',
    label: 'Open in VS Code',
    availableWhen: ctx => !!ctx.source,
    run: async (ctx, services) => {
      const { fileName, lineNumber, columnNumber } = ctx.source;
      services.openUrl(`vscode://file/${fileName}:${lineNumber}:${columnNumber || 1}`);
    },
  }
  ```
- `services.openUrl` opens the deep link (e.g. via a transient `<a>` click or
  `window.location`). Chrome will prompt to open VS Code the first time.

## Context Package (partial, this part)

Populate `version`, `capturedAt`, `component`, `source`, and `dom`. `props`,
`hierarchy`, and `snippet` come in Part 4. Shape must match `architecture.md` §3.

## Acceptance criteria ("done when")

1. On the test React dev app, entering inspect mode and clicking an element opens
   VS Code at the correct file **and line**.
2. Column is used when available; falls back to column 1.
3. Elements with no source info surface a clear, non-crashing message.
4. The action runs through the registry (not a hardcoded call), proving the seam.

## Learning notes

This is where the "magic" lands for the first time. Also the moment to validate our
central assumption (runtime source info exists in dev builds) before investing in
Parts 3–5.
