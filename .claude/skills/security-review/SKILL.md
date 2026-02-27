---
name: security-review
description: |
  Pre-commit security review for Open Biosciences repos. Scans staged files
  for secrets, private paths, and config issues using a 3-agent team.
  Use when: "security scan", "pre-commit check", "review for secrets",
  "check for vulnerabilities", before committing or pushing code.
---

# Pre-Commit Security Review

Orchestrate a 4-agent security review before committing code across the Open Biosciences workspace. A **Coordinator** identifies repos with changes, dispatches 3 specialist agents in parallel, collects findings, and produces a final verdict.

## Severity Tiers

| Severity | Behavior | Examples |
|----------|----------|---------|
| CRITICAL | BLOCK commit | Real API keys, passwords, JWT tokens |
| HIGH | BLOCK commit | .env file staged, connection strings with credentials |
| MEDIUM | WARN only | Private paths (`/home/user/...`), internal hostnames |
| LOW | WARN only | Stale predecessor references, suspicious placeholders |

## Team Roles

| Agent | Role | Runs | Checks |
|-------|------|------|--------|
| Coordinator | Identify changed repos, dispatch agents, produce verdict | Sequential (wraps all) | Repo discovery, result aggregation |
| Secrets Scanner | Detect real API keys, passwords, tokens | Parallel | Regex patterns for AWS, OpenAI, GitHub, JWT keys |
| Path Sanitizer | Detect private filesystem paths, staged .env files | Parallel | Home dirs, private IPs, .env detection |
| Config Validator | Validate .env.example, .mcp.json, .gitignore hygiene | Parallel | Placeholder values, missing gitignore rules |

## Instructions (Coordinator)

When this skill is invoked, you (Claude) act as the Coordinator. Follow these steps exactly.

### Step 1: Identify repos with changes

Run the following to discover which repos have staged or untracked changes:

```bash
for repo in biosciences-architecture biosciences-skills biosciences-program biosciences-mcp biosciences-memory biosciences-deepagents biosciences-temporal biosciences-evaluation biosciences-research biosciences-education biosciences-workspace-template; do
  cd "/home/donbr/open-biosciences/$repo"
  changes=$(git status -s 2>/dev/null)
  if [ -n "$changes" ]; then
    echo "$repo"
  fi
done
```

Collect the list of changed files across all repos with changes:

```bash
cd "/home/donbr/open-biosciences/$repo" && git status -s | awk '{print $NF}'
```

If no repos have changes, report `CLEAN` immediately and stop.

### Step 2: Dispatch 3 specialist agents in parallel

Use the **Task tool** (`subagent_type: general-purpose`) to launch all 3 agents simultaneously. Pass each agent:
- The list of repos with changes
- The absolute paths to all changed files
- Their specific prompt (see sections below)

### Step 3: Collect results

Wait for all 3 agents to complete. Each returns findings in the format:
`file:line | SEVERITY | description`

### Step 4: Produce verdict

Apply the verdict logic (see below) and emit the final report.

## Secrets Scanner Agent Prompt

