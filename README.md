# dotclaude

Personal Claude Code plugin marketplace — a curated collection of plugins for code review, design systems, and more.

## Plugins

### codereview

Code review commands with different personas and styles.

| Component | Type | Description |
|:----------|:-----|:------------|
| `/codereview:linus` | Command | Linus Torvalds-style code review with language-aware analysis (C, C++, Rust, Go, docs) |

### design-system

Design system skills for applying consistent visual language to front-end projects.

| Component | Type | Description |
|:----------|:-----|:------------|
| `linear-dark` | Skill | Activates automatically when applying a dark-mode design system (Linear/Vercel-style) to front-end code. Provides color tokens, typography, component patterns, animation specs, and accessibility guidelines. |

### release-automation

Skills for setting up automated release workflows in Rust projects.

| Component | Type | Description |
|:----------|:-----|:------------|
| `knope-rust` | Skill | Activates when setting up knope release automation for Rust projects. Provides knope.toml configuration, SHA-pinned GitHub Actions workflow templates, multi-platform build matrix, crates.io OIDC publishing, and pitfall avoidance for knope 0.22.2. |

### testing

Skills for automated visual verification of web applications during test authoring.

| Component | Type | Description |
|:----------|:-----|:------------|
| `vision-verify` | Skill | Activates when writing or modifying Playwright tests involving UI interactions, visual behavior, animations, or layout. Records browser sessions and analyzes them with Gemini's vision model to catch visual bugs that functional tests miss. |

### dev-journal

Skills for keeping running engineering notes during implementation — decisions, deviations, tradeoffs, and followups.

| Component | Type | Description |
|:----------|:-----|:------------|
| `implementation-notes` | Skill | Activates when implementing a SPEC/PLAN, building a multi-step feature, or when the user asks to keep a running log. Maintains an `implementation-notes.md` (or `.html`) file alongside the code, capturing decisions outside the spec, deviations from spec, tradeoffs, open questions, and followups — written *while* the work happens, not after. |

### thinking-tools

Skills for switching perspectives and reasoning through problems using established mental models.

| Component | Type | Description |
|:----------|:-----|:------------|
| `perspective` | Skill | Activates when the user asks to switch perspectives or think from a specific thinker's viewpoint. Loads one of ten distilled "thinking operating systems" (Elon Musk, Richard Feynman, Charlie Munger, Naval Ravikant, Steve Jobs, Nassim Taleb, Ilya Sutskever, Andrej Karpathy, Paul Graham, Donald Trump) and responds in that persona. Adapted from [nuwa-skill](https://github.com/alchaincyf/nuwa-skill) (MIT). |

### writing-tools

Skills for writing well — removing AI writing patterns with bilingual (English + Chinese) audit support.

| Component | Type | Description |
|:----------|:-----|:------------|
| `humanly` | Skill | Activates when reviewing or rewriting text to remove AI tells (or 去 AI 味 / 潤稿 in Chinese). Three modes (pre-write, review, detect) with severity tiers (P0/P1/P2), context-profile tolerance, and 5-dimension scoring. References cover patterns, word tables, and cheatsheets in both English and Traditional Chinese. Based on [Wikipedia:Signs of AI writing](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing) and [avoid-ai-writing](https://github.com/conorbronsdon/avoid-ai-writing) (MIT). |

### agent-workflow

Skills for packaging context across agent sessions, machines, and tools.

| Component | Type | Description |
|:----------|:-----|:------------|
| `handoff` | Skill | Activates on `/handoff` or when the user wants to package current session state for another agent (a fresh Claude Code session, Codex, ChatGPT, or an agent on another machine). Prints a self-contained markdown handoff document inline — no file written, no worktree created — covering goal, current state, next step, decisions, dead ends, environment, and relevant files. |
| `worktree-handoff` | Skill | Activates when the user wants to start a new branch with a committed handoff context — "open worktree", "worktree handoff", "create a worktree". Creates a git worktree under `.worktrees/`, discovers or creates a matching plan file (defaults to `docs/plans/`; opts into state-machine mode via `BACKLOG` and `DOING` env vars), appends a timestamped Handoff Context section, commits and pushes, then prints the next-session pickup command. |

### retro-tools

Skills for retrospective analysis — conversation retrospectives and post-mortem bug archaeology.

| Component | Type | Description |
|:----------|:-----|:------------|
| `session-retro` | Skill | Activates when the user says "session retro", "retrospective", or "review this session". Scans the conversation for corrections, attributes each error to its source (skill, CLAUDE.md, memory, or tool), proposes targeted fixes that persist, and analyzes token efficiency (subagent dispatches, redundant operations, oversized reads). |
| `post-mortem` | Skill | Activates when the user says "post-mortem", "trace this bug", or describes a regression that appeared in a specific version. Walks git history to find the introduction commit by first locating the fix commit, then traces the causal chain through diffs and commit messages, and produces a structured markdown report (timeline, root cause, prevention measures). Works with any git repo and default branch. |
| `second-opinion` | Skill | Activates when the user says "second opinion", "challenge this plan", "adversarial review", or asks for an independent take on a plan or spec. Dispatches the document to Codex CLI (with a clean-context subagent fallback) for a 5-dimension critique (Completeness, Consistency, Clarity, Scope, Feasibility). Findings are advisory — each must be explicitly adopted by the user. |

### gtm-tools

Skills for go-to-market strategy — codebase-aware GTM document generation with multi-session interview support.

| Component | Type | Description |
|:----------|:-----|:------------|
| `gtm` | Skill | Activates when the user says "gtm", "go to market", "GTM plan", "brand strategy", or asks to generate marketing/positioning documents. Scans the codebase, runs a 5-phase interview (product, audience, competition, brand, channels) across multiple sessions, and produces 4 strategy documents (brand strategy, market landscape, messaging framework, channel playbook) plus a `gtm-state.yaml` for session resume. Templates kept in `references/` for progressive disclosure. |

### pr-workflow

Skills for driving GitHub PR review workflows — Copilot iteration loops, thread bookkeeping.

| Component | Type | Description |
|:----------|:-----|:------------|
| `copilot-iterate` | Skill | Activates when the user says "iterate on Copilot review", "address Copilot feedback", "next round of Copilot", "drive PR to convergence", or `/copilot-iterate`. Runs an autonomous loop on an open PR: address each unresolved Copilot thread (fix or push-back), run tests, push, reply and resolve threads, re-add `copilot-pull-request-reviewer` (the only valid re-trigger), poll for the next review with a 10-minute deadline, then classify as CONVERGED (`"generated no new comments"`), ITERATE, or ABORT. Branch guard refuses `main`/`master`/`develop`; max 5 rounds before forced check-in. Repo-agnostic — auto-detects owner/repo/PR from the current branch. |

## Installation

Add this marketplace and install any plugin:

```shell
/plugin marketplace add /path/to/dotclaude
/plugin install codereview@dotclaude
/plugin install design-system@dotclaude
/plugin install release-automation@dotclaude
/plugin install testing@dotclaude
/plugin install dev-journal@dotclaude
/plugin install thinking-tools@dotclaude
/plugin install writing-tools@dotclaude
/plugin install agent-workflow@dotclaude
/plugin install retro-tools@dotclaude
/plugin install gtm-tools@dotclaude
/plugin install pr-workflow@dotclaude
```

## Adding a Plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`
2. Add commands, skills, or agents under the plugin directory
3. Add an entry to `.claude-plugin/marketplace.json`
4. Run `/plugin marketplace update dotclaude`
