---
name: gtm
description: >
  Generate a complete Go-To-Market strategy by analyzing a codebase and interviewing
  the user. Produces 4 strategy documents covering brand, market landscape, messaging,
  and channel playbook. Supports initial deep research (multi-session with interviews)
  and weekly incremental updates.
  Use when: "gtm", "go to market", "GTM plan", "brand strategy", "market analysis",
  "messaging framework", "generate GTM".
---

# GTM Strategy Generator

A generic, codebase-driven GTM skill. Reads a repository, interviews the user across
multiple sessions, and produces a complete set of strategy documents.

## Output

4 Markdown files in `{repo}/docs/gtm/` plus a state file:

```
docs/gtm/
├── gtm-state.yaml            # Session state (skill internal use)
├── 01-brand-strategy.md       # Who we are
├── 02-market-landscape.md     # The battlefield
├── 03-messaging-framework.md  # How we talk
├── 04-channel-playbook.md     # Where we show up
```

## Modes

| Mode | Trigger | Description |
|------|---------|-------------|
| **Initial** | No `gtm-state.yaml` exists | Deep research: codebase scan → multi-session interviews → draft → review |
| **Update** | `gtm-state.yaml` exists with `status: complete` | Weekly: scan git diff → confirm with user → update affected sections |
| **Resume** | `gtm-state.yaml` exists with `status` != `complete` | Continue from where the last session left off |

---

## State Management

On first run, create `docs/gtm/gtm-state.yaml`:

```yaml
project: {repo name}
repo_path: {absolute path}
created: {YYYY-MM-DD}
last_updated: {YYYY-MM-DD}
last_scan_commit: {commit hash}

status: scanning  # scanning | interviewing | drafting | review | complete

phases:
  scan:
    status: pending      # pending | in_progress | complete
    depth: null           # quick | standard | deep
    completed_at: null
  interview:
    status: pending
    current_topic: null   # product | audience | competition | brand | channels
    completed_topics: []
    pending_topics:
      - product
      - audience
      - competition
      - brand
      - channels
  draft:
    status: pending
    completed_at: null
  review:
    status: pending
    completed_at: null

# Accumulated findings from interviews (append after each session)
findings:
  product: {}
  audience: {}
  competition: {}
  brand: {}
  channels: {}
```

**At the start of every session:**
1. Read `gtm-state.yaml`
2. Display a brief summary: "Last session we covered [X]. Conclusions: [Y]. Today we continue with [Z]."
3. Ask user: continue, go back and revise a topic, or skip to a specific topic.

**After each session:**
1. Update `findings` with new information
2. Update phase statuses
3. Update `last_updated`
4. **Save the state file. Do not auto-commit** — the user owns the git
   decision. Suggest `git add docs/gtm/ && git commit` if they want a
   checkpoint, but never run it for them.

---

## Initial Mode: Full Flow

### Phase 0: Codebase Scan

Ask the user first:

> How well-documented is your codebase?
> A) Well-documented (good README, docs/) → Quick scan
> B) Partially documented → Standard scan
> C) Minimal documentation → Deep scan

**Quick scan:**
- Read README.md, CLAUDE.md, docs/ directory
- Scan top-level directory structure
- Read package.json / Podfile / build.gradle for dependencies

**Standard scan:**
- Everything in quick scan, plus:
- Scan 2 levels of source directory structure
- Read key config files (API routes, database schemas, app manifests)
- Identify major feature modules by directory names

**Deep scan:**
- Everything in standard scan, plus:
- Read source code for each identified module (entry points, main classes)
- Identify features by class/function names and comments
- Map data models and API endpoints

**Output:** Draft feature inventory saved to `findings.product.features` in state file.

Update state: `phases.scan.status: complete`

---

### Phase 1: Product & Stage Interview

Present the feature inventory from Phase 0 and ask ONE question at a time.
Wait for the user's response before asking the next question.

**Questions (adapt based on what the codebase already revealed):**

1. "I found these features/modules in your codebase: [list]. Is anything missing — especially features that are planned or in development?"

2. "What stage is the product at?"
   - Pre-launch (no users yet)
   - Has users (people using it, not yet paying)
   - Has paying customers
   - Growth stage

3. "What's your current primary goal?"
   - Get first users
   - Increase retention
   - Monetize / convert free → paid
   - Expand to new segments

4. "In one sentence, how would you explain what this product does to a stranger?"
   - **Push if vague:** "That's broad. Specifically — who is this person, and what problem does it solve that they couldn't solve before?"

5. "What's the single most important thing your product does better than anything else out there?"
   - **Push if generic:** "Could a competitor say the same thing? What's the part that only you can claim?"

After each answer, briefly summarize your understanding and confirm before moving on.

Update state: `phases.interview.completed_topics: [product]`, save findings.

---

### Phase 2: Audience Interview (Deepest)

This is the most important phase. Push for specificity — categories are not people.

**Questions:**

1. "Describe your most typical user. Not a category — a person. What's their situation? Why did they come to your product?"
   - **Push:** "Can you name a real user (or composite) and describe a specific day in their life where your product matters?"

2. "Are there other distinct types of users? Describe 1-2 more if they exist."
   - For each: situation, goal, pain points, what JTBD they're hiring your product for

3. "Before your product existed, how did these people solve this problem? What did that workaround look like day-to-day?"
   - **Push:** "What specifically was frustrating about that? What moment made them say 'there has to be a better way'?"

4. "Have you directly heard users describe their pain points? What did they actually say — their exact words?"
   - **Push:** "The exact phrasing matters for messaging. 'I need better health tracking' vs 'I'm terrified the medication isn't working' are completely different messages."

