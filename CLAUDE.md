# CLAUDE.md — platform-skills

## Session Priming (REQUIRED)

Session priming is required before any work in this repo. See the workspace-level
[`../CLAUDE.md`](../CLAUDE.md) §Session Priming for the mandatory Graphiti queries.

## Purpose

Platform-aligned developer skills for the Open Biosciences platform. This repo is co-owned by the **Quality & Skills Engineer** (Agent 8).

Platform skills are developer-facing tools that scaffold, validate, and enforce project conventions. They are distinct from **domain skills** (research-facing, in `biosciences-skills`) which encode life-sciences research workflows.

## Contents

### Commands (`.claude/commands/`)

| Command | Purpose | Trigger |
|---------|---------|---------|
| `scaffold-fastmcp` | Scaffold a new FastMCP server with standard project structure | `/scaffold-fastmcp <api-name>` |
| `scaffold-fastmcp-v2` | Updated v2 scaffold with ADR-006 Single Writer pattern | `/scaffold-fastmcp-v2 <api-name>` |

### Skills (`.claude/skills/`)

| Skill | Purpose | Trigger |
|-------|---------|---------|
| `security-review` | Pre-commit security review (secrets, paths, config hygiene) | "security scan", "pre-commit check" |

## Domain Skills vs Platform Skills

| Aspect | Domain Skills (`biosciences-skills`) | Platform Skills (`platform-skills`) |
|--------|--------------------------------------|-------------------------------------|
| Audience | Researchers, domain experts | Developers, platform engineers |
| Examples | CRISPR workflows, genomics queries | Server scaffolding, security review |
| Team Topology | Value-aligned (research-facing) | Platform-aligned (developer-facing) |
| Owner | Quality & Skills Engineer (Agent 8) | Quality & Skills Engineer (Agent 8) |

## Dependencies

- **Upstream**: `biosciences-architecture` (ADR-001 patterns, ADR-002 skill structure, ADR-004 FastMCP lifecycle)
- **Downstream**: `biosciences-mcp` (scaffold commands target this repo's structure)

## Conventions

- Skills follow the ADR-002 structure: `.claude/skills/{name}/SKILL.md`
- Commands follow Claude Code convention: `.claude/commands/{name}.md`
- All scaffold templates target the `biosciences_mcp` package layout
