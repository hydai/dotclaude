# dotclaude

This is a Claude Code plugin marketplace repository. It contains multiple plugins, each providing commands, skills, or agents.

## Repository Structure

```
.claude-plugin/
  marketplace.json          # Plugin registry — lists all available plugins
plugins/
  <name>/
    .claude-plugin/
      plugin.json           # Plugin manifest (name, description, version)
    commands/               # Slash commands (user-invoked via /<plugin>:<command>)
      <command>.md
    skills/                 # Skills (model-invoked automatically based on context)
      <skill-name>/
        SKILL.md            # Skill definition with frontmatter (name, description)
        references/         # Supporting reference files for progressive disclosure
```

## Conventions

- **Plugin names** are kebab-case and generic enough to host multiple related components (e.g., `design-system` not `linear-dark-theme`)
- **Command files** are markdown with YAML frontmatter (`description`, `allowed-tools`, `argument-hint`)
- **Skill files** use YAML frontmatter with `name` and `description` — the description is the semantic trigger that tells Claude when to activate the skill
- **SKILL.md should stay under 2000 words** — split detailed specs into `references/` files for progressive disclosure
- **plugin.json** must have `name`, `description`, and `version` fields
- **marketplace.json** must list every plugin with `name`, `source` (relative path), and `description`

## Current Plugins

| Plugin | Components |
|:-------|:-----------|
| `example` | `hello` command |
| `codereview` | `linus` command |
| `design-system` | `linear-dark` skill |

## Adding a New Plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`
2. Add commands under `commands/` or skills under `skills/<skill-name>/SKILL.md`
3. Add an entry to `.claude-plugin/marketplace.json`
4. Update this file's "Current Plugins" table
5. Update `README.md` with the new plugin and its components
