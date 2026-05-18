# Implementation Notes — Capture Criteria Reference

This reference answers the hardest question the skill faces: *what is worth recording, and how do I phrase it?* Load this when judging whether a decision deserves a bullet, when an entry feels vague, or when you need to handle a decision that supersedes an earlier one.

---

## The Future-Maintainer Test

The single rule of thumb:

> Would someone reading this code six months from now — without conversation history, without your memory, with only the diff and the spec — wonder *why* this choice was made?

If yes, record it. If no, skip it.

The test focuses you on the *gap between the diff and the reasoning*. A diff shows the final code. A spec shows the intent. Notes fill the space between: every place where the diff is one of several reasonable shapes, and only context explains which one was picked.

---

## What to Record (with examples)

Anything that meets the future-maintainer test. Common categories:

1. **Spec gaps you filled** — the spec was silent on something concrete; you chose. (Goes in: Decisions Outside the Spec.)
   - *e.g.* "Spec said 'cache results'; picked LRU with 256-entry cap. Spec didn't specify eviction or size."

2. **Deliberate spec divergence** — the spec said one thing; you did another on purpose. (Goes in: Deviations from Spec.)
   - *e.g.* "Spec specified PostgreSQL; used SQLite for the local dev variant because the spec's deployment assumptions don't apply to the test harness."

3. **Real tradeoffs** — at least two credible alternatives existed; you picked one. (Goes in: Tradeoffs.)
   - *e.g.* "Chose `serde_json` over `simd-json`: simd-json is ~2× faster but adds an unsafe dependency and requires runtime CPU feature detection. The parse path isn't hot enough to justify it."

4. **Workarounds for external constraints** — the choice would be different in a different environment. (Decisions Outside the Spec, or Tradeoffs if alternatives were weighed.)
   - *e.g.* "Polling instead of webhooks because the target environment can't receive inbound HTTP."

5. **Non-obvious correctness reasoning** — the code is right but the reason isn't visible. (Decisions Outside the Spec.)
   - *e.g.* "Locked the read order to (config, db, secrets) so a partially-initialised secrets store can't be observed."

6. **Things you tried first and abandoned** — a failed approach saves future you from trying the same thing. (Tradeoffs.)
   - *e.g.* "First tried a regex-based parser; abandoned after the grammar grew nested quoting. Switched to a state machine. Don't go back to regex."

7. **Deferred work** — known incomplete, with a reason. (Followups.)
   - *e.g.* "Per-tenant quotas: out of scope for v1; needs billing integration that lands in v2."

8. **Unresolved ambiguity** — something the spec is ambiguous about and you picked an interpretation. (Decisions Outside the Spec + Open Questions if it might need revisiting.)
   - *e.g.* "Spec says 'idle for 30 seconds'; ambiguous whether 'idle' means no CPU or no requests. Interpreted as no requests. Flagged for product confirmation."

---

## What to Skip (with examples)

If a future maintainer can re-derive the choice from the code itself or from generic engineering common sense, don't record it. Examples that **should not** appear in notes:

1. **Naming choices that match conventions** — `userCount` vs `users`, `i` vs `idx`. Skip unless the name encodes a non-obvious meaning.

2. **Formatter / linter output** — the tool made the change. The tool itself is the documentation.

3. **Standard library use** — "Used `Array.prototype.map` to transform the list." That's just the diff.

4. **One-credible-option decisions** — if there was only one sensible way to do it, there was no decision. Skip.

5. **Implementation order** — "Wrote the validator before the parser." Order of writing doesn't matter to the reader; the final shape does.

6. **Restating the spec** — "Implemented F2 as specified." That goes in the commit message or PR description, not in notes. Notes are for what the diff *can't* tell you.

7. **Generic best-practice acknowledgements** — "Added input validation." If validation is non-trivial or unusual, *that's* worth a note; the existence of validation is not.