> You are the Secrets Scanner agent for a pre-commit security review. Your job is to detect real API keys, passwords, and tokens in changed files across the Open Biosciences workspace.
>
> **Changed repos:** {repos_list}
> **Changed files (absolute paths):** {files_list}
>
> ### Instructions
>
> 1. Run these grep patterns against ALL changed files. Use the Grep tool or bash grep. Replace `<files>` with the actual file paths.
>
> ```bash
> # General secrets: API keys, passwords, tokens with real-looking values
> grep -rniE '(api[_-]?key|secret|password|token)\s*[=:]\s*["'"'"']?[a-zA-Z0-9_/+=-]{20,}' <files>
>
> # AWS access keys
> grep -rniE 'AKIA[A-Z0-9]{16}' <files>
>
> # OpenAI API keys
> grep -rniE 'sk-[a-zA-Z0-9]{20,}' <files>
>
> # GitHub Personal Access Tokens
> grep -rniE 'ghp_[a-zA-Z0-9]{36}' <files>
>
> # GitHub OAuth tokens
> grep -rniE 'gho_[a-zA-Z0-9]{36}' <files>
>
> # JWT tokens
> grep -rniE 'eyJ[a-zA-Z0-9+/=]{20,}\.' <files>
> ```
>
> 2. **Filter out false positives.** For each match, check if the value contains any of these placeholder indicators. If it does, it is NOT a finding:
>    - `your_`, `placeholder`, `example`, `here`, `changeme`, `xxx`, `TODO`, `CHANGE_ME`, `INSERT`, `dummy`, `fake`, `test_`, `sample`
>
> 3. **Filter out environment variable references.** These are NOT findings:
>    - `$API_KEY`, `${API_KEY}`, `os.environ["API_KEY"]`, `os.getenv("API_KEY")`, `process.env.API_KEY`
>    - Any pattern where the matched value is a variable dereference rather than a literal
>
> 4. **Filter out documentation patterns.** These are NOT findings:
>    - Lines inside markdown code fences that show example patterns
>    - Comments explaining what a key looks like (e.g., `# API keys start with sk-`)
>
> 5. **Severity assignment:**
>    - CRITICAL: Real API keys, passwords, JWT tokens, AWS keys, OpenAI keys, GitHub tokens
>    - HIGH: Connection strings containing credentials, database passwords
>
> 6. **Report format** (one line per finding):
> ```
> file:line | CRITICAL | Real API key detected (value redacted: sk-...XXXX)
> file:line | HIGH | Password in connection string (value redacted)
> ```
>
> If no real secrets are found, report: `No findings.`
>
> **IMPORTANT:** Always redact actual secret values in your report. Show only the prefix and last 4 characters, or describe the pattern.

## Path Sanitizer Agent Prompt

> You are the Path Sanitizer agent for a pre-commit security review. Your job is to detect private filesystem paths, internal network addresses, and staged .env files.
>
> **Changed repos:** {repos_list}
> **Changed files (absolute paths):** {files_list}
>
> ### Instructions
>
> 1. **Check for staged .env files** (HIGH severity). For each repo with changes:
>
> ```bash
> cd "/home/donbr/open-biosciences/{repo}"
> # Detect any .env files (not .env.example) in the repo
> find . -name '.env' -not -name '.env.example' -not -path '*/.git/*' 2>/dev/null
> # Detect staged .env files specifically
> git diff --cached --name-only 2>/dev/null | grep '\.env$' || true
> git status -s | grep '\.env$' || true
> ```
>
> 2. **Scan for private filesystem paths** (MEDIUM severity). Run against all changed files:
>
> ```bash
> # Private home directory paths
> grep -rniE '/home/[a-zA-Z0-9_]+/' <files>
> ```
>
> For each match, check context:
> - If the path is in a CLAUDE.md, SKILL.md, or documentation file that references workspace layout, it is acceptable. Report as LOW severity with note "workspace reference."
> - If the path is in source code (.py, .ts, .js, .json, .yaml, .toml), report as MEDIUM severity.
> - If the path is in a config file that gets deployed, report as HIGH severity.
>
> 3. **Scan for private IP addresses** (MEDIUM severity). Run against all changed files:
>
> ```bash
> # RFC 1918 private IPs
> grep -rniE '\b(10\.\d+\.\d+\.\d+|192\.168\.\d+\.\d+|172\.(1[6-9]|2[0-9]|3[01])\.\d+\.\d+)\b' <files>
> ```
>
> Filter out:
> - `127.0.0.1` and `0.0.0.0` (standard localhost references in configs) — these are NOT private IPs
> - IP addresses inside comments explaining network architecture
>
> 4. **Severity assignment:**
>    - HIGH: .env file staged for commit
>    - MEDIUM: Private paths in source code, private IPs in config
>    - LOW: Workspace path references in documentation, stale predecessor paths
>
> 5. **Report format** (one line per finding):
> ```
> file:line | HIGH | Staged .env file detected
> file:line | MEDIUM | Private home path in source code: /home/donbr/...
> file:line | LOW | Workspace path reference in documentation (acceptable)
> ```
>
> If no issues are found, report: `No findings.`

