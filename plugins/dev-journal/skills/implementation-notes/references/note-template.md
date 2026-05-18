# Implementation Notes — Note Template Reference

This reference provides the full Markdown and HTML scaffolds for the `implementation-notes` file, plus per-section field-level guidance and a worked example. Use the Markdown template by default; use the HTML template only when the user explicitly asks for HTML.

---

## Markdown Template

Copy this scaffold verbatim into `implementation-notes.md` at file creation. Replace `<feature name>` and the placeholder bullets. Leave sections empty (or with `_None yet._`) rather than deleting them — empty sections signal that they were considered.

```markdown
# Implementation Notes — <feature name>

## Context

<!-- One short paragraph. What is being implemented, which spec/plan it follows, the build environment, the start date. Set once at the start; update only if the framing materially changes. -->

_e.g._ Implementing **F2: Rate-limit middleware** from `SPEC.md` §Phase 3. Target runtime is Node 22 LTS; deployment is a single-node container behind nginx. Started 2026-05-19.

## Decisions Outside the Spec

<!-- Choices the spec didn't explicitly cover. One bullet per decision. Lead with a short summary, then *why*. -->

- _None yet._

## Deviations from Spec

<!-- Where the implementation intentionally diverges from the spec. Name the spec anchor (section or feature ID), what the spec said, what was done instead, and why. -->

- _None yet._

## Tradeoffs

<!-- Decisions where a specific alternative was rejected. Name the alternatives, name the winner, give the one-sentence reason. Avoid "for simplicity" — say *what* was simpler about it. -->

- _None yet._

## Open Questions

<!-- Things that came up unresolved during the build. Resolve in place, or promote to Followups before finalising. -->

- _None yet._

## Followups

<!-- Known incomplete work, intentionally deferred. Include the reason for deferral and what would unblock it. -->

- _None yet._
```

---

## Entry Format

Each entry under any section follows the same shape:

```markdown
- **<anchor>** — <one-sentence summary>. <Optional 1–2 line elaboration giving the *why*.>
```

Where `<anchor>` is one of:

| Anchor type | Example | Use for |
|:------------|:--------|:--------|
| ISO date | `2026-05-19` | Always include — fastest visual chronology |
| Date + step | `2026-05-19 / Step 3` | When the implementation follows a numbered plan |
| Date + SHA | `2026-05-19 / 9f3a1c2` | When the entry corresponds to a specific commit |
| Date + SPEC anchor | `2026-05-19 / SPEC §F2` | When the entry calls out a spec section by reference |

Pick the anchor that gives the reader the fastest jump back to context. When in doubt, include date plus the most specific other anchor available.

---

## Field-Level Guidance Per Section

### Context

**Belongs:** The minimum a reader needs to understand the rest of the file. Spec/plan reference, target environment, start date, who is doing the work (if collaborative).

**Doesn't belong:** The actual implementation plan. The notes file documents the *journey*, not the *plan*. If a plan exists, link to it; don't duplicate it.

**Worked example:**
```markdown
## Context

Implementing **F2: Rate-limit middleware** from `SPEC.md` §Phase 3. Target runtime is Node 22 LTS; deployment is a single-node container behind nginx. Started 2026-05-19.
```

### Decisions Outside the Spec

**Belongs:** Concrete choices where the spec was silent. Naming a header, picking a default, choosing an algorithm, selecting a library when the spec just said "store somewhere."

**Doesn't belong:** Choices the spec explicitly mandated. If the spec said "use Redis" and you used Redis, that's not a Decision — it's just implementation.

**Worked example:**
```markdown
- **2026-05-19 / Step 2** — Used `Retry-After` header (RFC 6585) for 429 responses. Spec said "return rate-limit info" without specifying the header. Aligns with standard client behavior.
- **2026-05-19 / Step 4** — Picked sliding-window over fixed-window counter for the limiter. Spec said "limit requests per minute"; sliding-window avoids the well-known boundary-burst issue.
```

### Deviations from Spec

**Belongs:** Cases where the spec said X and the implementation does Y on purpose. Include the spec anchor so a reviewer can find what the spec actually said.

**Doesn't belong:** Bugs in the implementation (those belong in the diff and a test). Or cases where the spec is being amended — those go back into the spec, not into notes.

**Worked example:**
```markdown
- **2026-05-19 / SPEC §F2.3** — Spec said use Redis as the token store; implemented in-memory `Map`. Reason: single-node deployment, Redis isn't provisioned. Acceptable because rate-limit state is per-instance and bounded. Revisit when we go multi-node — see Followups.
```

### Tradeoffs

**Belongs:** Decisions where two or more credible alternatives were considered. Name the alternatives, name the winner, give a one-sentence reason that mentions *what specifically* the winner does better. Avoid "for simplicity" — say *what* was simpler about it.

**Doesn't belong:** "Decisions" with only one credible option (those go in Decisions Outside the Spec instead). Or hypothetical alternatives that weren't seriously considered.