8. **Trivial refactors** — extracted a constant, renamed a variable, split a function. Unless the refactor reveals a design decision, skip.

---

## Tradeoff Vocabulary

A tradeoff entry that says "did X for simplicity" is almost always a missed opportunity. *Simpler in what way?* Be specific.

| Vague phrasing | Better phrasing |
|:---------------|:----------------|
| "for simplicity" | "fewer moving parts at the cost of …" or "fewer dependencies; loses …" |
| "for performance" | "X% faster on the Y workload; pays … in memory" or "avoids a hot allocation in …" |
| "more idiomatic" | "matches the rest of the codebase's pattern in `<file>`; alternative would have looked foreign" |
| "more flexible" | "lets us swap …  without changing …; pays … in current-day verbosity" |
| "future-proof" | (delete and rewrite — usually means "no good reason today") |

A good tradeoff sentence names the loser, names what the loser was *better* at, and names what specifically the winner gives in exchange. Three concrete pieces, no abstractions.

---

## Anchoring and Linking

Every entry should be findable from context the reader has. Use these anchors:

| Anchor | When to use | Example |
|:-------|:------------|:--------|
| ISO date | Always | `2026-05-19` |
| Step number | When the implementation follows a numbered plan | `Step 3` |
| Short SHA | When the entry corresponds to a specific commit (add after commit lands) | `9f3a1c2` |
| PR number | When the entry is reviewed in a specific PR | `#42` |
| SPEC anchor | When the entry references a spec section | `SPEC §F2.3` or `SPEC §Phase 3` |
| File path + line range | When the entry is grounded in a specific code site | `src/limiter.ts:42-58` |

Stack anchors when more than one applies. The goal is for any reader — including a search-engine grep — to follow the entry back to the artifact it documents.

---

## Supersession: Handling Reversed Decisions

When a later decision invalidates an earlier one, **do not delete the earlier entry**. The history of *why* a decision was reversed is often more valuable than the final state — it stops the same wrong path from being tried again.

The pattern:

```markdown
- ~~**2026-05-19 / Step 2** — Picked in-memory token store; single-node deployment, no Redis available.~~ _Superseded by 2026-05-21 entry below._
- **2026-05-21 / Step 7** — Migrated token store to Redis. Discovered that two app instances will run during blue/green deployments, breaking the per-instance assumption. In-memory was correct for steady state but wrong during deploys.
```

Use strikethrough on the superseded entry, append `_Superseded by …_`, and write the new entry immediately below (or in a later section if context changed). Same for the HTML template — wrap the superseded `<li>` in `<s>…</s>` and add the supersession note.

---

## Vagueness Checklist

Before committing an entry, scan it against these failure modes:

| Vague | Why it fails | Rewrite by |
|:------|:-------------|:-----------|
| "Refactored for clarity" | Doesn't say what was unclear | Naming the smell — "the function was doing parsing and validation in one pass; split because errors couldn't be tied to phase" |
| "Improved error handling" | The diff already shows that errors are handled | Naming the *decision* — "errors from the parser now include the input position; the alternative was a separate position-tracking layer" |
| "Better than the previous approach" | What was the previous approach? Why is this better? | Naming both and the dimension — "previous approach was X; new approach is Y; trades A for B" |
| "Standard pattern" | Standard where? Why is it the right standard here? | Naming the precedent — "matches the pattern in `src/cache.ts`; chosen over alternative Y because it would have introduced two namespaces" |
| "Felt right" | Notes are not vibes | Refusing to commit until a concrete reason exists |

If an entry survives the checklist, it's worth recording. If not, either rewrite it or skip it.

---

## When the Notes File Itself Needs Notes

Occasionally, the *act* of taking notes requires a meta-decision (e.g., "we're going to use a new section that wasn't in the template"). Record that in the Context section, not in a new top-level section. Adding sections to the template is fine, but each one should appear in at least three implementations before being added to the canonical scaffold in `note-template.md`.
