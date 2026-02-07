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

## Installation

Add this marketplace and install any plugin:

```shell
/plugin marketplace add /path/to/dotclaude
/plugin install codereview@dotclaude
/plugin install design-system@dotclaude
```

## Adding a Plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`
2. Add commands, skills, or agents under the plugin directory
3. Add an entry to `.claude-plugin/marketplace.json`
4. Run `/plugin marketplace update dotclaude`
