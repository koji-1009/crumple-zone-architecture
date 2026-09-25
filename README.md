# Crumple Zone Architecture

An architecture for building healthy web applications by trusting the browser. Design from failure modes.

For web applications where correctness and maintainability matter. Built on MPA + Islands.

What it provides:

* Fewer bugs — client state is structurally minimized. Less state means fewer things to break
* Secure by default — following the rules produces secure implementations without extra effort
* Maintainable — the frontend stays thin enough to rewrite when needed
* AI-agent compatible — the skill file gives AI consistent implementation criteria

Principles:

A reload reconstructs the correct state. Build the user experience together with the browser.
See [architecture.md](architecture.md) for full design principles.

Japanese: [ja/](ja/)

## Documents

[**skill/crz.md**](skill/crz.md) — Astro implementation skill. Self-contained. Pass to an AI agent during development.

[**skill/sieve.md**](skill/sieve.md) — CSS implementation skill. Companion to crz.md. Pass alongside crz.md when the task involves styling.

[**architecture.md**](architecture.md) — Design principles behind the skill's rules. Add as context when reviewing AI-generated code or resolving edge cases.

[**extensions.md**](extensions.md) — Patterns beyond MPA + Islands: real-time updates, cache layers, optimistic updates, and their security considerations.

[**operations.md**](operations.md) — How to deploy CRZ in AI-driven development: document roles, two-agent separation, and skill prompt authoring.

## Setup

### Claude Code (plugin)

This repository is a Claude Code plugin marketplace. Add it, then install crz and, for styling tasks, sieve:

```bash
claude plugin marketplace add koji-1009/crumple-zone-architecture
claude plugin install crz@crumple-zone-architecture
claude plugin install sieve@crumple-zone-architecture
```

Plugins install at user scope by default; pass `--scope project` or `--scope local` to change it. The skills are invoked as `/crz:crz` and `/sieve:sieve`.

Auto-update is off by default for third-party marketplaces. Enable it under `/plugin` → Marketplaces, or update manually:

```bash
claude plugin marketplace update crumple-zone-architecture
claude plugin update crz@crumple-zone-architecture
claude plugin update sieve@crumple-zone-architecture
```

Migrating from a copied skill file: remove the copies so only the plugin serves the guidance:

```bash
rm -rf .claude/skills/crz .claude/skills/sieve ~/.claude/skills/crz ~/.claude/skills/sieve
```

### Codex (plugin)

Codex reads the same marketplace. Add it, then install crz and, for styling tasks, sieve:

```bash
codex plugin marketplace add koji-1009/crumple-zone-architecture
codex plugin add crz@crumple-zone-architecture
codex plugin add sieve@crumple-zone-architecture
```

Migrating from the previous AGENTS.md install: remove the crz.md content from AGENTS.md, or it keeps serving outdated guidance on every task.

Otherwise, copy the skill file to your AI agent's configuration:

### Claude Code (project)

```bash
mkdir -p .claude/skills/crz && curl -o .claude/skills/crz/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/plugins/crz/skills/crz/SKILL.md
```

### Claude Code (global)

```bash
mkdir -p ~/.claude/skills/crz && curl -o ~/.claude/skills/crz/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/plugins/crz/skills/crz/SKILL.md
```

Migrating from the previous slash-command install: remove the old file, or it keeps serving outdated guidance under the same `/crz` name:

```bash
rm -f .claude/commands/crz.md ~/.claude/commands/crz.md
```

### Cursor

```bash
mkdir -p .cursor/rules && curl -o .cursor/rules/crz.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

### Codex (project)

```bash
mkdir -p .agents/skills/crz && curl -o .agents/skills/crz/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/plugins/crz/skills/crz/SKILL.md
```

### Codex (global)

```bash
mkdir -p ~/.agents/skills/crz && curl -o ~/.agents/skills/crz/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/plugins/crz/skills/crz/SKILL.md
```

### Sieve (CSS companion)

Add alongside crz.md when the task involves styling:

#### Claude Code (project)

```bash
mkdir -p .claude/skills/sieve && curl -o .claude/skills/sieve/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/plugins/sieve/skills/sieve/SKILL.md
```

#### Claude Code (global)

```bash
mkdir -p ~/.claude/skills/sieve && curl -o ~/.claude/skills/sieve/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/plugins/sieve/skills/sieve/SKILL.md
```

Migrating from the previous slash-command install:

```bash
rm -f .claude/commands/sieve.md ~/.claude/commands/sieve.md
```

#### Cursor

```bash
curl -o .cursor/rules/sieve.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/sieve.md
```

#### Codex (project)

```bash
mkdir -p .agents/skills/sieve && curl -o .agents/skills/sieve/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/plugins/sieve/skills/sieve/SKILL.md
```

#### Codex (global)

```bash
mkdir -p ~/.agents/skills/sieve && curl -o ~/.agents/skills/sieve/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/plugins/sieve/skills/sieve/SKILL.md
```

For Claude Code and Codex, the skills load automatically when the task matches their frontmatter description — crz for Astro implementation, sieve for styling. In Claude Code, `/crz` and `/sieve` (`/crz:crz` and `/sieve:sieve` when installed as plugins) invoke them explicitly, which remains the deterministic path. Forgetting to invoke a command is a silent failure; automatic loading turns it into recovery. The `skill/plugins/` files are copies of `skill/crz.md` and `skill/sieve.md` with skill frontmatter, packaged as plugins that Claude Code and Codex both read; Cursor uses the plain files directly.

## License

MIT