## Config Validator Agent Prompt

> You are the Config Validator agent for a pre-commit security review. Your job is to validate that .env.example files, .mcp.json files, and .gitignore files follow security best practices.
>
> **Changed repos:** {repos_list}
> **Changed files (absolute paths):** {files_list}
>
> ### Instructions
>
> 1. **Validate .env.example files** (HIGH severity for real values). For each `.env.example` found in changed repos:
>
> ```bash
> find "/home/donbr/open-biosciences/{repo}" -name '.env.example' -not -path '*/.git/*'
> ```
>
> For each file found, check every line with a value assignment:
> - If a value is a long alphanumeric string (20+ chars) that does NOT contain `your_`, `placeholder`, `here`, `example`, `changeme`, `TODO`, `xxx` — it may be a real credential leaked into the example file. Report as HIGH.
> - Acceptable placeholder patterns: `your_api_key_here`, `changeme`, `sk-your-key-here`, empty string, `TODO`, `xxx`
>
> 2. **Validate .mcp.json files** (HIGH severity for embedded secrets). For each `.mcp.json` found:
>
> ```bash
> find "/home/donbr/open-biosciences/{repo}" -name '.mcp.json' -not -path '*/.git/*'
> ```
>
> Check that no `password`, `secret`, `token`, or `api_key` fields contain non-empty, non-placeholder values. Environment variable references like `${API_KEY}` are acceptable.
>
> 3. **Validate .gitignore files** (MEDIUM severity for missing rules). For each repo with changes:
>
> ```bash
> cat "/home/donbr/open-biosciences/{repo}/.gitignore" 2>/dev/null
> ```
>
> Verify these entries exist:
> - `.env` (must be present — blocks accidental .env commits)
> - `.env.local` (recommended)
> - `*.pem` (recommended for key files)
>
> If `.gitignore` is missing entirely, report as MEDIUM.
> If `.env` rule is missing from .gitignore, report as MEDIUM.
>
> 4. **Severity assignment:**
>    - HIGH: Real credential values in .env.example, secrets in .mcp.json
>    - MEDIUM: Missing .env rule in .gitignore, missing .gitignore entirely
>    - LOW: Missing recommended rules (.env.local, *.pem)
>
> 5. **Report format** (one line per finding):
> ```
> file:line | HIGH | Real value in .env.example: OPENAI_API_KEY has non-placeholder value
> file | MEDIUM | .gitignore missing .env exclusion rule
> file | LOW | .gitignore missing recommended *.pem exclusion
> ```
>
> If no issues are found, report: `No findings.`

## Verdict Logic

After collecting results from all 3 agents, apply this logic:

```
If any CRITICAL or HIGH findings:
  -> BLOCKED — list all blocking findings
  -> "Do NOT proceed with commit. Fix these issues first."

If only MEDIUM or LOW findings:
  -> PASS WITH WARNINGS — list all warnings
  -> "Commit may proceed. Review warnings above."

If no findings:
  -> CLEAN — "No security issues found. Safe to commit."
```

## Output Format

Emit the final report using this exact template:

```
Pre-Commit Security Review
Repos scanned: N | Files scanned: N

=== Secrets Scanner ===
Status: CLEAN | N WARNINGS | N BLOCKING
[findings if any]

=== Path Sanitizer ===
Status: CLEAN | N WARNINGS | N BLOCKING
[findings if any]

=== Config Validator ===
Status: CLEAN | N WARNINGS | N BLOCKING
[findings if any]

=== Verdict: [CLEAN | PASS WITH WARNINGS | BLOCKED] ===
[summary message]
```

Status indicators per section:
- `CLEAN` — no findings from this agent
- `N WARNINGS` — only MEDIUM/LOW findings (N = count)
- `N BLOCKING` — any CRITICAL/HIGH findings (N = count)

## Integration Note

This skill is also invoked automatically as **Step 8** of the `migration-team` workflow. When called from the migration workflow, the Coordinator receives the list of repos from the migration context rather than discovering them via `git status`.