**Worked example:**
```markdown
- **2026-05-19 / Step 5** — JSON over MessagePack for the limiter's persistence dump. MessagePack would have been ~40% smaller, but JSON wins because: (1) ops can `cat | jq` it during incidents, (2) no extra dependency, (3) persistence is recovery-only, not a hot path.
- **2026-05-19 / Step 6** — Function exports over a class. The middleware has no instance state; a class would just be a namespace with `this`-less methods.
```

### Open Questions

**Belongs:** Things you genuinely don't know the answer to and that affect the implementation. Phrase as questions, not statements. Include what you'd need to resolve them.

**Doesn't belong:** Stylistic preferences. "Should we use semicolons?" is not an Open Question. Open Questions are about behavior or design, not aesthetics.

**Worked example:**
```markdown
- **2026-05-19** — Should the limiter expose a `/healthz`-style endpoint reporting current token counts? Useful for ops dashboards; not in spec. Need product input.
- **2026-05-19** — When a request is rate-limited, should we log the user ID? Privacy implication. Need to check with security before defaulting either way.
```

### Followups

**Belongs:** Known incomplete work, deferred intentionally. Include the reason for deferral and what would unblock it. Followups should be specific enough that someone other than you could pick them up.

**Doesn't belong:** Generic TODOs ("improve performance"). A Followup is a *specific* deferred change with a clear owner or unblocker.

**Worked example:**
```markdown
- **Multi-node distributed token store.** Currently per-instance in-memory. Unblocker: when we add a second app node, switch to Redis (or similar). Owner: same team.
- **Prometheus metrics for limiter hit/miss/reject counters.** Out of scope for v1; observability stack lands in v2.
```

---

## HTML Template

Use only when the user explicitly asks for HTML. Standalone, single-file, no external dependencies, no JS.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Implementation Notes — &lt;feature name&gt;</title>
<style>
  :root {
    --fg: #1a1a1a;
    --fg-muted: #555;
    --bg: #fdfdfd;
    --border: #e5e5e5;
    --accent: #2962ff;
    --code-bg: #f4f4f6;
  }
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", system-ui, sans-serif;
    color: var(--fg);
    background: var(--bg);
    max-width: 760px;
    margin: 2.5rem auto;
    padding: 0 1.25rem;
    line-height: 1.6;
  }
  h1 { font-size: 1.75rem; margin-bottom: 0.25rem; }
  h2 {
    font-size: 1.1rem;
    margin-top: 2.25rem;
    padding-bottom: 0.35rem;
    border-bottom: 1px solid var(--border);
  }
  ul { padding-left: 1.25rem; }
  li { margin: 0.5rem 0; }
  li strong { color: var(--accent); font-weight: 600; }
  code {
    background: var(--code-bg);
    padding: 0.1rem 0.35rem;
    border-radius: 3px;
    font-size: 0.9em;
  }
  .empty { color: var(--fg-muted); font-style: italic; }
  section { margin-top: 1.5rem; }
</style>
</head>
<body>

<h1>Implementation Notes — &lt;feature name&gt;</h1>

<section id="context">
  <h2>Context</h2>
  <!-- One short paragraph. What is being implemented, which spec/plan it follows, the build environment, the start date. -->
  <p class="empty">No context recorded yet.</p>
</section>

<section id="decisions">
  <h2>Decisions Outside the Spec</h2>
  <!-- Choices the spec didn't explicitly cover. One <li> per decision. Lead with a short summary, then *why*. -->
  <ul>
    <li class="empty">None yet.</li>
  </ul>
</section>

<section id="deviations">
  <h2>Deviations from Spec</h2>
  <!-- Where the implementation intentionally diverges from the spec. Name the spec anchor and the reason. -->
  <ul>
    <li class="empty">None yet.</li>
  </ul>
</section>

<section id="tradeoffs">
  <h2>Tradeoffs</h2>
  <!-- Decisions where a specific alternative was rejected. Name the alternatives and the winner's reason. -->
  <ul>
    <li class="empty">None yet.</li>
  </ul>
</section>

<section id="open-questions">
  <h2>Open Questions</h2>
  <!-- Unresolved questions that affect the implementation. Resolve or promote to Followups before finalising. -->
  <ul>
    <li class="empty">None yet.</li>
  </ul>
</section>

<section id="followups">
  <h2>Followups</h2>
  <!-- Known incomplete work, intentionally deferred. Include the reason and what would unblock it. -->
  <ul>
    <li class="empty">None yet.</li>
  </ul>
</section>

</body>
</html>
```

**Entry format inside `<ul>`:**

```html
<li><strong>2026-05-19 / Step 3</strong> — Chose in-memory token store over Redis because the deployment target is single-node; spec assumed Redis. Revisit if we ever go multi-node.</li>
```

When adding the first real entry to a section, remove the `<li class="empty">None yet.</li>` placeholder.

---

## When to Switch Templates Mid-Implementation

If the user starts in markdown and later asks for HTML (e.g., to share with someone outside the repo), do not delete the markdown file. Generate the HTML version *from* the markdown file as a side-by-side artifact. Keep the markdown file as the source of truth — markdown is easier to keep updated incrementally; HTML is a presentation layer.
