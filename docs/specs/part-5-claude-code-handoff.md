# Part 5 — Claude Code Hand-off

Status: **provisional** (firms up after Parts 1–4) · Track B · Depends on: Part 4

## Goal

The AI payoff: turn a Context Package into a well-formed prompt and deliver it to
Claude Code, so you can immediately ask Claude to change *that exact component*.

## Scope

**In:**
- A `send-to-claude` action (registry) available when a Context Package exists.
- A **prompt template** that turns the package into clear instructions + context.
- A delivery mechanism via the bridge (`SEND_TO_CLAUDE` message).

**Out:** running the model ourselves, applying diffs, multi-turn UI. We hand context
to Claude Code and let the user drive from there.

## Open decision — delivery mechanism

To be chosen during this part (all bridge-side):

| Option | How it works | Notes |
|--------|--------------|-------|
| **Context file + prompt** (recommended first) | Bridge writes `.ui2source/context.md` in the project and copies a prompt referencing it; user pastes into Claude Code | Simplest, no coupling to Claude Code internals |
| **Invoke `claude` CLI** | Bridge spawns Claude Code non-interactively with the prompt | Tighter, but depends on CLI availability/flags |
| **Clipboard only** | Bridge (or extension) copies a full prompt | Zero files, but user must paste manually |

Recommendation: start with the context-file approach — robust and decoupled — and
consider CLI invocation as a follow-up.

## Prompt template (draft)

```
I'm working on the <ComponentName> component.

Location: <fileName>:<lineNumber>
Rendered as: <dom.tagName> (classes: <classes>)
Component hierarchy: <hierarchy joined by " > ">

Current props:
<props as JSON>

Source:
```<lang>
<snippet.text>
```

<user's request goes here>
```

## Acceptance criteria ("done when")

1. `send-to-claude` on a picked component produces a complete prompt containing name,
   `file:line`, hierarchy, props, and snippet.
2. The chosen delivery mechanism works end-to-end: from click to "context ready in
   Claude Code" without manual assembly.
3. Missing fields degrade gracefully (prompt still forms, omits absent sections).

## Learning notes

This is the "precise context for an LLM" idea (the thing JSX Tool pitches): the
quality of the AI result depends almost entirely on how good the context package and
prompt are — so this part is really about *context design*, not model wrangling.
