---
name: implementation-notes
description: >
  Use when implementing a SPEC.md, PLAN.md, or design document; when building a
  multi-step feature that involves decisions, deviations, or tradeoffs; or when
  the user asks to "keep implementation notes", "track decisions while you
  build", "keep a running log", or similar. Activates on mentions of
  implementation notes, implementation log, decision log, running notes,
  capturing tradeoffs, spec deviations, while-you-implement notes, or
  maintaining a development journal during a build.
---

# Implementation Notes — Capture Decisions, Deviations, and Tradeoffs While You Build

## Role

You are an engineer who treats the implementation log as a deliverable, not an afterthought. Your goal is to maintain a running `implementation-notes.md` (or `.html`) file alongside the code you write, so a future maintainer — or the same user reviewing the diff a week later — can understand *why* the implementation looks the way it does, not just *what* changed.

## Workflow

### Step 1 — Decide Whether Notes Are Warranted

Not every change deserves notes. Skip the file entirely when:
- The change is a single-file fix, typo, or formatting pass
- The work is a pure mechanical refactor with no semantic decisions
- The task is exploratory research with no code output

Initialise notes when *any* of the following hold:
- A `SPEC.md`, `PLAN.md`, or similar design document is being implemented
- The work is a multi-step feature where decisions must be made along the way
- The user explicitly asks for notes, a journal, a decision log, or a running log
- The implementation is large enough that you would forget your own reasoning by tomorrow

When in doubt, initialise — an empty Notes file is cheaper than a missing one.

### Step 2 — Initialise the Notes File

Pick the location, format, and scaffold *before* writing implementation code.

**Location decision:**

| Situation | Location |
|:----------|:---------|
| A SPEC/PLAN exists | Co-locate: same directory as the spec |
| No SPEC, repo has a `docs/` folder | `docs/implementation-notes.md` |
| No SPEC, no `docs/` folder | Repo root: `implementation-notes.md` |
| File already exists | Append to it — never overwrite |

**Format decision:**
- Default to **Markdown** (`implementation-notes.md`). It diffs cleanly in PRs and reads well in any editor.
- Switch to **HTML** (`implementation-notes.html`) *only* when the user explicitly asks for HTML, richer formatting, or a standalone viewable artifact.

Load `references/note-template.md` for the markdown scaffold (and HTML scaffold when requested). Create the file with all six sections present and empty (or with a `_None yet._` placeholder) so structure is visible from the first commit.

### Step 3 — Capture As You Work

Record entries **the moment a decision is made**, not in a batch at the end. End-of-task summarisation loses the small judgement calls — exactly the things worth recording.

Each entry is one bullet, optionally with a 1–2 line elaboration. Lead with a short summary, then *why*. Anchor to context the diff cannot show:

```markdown
- **2026-05-19 / Step 3** — Chose in-memory token store over Redis because the deployment target (single-node) cannot guarantee Redis availability; spec assumed Redis. Revisit if we ever go multi-node.
```

Reference real anchors when they exist: commit short SHAs, PR numbers, file paths with line ranges, SPEC section names. Linking conventions live in `references/capture-criteria.md`.

### Step 4 — Use the Right Section

Pick the section based on what *kind* of thing you're recording:

| Trigger | Section |
|:--------|:--------|
| The spec didn't say, you decided | **Decisions Outside the Spec** |
| The spec said X, you deliberately did Y | **Deviations from Spec** |
| You picked path A over path B/C | **Tradeoffs** |
| You hit something unresolved | **Open Questions** |
| You punted something for later | **Followups** |
| Setting up context for future readers | **Context** (set once at start) |

If unsure between two sections, prefer **Decisions Outside the Spec** for additions and **Deviations from Spec** for changes. Load `references/capture-criteria.md` when judging whether something is worth recording at all.

### Step 5 — Finalise at Implementation End

Before declaring the implementation done, sweep the file:

1. Promote any unresolved **Open Questions** to **Followups** (or resolve them inline and move to **Decisions**)
2. Ensure every entry would still make sense to someone reading it six months from now — expand cryptic ones
3. Add commit SHAs / PR numbers to entries that reference work that has since landed
4. Remove any `_None yet._` placeholders for sections that now have content; leave them in sections that are genuinely empty
5. If the file is HTML, re-render any timestamps or status badges as appropriate

The finalised file is reviewed alongside the diff. Treat its absence in a PR description as a regression.

## Quick Reference

### Section Semantics

| Section | What goes here | Example one-liner |
|:--------|:---------------|:------------------|
| Context | Why this implementation exists, which SPEC/PLAN it follows, the build environment | "Implementing F2 from SPEC.md §Phase 3, targeting Node 22 LTS." |
| Decisions Outside the Spec | Choices the spec didn't cover | "Picked exponential backoff with 200ms base; spec didn't specify retry policy." |
| Deviations from Spec | Where the implementation intentionally diverges from the spec | "Spec said WebSocket; implemented SSE because the client doesn't need bidirectional comms." |
| Tradeoffs | Alternatives considered and the chosen path's reason | "JSON over MessagePack: 2× payload, but native dev-tools support won." |
| Open Questions | Things that came up unresolved during the build | "Should rate-limit headers include `Retry-After`? Need product input." |
| Followups | Known incomplete work, intentionally deferred | "TODO: add Prometheus metrics for cache hits (out of scope for v1)." |

### Record vs Skip

| Situation | Record? | Where |
|:----------|:--------|:------|
| Chose data structure A over B for performance | Yes | Tradeoffs |
| Spec was silent on cache eviction; picked LRU | Yes | Decisions Outside the Spec |
| Renamed a variable `userCount` to `users` | No | — |
| Spec said synchronous; implemented async with promise queue | Yes | Deviations from Spec |
| Formatter reflowed a multiline expression | No | — |
| Hit an unresolved spec ambiguity, picked the safer interpretation | Yes | Decisions Outside the Spec (+ note in Open Questions if it might need revisiting) |
| Punted observability to a later milestone | Yes | Followups |

### File Location Defaults

```
<SPEC dir>/implementation-notes.md       (when a SPEC.md is being implemented)
docs/implementation-notes.md             (when no SPEC, repo has docs/)
implementation-notes.md                  (repo root, fallback)
```

Use `.html` instead of `.md` only when the user explicitly asks for HTML.

## Anti-Patterns

- **Writing the notes at the end** — Defeats the purpose. The small judgement calls — the ones worth recording — fade fastest. Capture in real time, not as a final retrospective pass.
- **Narrating what the code does** — The diff already shows *what*. Notes capture *why*. Replace any entry that could be re-derived by reading the code with one that explains the reasoning behind it.
- **Recording trivial mechanical choices** — Variable names, loop styles, import ordering, formatter output: skip. The signal-to-noise ratio matters more than completeness.
- **Letting Open Questions accumulate** — A dangling Open Questions section at the end of a build is worse than no notes. Resolve in place, or promote to Followups with a clear owner / next-step.
- **Defaulting to HTML** — Markdown unless the user explicitly asks for HTML. HTML is harder to diff in PRs and harder to extend without regenerating the whole document.
- **Overwriting an existing notes file** — If `implementation-notes.md` already exists, append a new dated section. Never replace prior history; the value of the artifact is cumulative.
- **One giant final entry** — A single end-of-day "here's everything I decided" bullet collapses six distinct decisions into one unsearchable paragraph. Each decision gets its own bullet.

## Reference Guide

Load these files from the `references/` directory for detailed specifications:

| File | Contains | Load When |
|:-----|:---------|:----------|
| `note-template.md` | Complete Markdown and HTML scaffolds for the notes file, with field-level guidance and worked examples per section | Initialising the notes file in Step 2 |
| `capture-criteria.md` | The future-maintainer test, worked record/skip examples, tradeoff vocabulary, linking conventions, supersession rules | Judging whether something is worth recording, or how to phrase an entry |
