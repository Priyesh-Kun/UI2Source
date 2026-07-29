# Part 4 — Context Extraction

Status: **provisional** (firms up after Parts 1–3) · Tracks A + B · Depends on: Part 3

## Goal

Fill out the full **Context Package** (`architecture.md` §3) for a picked component:
props, ancestor hierarchy (browser side), and the source snippet (bridge side).
This is the raw material the AI hand-off needs.

## Scope

**In:**
- **Browser side (Track A):**
  - `props`: read the fiber's `memoizedProps`, filter to a JSON-serializable subset
    (drop functions, React elements, circular refs; cap depth/size).
  - `hierarchy`: walk `fiber.return` collecting named component ancestors.
  - `dom`: finalize `tagName`, `id`, `classes`, `textPreview`.
- **Bridge side (Track B):**
  - `snippet`: via `GET_SNIPPET` from Part 3, fetch source lines around
    `source.lineNumber` (configurable radius, default ~7).
- A `copy-context` action (registry) that copies the assembled package as formatted
  JSON/markdown — useful on its own and a stepping stone to Part 5.

**Out:** prompt construction + Claude hand-off (Part 5).

## Serialization rules for `props`

- Include: primitives, plain arrays/objects (depth-capped, e.g. 3 levels).
- Exclude/replace: functions → `"[Function]"`, React elements → `"[ReactElement]"`,
  DOM nodes → `"[Node]"`, circular refs → `"[Circular]"`.
- Cap total size (e.g. 10KB) to avoid dumping huge stores.

## Acceptance criteria ("done when")

1. Picking a component yields a Context Package validating against §3 (all fields
   present or explicitly null).
2. `props` never throws on functions/elements/circular structures and respects the
   size cap.
3. `hierarchy` matches the visible component nesting on the test app.
4. `snippet` shows the correct source lines around the component's definition.
5. `copy-context` puts a readable package on the clipboard.

## Learning notes

The interesting problem here is **safe serialization** of arbitrary runtime props,
and stitching browser-side data together with bridge-side file reads into one object.
