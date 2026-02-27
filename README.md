# platform-skills

Platform-aligned developer skills for the [Open Biosciences](https://github.com/open-biosciences) platform.

Owned by the **Quality & Skills Engineer** (Agent 8). This repo provides developer-facing tools that scaffold, validate, and enforce project conventions.

## What's Here

### Scaffold Commands

Two commands in `.claude/commands/` for creating new MCP servers:

| Command | Description |
|---------|-------------|
| `scaffold-fastmcp` | Generate a new FastMCP server with standard project structure (ADR-001) |
| `scaffold-fastmcp-v2` | Updated v2 scaffold with ADR-006 Single Writer client pattern |

### Security Review Skill

A pre-commit security review skill in `.claude/skills/security-review/` that orchestrates a 4-agent review team:
- **Secrets Scanner** — detects real API keys, passwords, tokens
- **Path Sanitizer** — detects private filesystem paths, staged .env files
- **Config Validator** — validates .env.example, .mcp.json, .gitignore hygiene

## Domain Skills vs Platform Skills

This repo contains **platform skills** (developer-facing). Research-facing **domain skills** live in [biosciences-skills](https://github.com/open-biosciences/biosciences-skills):

| This Repo (Platform) | biosciences-skills (Domain) |
|-----------------------|-----------------------------|
| Server scaffolding | CRISPR screen workflows |
| Security review | Genomics queries |
| Developer tooling | Pharmacology lookups |

The split follows [Team Topologies](https://teamtopologies.com/) — platform teams provide developer experience; stream-aligned teams provide domain value.

## Related Repos

- [biosciences-skills](https://github.com/open-biosciences/biosciences-skills) — 6 domain skills (research-facing)
- [biosciences-architecture](https://github.com/open-biosciences/biosciences-architecture) — ADRs and governance
- [biosciences-mcp](https://github.com/open-biosciences/biosciences-mcp) — the MCP servers that scaffold commands target
- [biosciences-program](https://github.com/open-biosciences/biosciences-program) — SpecKit commands and migration coordination

## License

MIT
