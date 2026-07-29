# Part 1 — Inspector Core

Status: **ready to build** · Track A (browser) · Depends on: none

## Goal

A Chrome (MV3) extension that, when active on a React dev page, highlights the
element under the cursor and identifies which **React component** renders it. No
editor jump yet — just detect + identify + show.

## Scope

**In:**
- MV3 extension scaffold: `manifest.json`, background service worker, content script.
- esbuild build pipeline bundling `src/*` into `build/`.
- An **inspect mode** toggle (keyboard shortcut, e.g. `Alt+C`, plus extension icon click).
- Overlay that highlights the hovered DOM element (box + small label).
- Fiber lookup: from a DOM node, find its React fiber and read the component name.

**Out (later parts):** source resolution, editor jump, bridge, props/snippet, actions.

## Modules & responsibilities

| File | Responsibility |
|------|----------------|
| `src/index.js` | Content-script entry. Sets up inspect-mode toggle, wires inspector → overlay. |
| `src/inspector.js` | Tracks pointer, resolves the current target element, emits "hover" / "pick" events. |
| `src/overlay.js` | Renders/updates the highlight box + label. Owns its own DOM (isolated, high z-index). |
| `src/fiber.js` | `getFiberForNode(node)` and `getComponentName(fiber)`. Pure functions, no DOM side effects. |
| `manifest.json` | MV3 manifest: content script, background, commands (shortcut), permissions. |
| `esbuild.config.mjs` | Bundle content + background scripts to `build/`. |

## Key technical notes (fiber lookup)

- React attaches an internal key to DOM nodes, prefixed `__reactFiber$…`. Find it:
  ```js
  const key = Object.keys(node).find(k => k.startsWith('__reactFiber$'));
  const fiber = key ? node[key] : null;
  ```
- Walk **up** via `fiber.return` to find the nearest fiber whose `type` is a
  function/class (a real component), skipping host fibers (`type` is a string like
  `"div"`), to get a meaningful component name.
- Component name: `fiber.type.displayName || fiber.type.name || "Anonymous"`.
- Guard everything: non-React pages, detached nodes, and shadow DOM should fail
  quietly (inspect mode simply shows "no component").

## Overlay requirements

- A single fixed-position `<div>` (or a small set) appended to `document.body`,
  with a very high `z-index`, `pointer-events: none`, so it never intercepts clicks.
- Follows the target element's bounding box on hover; hidden when inspect mode is off.
- Label shows the component name (e.g. `<Button>`). Positioned to avoid going
  off-screen.

## Acceptance criteria ("done when")

1. Load the unpacked extension in Chrome; it activates on a chosen React dev site.
2. Pressing the shortcut toggles inspect mode on/off (visible cursor/overlay change).
3. Hovering any element draws a highlight box that tracks correctly on scroll/resize.
4. The label shows the correct nearest component name for common cases
   (verified against a small sample React app).
5. Non-React pages / unknown nodes degrade gracefully (no console errors, label says
   "no component").

## Suggested test app

Spin up a throwaway Vite React app (`npm create vite@latest -- --template react`)
to inspect against — Vite dev mode is our primary target.

## Learning notes

This part is the heart of the "React internals" learning: how React links DOM nodes
to fibers, and how the fiber tree mirrors your component tree.
