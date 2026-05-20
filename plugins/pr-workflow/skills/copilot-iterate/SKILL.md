---
name: copilot-iterate
description: >
  Drive a GitHub pull request to convergence by repeatedly addressing
  Copilot review feedback, pushing fixes, re-triggering the reviewer, and
  detecting the literal "generated no new comments" summary as the stop
  signal. Use when the user says /copilot-iterate,
  /pr-workflow:copilot-iterate, "iterate on Copilot review", "address
  Copilot feedback", "next round of Copilot", "drive this PR to
  convergence", "loop on Copilot until merge", or "respond to Copilot
  review". Runs an autonomous loop: fix → test → push → resolve threads
  → re-add reviewer → poll for the next review → classify → repeat or
  stop. Repo-agnostic — auto-detects owner / repo / PR from the current
  branch.
---

# Copilot Iterate — Drive a PR to "no new comments" via repeated Copilot review

## Why this matters

GitHub's Copilot reviewer does not re-review on subsequent pushes. The
only trigger for a fresh review is **re-adding**
`copilot-pull-request-reviewer` to the PR. Convergence is signalled by a
literal phrase in the review summary — `"generated no new comments"` —
and missing the phrase means the PR is still being iterated. The loop is
mechanical but easy to get wrong: passing the display name `Copilot`
instead of the slug 404s silently (gh lowercases the input before
lookup), and resolving a thread without addressing it erodes the next
review's trust. This skill encodes the loop with safety bars so it can
run autonomously and stop at the right moment.

## When this is the right tool

Use it when:
- An open PR has had at least one Copilot review (warm start) or is
  ready for an initial one (cold start)
- The fix work is small enough that each thread can be addressed by a
  code change or a written push-back

Not the right tool when:
- The branch is `main` / `master` / `develop` — the branch guard refuses
- Copilot can't read the diff (`"reviewed 0 of M files"` or
  `"No files were reviewed"` in prior reviews) — split the PR first
- The user wants a general code-review pass — use `/codereview:linus`
- The user wants one bookkeeping step (resolve a single thread) — out of
  scope

## Initial state

Resolve `$OWNER` / `$REPO` / `$PR` via this precedence chain before the
first round:

| Step | Source |
|:-----|:-------|
| 1 | Explicit `--pr <n>` argument from the user |
| 2 | `gh pr view --json number --jq .number` for the current branch |
| 3 | `gh repo view --json nameWithOwner --jq .nameWithOwner` → split on `/` |
| 4 | If no PR for the current branch → abort: "no open PR; run `gh pr create` first" |
| 5 | If >1 PRs for the branch → list and ask the user to pick |

```bash
PR=$(gh pr view --json number --jq .number)
OWNER_REPO=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
OWNER="${OWNER_REPO%/*}"
REPO="${OWNER_REPO#*/}"
BRANCH=$(git branch --show-current)
echo "PR=$PR  REPO=$OWNER/$REPO  BRANCH=$BRANCH"
```

Print the one-line summary so the user can interrupt if anything looks
wrong. Then run the branch guard (see **Safety bars**) before any
mutating step.

## Round 0 — warm or cold start

Decide the first iteration's shape by querying for unresolved Copilot-
authored threads:

```bash
gh api graphql -f owner="$OWNER" -f repo="$REPO" -F pr=$PR -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          comments(first: 1) {
            nodes { author { login } databaseId path line body }
          }
        }
      }
    }
  }
}' | jq '[.data.repository.pullRequest.reviewThreads.nodes[]
        | select(.isResolved | not)
        | select(.comments.nodes[0].author.login == "copilot-pull-request-reviewer")]'
```

| Result | Start mode | First step |
|:-------|:-----------|:-----------|
| Empty array, no prior Copilot review | **Cold** | Skip step 1; jump to step 5 (re-add reviewer) |
| Empty array, prior Copilot review with convergence string | **Converged** | Exit immediately, report merge-ready |
| Non-empty | **Warm** | Full round 1 starting at step 1 |