5. "Has any user used your product in a way you didn't expect? What surprised you?"

6. "If your product disappeared tomorrow, who would be most upset? Why?"
   - **Push:** "Would they scramble to find an alternative, or just shrug? That tells us how deep the need is."

Update state: `phases.interview.completed_topics: [product, audience]`, save findings.

---

### Phase 3: Competition Research

This phase happens AFTER Product + Audience because we now know what to search for.

**Step 1: Ask the user**

1. "What do you consider your direct competitors? (Other products solving the same problem for the same people)"

2. "What's their biggest weakness, from your users' perspective?"

3. "What's the one thing about your product that competitors can't easily copy?"
   - **Push:** "A feature list isn't a moat. What's the underlying insight or advantage?"

**Step 2: Online research**

Search the web for:
- "[product category] best apps/tools {current year}"
- "[product category] vs [competitor names]"
- "[product category] reviews complaints"
- "[target audience] [problem] solutions"

Read top 3-5 results. For each competitor found:
- What they do well
- Where they fall short
- How they position themselves
- How our product is different

**Step 3: Synthesize**

Present findings to the user:
- "I found these additional competitors: [list]. Any I should add or remove?"
- "Here's how I see the competitive landscape: [summary]. Does this match your understanding?"

Update state: `phases.interview.completed_topics: [product, audience, competition]`, save findings.

---

### Phase 4: Brand & Voice Interview

**Questions:**

1. "If your product were a person walking into a room, how would others describe them?"
   - **Push for personality axes:** "On a scale: funny vs serious? Formal vs casual? Respectful vs irreverent? Enthusiastic vs matter-of-fact?"

2. "Is there a brand whose tone you admire and want to emulate? Why?"

3. "What style or tone is absolutely off-limits for your brand?"
   - **Push:** "Any specific phrases, vibes, or associations you want to avoid? (e.g., 'salesy', 'clinical', 'corporate', 'try-hard')"

4. "What emotional state are your users in when they interact with your product? (anxious? curious? frustrated? hopeful?)"
   - This informs voice calibration — you don't talk to an anxious person the same way you talk to a curious one.

5. "What are the 3-5 values your brand stands for? Not aspirational — things you actually practice."
   - **Push:** "For each value, can you give me a concrete example of how it shows up in your product or communication?"

Update state: `phases.interview.completed_topics: [product, audience, competition, brand]`, save findings.

---

### Phase 5: Channels Interview

**Questions:**

1. "Where do your users spend their time online? Which platforms, communities, forums?"

2. "Where do they go when they have questions about the problem your product solves?"

3. "Which platforms do you currently have a presence on? How's it going?"

4. "Are there any platforms you've tried and abandoned, or deliberately avoiding? Why?"

Update state: `phases.interview.completed_topics: [product, audience, competition, brand, channels]`, save findings.

---

### Phase 6: Draft & Review

Generate all 4 documents based on accumulated findings. Follow the templates below.

Present to user:
> "I've drafted all 4 GTM documents. Would you like to:
> A) Review them one by one
> B) Review all at once
> C) Focus on a specific document"

Iterate based on feedback. When the user approves:
- Update state: `status: complete`, `phases.review.completed_at: {date}`
- Save all files. If the user uses git, suggest committing the whole
  `docs/gtm/` directory in one commit — but do not run `git commit`
  automatically.

---

## Update Mode: Weekly Refresh

When `gtm-state.yaml` exists with `status: complete`:

1. **Check if the repo is git-tracked** (`git rev-parse --is-inside-work-tree`).
   - **If git:** run `git log --oneline --since="7 days ago"` (or since
     `last_scan_commit`) to detect code changes automatically.
   - **If not git:** skip the log step and ask the user directly:
     "What changed in the product since the last scan?"
2. Analyze changes: new features, modified features, removed features.
3. Present changelog to user:
   > "Since last scan, I see these codebase changes: [list]. Any strategic direction changes this week?"
4. Update affected documents (usually 01-brand-strategy § Product Profile and 03-messaging-framework).
5. Update `last_updated` in state file. If using git, also update
   `last_scan_commit` with the current HEAD commit hash. **Save only —
   do not auto-commit.**

---

## Document Templates

The 4 deliverable documents are generated from templates in `references/`. Load each
template only when about to draft the corresponding document in Phase 6 — this
keeps the skill body lean.

| Document | Template |
|----------|----------|
| `01-brand-strategy.md` | `references/01-brand-strategy.md` |
| `02-market-landscape.md` | `references/02-market-landscape.md` |
| `03-messaging-framework.md` | `references/03-messaging-framework.md` |
| `04-channel-playbook.md` | `references/04-channel-playbook.md` |

---

## Interview Style Guide

- **Collaborative but specific.** Don't be confrontational, but don't accept vague answers.
- **Show your work first.** Before asking, present what you inferred from the codebase. This reduces user burden and demonstrates competence.
- **One question at a time.** Wait for the response before asking the next.
- **Push once.** If the answer is vague, ask for specificity once. If the user can't be more specific, note it and move on — they may not have the data yet.
- **Summarize after each topic.** Before moving to the next phase, present a brief summary of what you learned and get confirmation.
- **Respect the user's time.** If they give rich answers quickly, don't artificially slow down. If a topic's answers already cover the next topic, skip it.
- **Mark progress.** After each meaningful exchange, update the state file so the next session can resume cleanly.

---

**Origin**: Adapted from [hanamizuki/solopreneur](https://github.com/hanamizuki/solopreneur/blob/40cfff6/plugins/marketer/skills/gtm/SKILL.md) (MIT).
