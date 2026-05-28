---
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(gh api:*), Bash(git blame:*), Bash(git log:*), Bash(git rev-parse:*)
description: Lean PR code review — same coverage as /code-review, ~50% fewer Sonnet tokens via mixed-model fan-out
argument-hint: [PR number, optional]
disable-model-invocation: false
---

Provide a lean code review for the given pull request.

This command preserves the four-dimension review (CLAUDE.md, bug scan, git history, prior PR comments) and the ≥80 confidence threshold of the original `/code-review`, but cuts Sonnet usage in half by routing retrieval-style dimensions to Haiku.

Follow these steps precisely:

1. **Eligibility check (Haiku, single pass).** Use a Haiku agent to verify the PR (a) is open, (b) is not a draft, (c) is non-trivial / not an automated PR, and (d) does not already have a code review from you. If any check fails, stop and do not proceed. Do **not** repeat this check at the end — one pass is enough.

2. **Shared context fetch (Haiku).** Use one Haiku agent to:
   - List file paths of all relevant CLAUDE.md files (root + directories whose files were modified). Return paths only, not contents.
   - Capture the PR head SHA via `gh pr view --json headRefOid` for later citation links.

   Return both as a structured payload that the next step's agents can reuse.

3. **Parallel review (mixed Sonnet + Haiku fan-out).** Launch all four agents in parallel. Each receives the payload from step 2 and the PR number; they each fetch the diff once via `gh pr diff`. Each returns a list of `(issue_description, dimension)` tuples.

   - **Agent A — Bug scan (Sonnet).** Read the diff. Shallow scan for obvious bugs in the changes only: null derefs, off-by-one, missed error paths, race conditions, broken control flow, security issues introduced by the diff. Ignore pre-existing issues, nitpicks, style, anything a linter/typechecker/compiler catches, and missing test coverage (unless CLAUDE.md explicitly requires it).

   - **Agent B — Git history reasoning (Sonnet).** Use `git blame` and `git log` on the modified files to understand intent. Flag issues only where historical context reveals a problem: regression of a past fix, revival of code that was explicitly removed, contradiction with a documented decision in commit messages. Reasoning-heavy — needs Sonnet.

   - **Agent C — CLAUDE.md compliance (Haiku).** Read the CLAUDE.md files from the payload. For each modified file in the diff, check whether the changes violate any explicit instruction. **Only flag violations of instructions that are written verbatim in CLAUDE.md** — do not invent rules. Pattern-matching against explicit rules, so Haiku is sufficient.

   - **Agent D — Prior PR comments (Haiku).** Use `gh pr list` and `gh api` to find merged PRs in the last 90 days that touched any of the same files. Read their review comments. Flag a current diff hunk only if a prior comment on the same code path identified a recurring concern that applies. Retrieval-heavy, so Haiku is sufficient.

4. **Per-issue confidence scoring (Haiku, one per issue).** For each issue returned in step 3, launch a parallel Haiku agent that receives the issue, the PR diff, and the CLAUDE.md path list. The agent scores the issue 0–100 using this rubric verbatim:
   - **0**: Not confident. False positive or pre-existing issue.
   - **25**: Somewhat confident. Might be real, might not. Couldn't verify. Stylistic issues not explicit in CLAUDE.md fall here.
   - **50**: Moderately confident. Verified real, but a nitpick or rare in practice.
   - **75**: Highly confident. Verified real, will be hit in practice, directly impacts functionality — or directly mentioned in CLAUDE.md.
   - **100**: Absolutely certain. Verified real, frequent in practice, evidence directly confirms it.

   For CLAUDE.md-flagged issues, the scorer must double-check that the CLAUDE.md text explicitly calls out the issue — not paraphrased, not inferred.

5. **Filter.** Drop any issue with a score below **80**. If no issues remain, stop and do not post a comment.

6. **Post the review comment.** Use `gh pr comment` to post the result. Keep it brief, no emojis, link every issue with a full-SHA permalink.

## False positives to avoid (steps 3 and 4)

- Pre-existing issues not introduced by the PR
- Code that looks like a bug but isn't (intentional pattern, framework idiom)
- Pedantic nitpicks a senior engineer wouldn't flag
- Issues a linter, typechecker, or compiler catches (imports, types, format) — CI runs these separately
- General quality concerns (test coverage, docs, broad security) unless CLAUDE.md explicitly requires them
- Rules in CLAUDE.md that are explicitly silenced in the code (e.g. lint ignore comments)
- Intentional behavior changes directly related to the PR's stated goal
- Real issues on lines the PR did **not** modify

## Notes

- Do **not** run build, typecheck, or tests yourself — CI runs these.
- Use `gh` for all GitHub interaction; do not web-fetch URLs.
- Make a todo list before starting.
- Every cited bug must include a full-SHA permalink.

## Output format

If 1 or more issues pass the filter, post exactly:

---

### Code review

Found N issues:

1. <brief description> (CLAUDE.md says "<quote>")

<full-SHA permalink with line range>

2. <brief description> (bug due to <reason>)

<full-SHA permalink with line range>

🤖 Generated with [Claude Code](https://claude.ai/code) — lean variant

<sub>If this code review was useful, please react with 👍. Otherwise, react with 👎.</sub>

---

If zero issues pass the filter, post exactly:

---

### Code review

No issues found. Checked CLAUDE.md compliance, obvious bugs, git history, and prior PR feedback.

🤖 Generated with [Claude Code](https://claude.ai/code) — lean variant

---

## Permalink format (required)

```
https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<start>-L<end>
```

- Full SHA (not abbreviated); fetched in step 2.
- `#L` notation; line range `L[start]-L[end]`.
- Include at least 1 line of context before and after the cited line.
- Do **not** use `$(git rev-parse HEAD)` in the URL — Markdown won't interpolate it.