## The loop

Run these steps in order. Order is load-bearing — push must happen
**before** re-add, otherwise Copilot reviews stale `HEAD` and the loop
diverges.

### Step 1 — Address every unresolved Copilot thread

For each thread, take exactly one path:

- **Fix**: commit a code change that resolves the cited issue. The
  commit's diff must touch the cited `path` (or a clearly equivalent
  location).
- **Push-back**: write a reply that engages with the substance — why
  the comment is wrong, out of scope, or already addressed elsewhere.
  "No" alone does not count.

Never silently resolve. Resolution implies *addressed*; burning that
signal corrupts the next review's input.

### Step 2 — Run the project test command before pushing

Detect the test command in this order:

| File present | Default command |
|:-------------|:----------------|
| `package.json` with `scripts.test` | `npm test` (or the workspace equivalent) |
| `pyproject.toml` | `pytest` (or the project's documented test entry) |
| `Cargo.toml` | `cargo test --release` (per repo CLAUDE.md Rust rule) |
| `Makefile` with a `test` target | `make test` |
| None of the above | Ask the user how to run tests |

If tests fail, abort the round and surface the failure. Do not push a
red branch.

### Step 3 — Commit and push

```bash
git push origin "$BRANCH"
```

Additive commits only. No force-push. No implicit `pull --rebase`
inside the loop — if `git push` is rejected, stop and hand back to the
user.

### Step 4 — Reply to and resolve each thread

For every thread addressed in step 1:

```bash
# Reply (uses the comment's REST databaseId):
gh api -X POST \
  "/repos/$OWNER/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" \
  -f "body=$REPLY_BODY"

# Resolve (uses the thread's GraphQL node id):
gh api graphql -F tid="$THREAD_ID" -f query='
mutation($tid: ID!) {
  resolveReviewThread(input: { threadId: $tid }) { thread { isResolved } }
}'
```

The reply names *what was done* — which commit, which file/line — so
the audit trail makes sense without re-reading the diff.

### Step 5 — Re-add Copilot as reviewer

Capture the current latest Copilot review id **before** re-adding, so
the poll in step 6 can detect a newer one:

```bash
LATEST_REVIEW_ID=$(gh api graphql -f owner="$OWNER" -f repo="$REPO" -F pr=$PR -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviews(last: 20) { nodes { databaseId author { login } } }
    }
  }
}' | jq -r '[.data.repository.pullRequest.reviews.nodes[]
            | select(.author.login == "copilot-pull-request-reviewer")]
            | last.databaseId // 0')

gh pr edit "$PR" --add-reviewer copilot-pull-request-reviewer
```

The slug is `copilot-pull-request-reviewer` — lowercase, hyphenated.
**Never** pass the display name `Copilot`: `gh` lowercases the argument
to `copilot` before lookup, which 404s.

### Step 6 — Poll for the next Copilot review

See **Polling pattern** below.

### Step 7 — Classify

Apply the rule in **Convergence**. The round ends with one of three
outcomes: CONVERGED (stop), ITERATE (next round), ABORT (stop, ask
user).

## Convergence

Three-state rule. A two-state (done / not-done) test misses the case
where Copilot couldn't read the diff — that looks like an empty review
but is not success.

| State | Condition | Action |
|:------|:----------|:-------|
| **CONVERGED** | Latest Copilot review `bodyText` contains `"generated no new comments"` **AND** 0 unresolved Copilot-authored threads | Stop. Print the review URL and "merge-ready". |
| **ITERATE** | Any unresolved Copilot-authored thread, **OR** body contains `"generated"` with a positive comment count | Start the next round at step 1. |
| **ABORT** | Body contains `"reviewed 0 of M files"`, `"No files were reviewed"`, an error string, **OR** the 10-min poll timed out | Stop. Print PR URL, last review id, elapsed time. Hand back to the user. |

Match on `bodyText` (not `body`) — `bodyText` is the rendered plain
text and avoids markdown splitting the trigger phrase across nodes.

## Safety bars

All bars must hold *before* each push and *before* each re-add. A
failed bar aborts the round.

| Bar | Check | On failure |
|:----|:------|:-----------|
| Branch guard | `git branch --show-current` not in `{main, master, develop}` | Abort: "refusing to iterate on protected branch" |
| Max rounds | Rounds in this session ≤ 5 | Stop and ask the user — Copilot may be chasing a moving target |
| Test gate | Test command exits 0 | Surface failure; do not push |
| Resolve-only-if-addressed | Each resolved thread has either a touching commit *in this round* or a written push-back reply | Re-open the thread; resolve nothing speculatively |
| Push hygiene | `git push` succeeds without `--force`, without an implicit `pull --rebase` | Stop and report; let the user untangle |
| No concurrent run | (v1 caveat) Do not start a second `copilot-iterate` session on the same PR | Mentioned for awareness; no lockfile in v1 |

## Polling pattern

The harness blocks long leading `sleep` calls (>300s flushes the prompt
cache). Use a short-sleep `until` loop so the loop *is* the wait:

```bash
DEADLINE=$(( $(date +%s) + 600 ))   # 10-minute timeout
NEW_REVIEW_ID=0
until [ "$NEW_REVIEW_ID" -gt "$LATEST_REVIEW_ID" ]; do
  if [ "$(date +%s)" -ge "$DEADLINE" ]; then
    echo "ABORT: no new Copilot review after 10 min"
    break
  fi
  sleep 30
  NEW_REVIEW_ID=$(gh api graphql -f owner="$OWNER" -f repo="$REPO" -F pr=$PR -f query='
    query($owner: String!, $repo: String!, $pr: Int!) {
      repository(owner: $owner, name: $repo) {
        pullRequest(number: $pr) {
          reviews(last: 20) { nodes { databaseId author { login } } }
        }
      }
    }' | jq -r '[.data.repository.pullRequest.reviews.nodes[]
                | select(.author.login == "copilot-pull-request-reviewer")]
                | last.databaseId // 0')
done
```

Do not use `Bash(run_in_background=true)` here — the next round
genuinely depends on the new review existing; there is no useful
parallel work.

## Per-round log

At the end of each round, print one line so the user can audit
progress:

```
Round N: addressed K threads (J fixed / K-J pushed back), pushed <sha7>, new Copilot review = <id>, classification = ITERATE|CONVERGED|ABORT
```

A round-over-round decline in comment count is the healthy shape. If
comment count is flat or rising across three rounds, the loop is not
converging — stop and surface to the user even before max-rounds.

## Anti-patterns

- **Display name instead of slug.** `--add-reviewer Copilot` → `gh`
  lowercases to `copilot` → 404. Always
  `copilot-pull-request-reviewer`.
- **Re-add before push.** Copilot reviews the prior `HEAD`. Each round
  reviews stale code; the loop never converges.
- **Resolving without addressing.** Burns the meaning of "resolved".
  Future reviews — Copilot or human — can no longer trust resolution
  state.
- **Iterating on `main` / `master`.** Branch guard refuses. Do not
  override.
- **Pushing without running tests.** Violates the repo CLAUDE.md rule
  "Run all tests before each commit."
- **Treating an empty review as convergence.** `"reviewed 0 of M files"`
  is the **ABORT** state, not CONVERGED. Empty ≠ approved.
- **Over-trusting a hallucinated comment.** When Copilot asks for a
  null check on a value typed non-nullable, push back in writing.
  Papering over a hallucination teaches the loop to accept noise.
- **Force-pushing to clean up the round.** Never. The review history is
  part of the audit trail; additive commits only.
- **Naked `sleep 240`.** Use the `until` loop in **Polling pattern** —
  long leading sleeps are blocked and break the prompt cache.
- **Letting the loop run silent.** Print the per-round log every round
  so the user can interrupt if Copilot is not converging.
