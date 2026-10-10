# Polling for the next Copilot review

Step 6 of `copilot-iterate` runs this after Step 5 re-adds the
reviewer. `$OWNER`, `$REPO`, `$PR`, `$BRANCH`, and `$LATEST_REVIEW_ID`
come from earlier steps.

## Polling pattern

Three attempts, 5-minute window each — **15 minutes total** before
the loop concludes Copilot is unresponsive. Each attempt is a
**separate Bash invocation** so the 10-minute Bash timeout has
margin; the model orchestrates the three-attempt sequence and breaks
early on the first attempt that returns a new review id. Within each
attempt, an `until` loop polls every 30s up to the 5-min window —
that keeps the wait responsive without violating the harness's
leading-sleep block.

Per attempt (substitute `$ATTEMPT` 1..3):

```bash
ATTEMPT=N            # 1, 2, or 3
DEADLINE=$(( $(date +%s) + 300 ))   # 5-min window
NEW_REVIEW_ID=0
until [ "$NEW_REVIEW_ID" -gt "$LATEST_REVIEW_ID" ]; do
  if [ "$(date +%s)" -ge "$DEADLINE" ]; then
    echo "Attempt $ATTEMPT/3: 5-min window expired, no new Copilot review"
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
[ "$NEW_REVIEW_ID" -gt "$LATEST_REVIEW_ID" ] \
  && echo "Attempt $ATTEMPT/3: new review = $NEW_REVIEW_ID"
```

Orchestration:
- New review id detected on any attempt → break, go to **Step 7 — Classify**.
- Attempt 3 also returns the old id → ABORT via **Abort summary** below.

Do not use `Bash(run_in_background=true)` here — the next round
genuinely depends on the new review existing; there is no useful
parallel work.

## Abort summary

When the 3-attempt poll exhausts without a new Copilot review, do not
silently exit. Print a structured summary so the user can decide the
next move without re-deriving state:

```
ABORT: Copilot silent after 3 polls × 5 min (15 min total)

What was tried in round $ROUND:
  • Addressed $K threads ($J fixed / $((K-J)) pushed back)
  • Pushed <sha7> to $BRANCH (tests green)
  • Re-added copilot-pull-request-reviewer
  • Polled review feed every 30s across three 5-min windows
  • Latest Copilot review id stayed $LATEST_REVIEW_ID

Last known PR state:
  • https://github.com/$OWNER/$REPO/pull/$PR @ <sha7>
  • Branch: $BRANCH
  • Unresolved Copilot-authored threads: <count>

Suggested next actions (pick one):
  1. Re-trigger Copilot manually:
       gh pr edit $PR --add-reviewer copilot-pull-request-reviewer
  2. Request a human reviewer:
       gh pr edit $PR --add-reviewer <username>
  3. Re-run /pr-workflow:copilot-iterate later — pushed commits and
     resolved threads persist across sessions.
  4. Check Copilot service health: https://www.githubstatus.com/
```

The abort is **not a failure of the PR** — it is a failure of Copilot
to respond in 15 minutes. The branch, the addressed threads, and the
resolved-thread audit trail are intact; resuming later (or handing
off to a human reviewer) is safe.
