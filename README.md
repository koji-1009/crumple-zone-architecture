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

Copy the skill file to your AI agent's configuration:

### Claude Code (project)

```bash
mkdir -p .claude/commands && curl -o .claude/commands/crz.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

### Claude Code (global)

```bash
curl -o ~/.claude/commands/crz.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

### Cursor

```bash
mkdir -p .cursor/rules && curl -o .cursor/rules/crz.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

### Codex

```bash
curl -o AGENTS.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

### Sieve (CSS companion)

Add alongside crz.md when the task involves styling:

#### Claude Code (project)

```bash
curl -o .claude/commands/sieve.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/sieve.md
```

#### Claude Code (global)

```bash
curl -o ~/.claude/commands/sieve.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/sieve.md
```

#### Cursor

```bash
curl -o .cursor/rules/sieve.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/sieve.md
```

For Claude Code, use `/crz` before or during development. Use `/sieve` alongside `/crz` for styling tasks.

## License

MIT
