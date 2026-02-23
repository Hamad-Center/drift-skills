# Project: drift-skills

AI-powered session tracking that connects git changes to Jira tickets. Available as skills/rules for Claude Code, Cursor, and Codex.

---

## Commit Rules

- NEVER include `Co-Authored-By` lines in commit messages
- NEVER mention Claude, Anthropic, or any AI tool in commit messages
- Keep commit messages concise and human-written in style

---

## What This Repo Is

This repo contains **Drift Skills** — a set of AI assistant instructions (not a standalone app, not a CLI tool, not a CI plugin) that run inside Claude Code, Cursor, and Codex. Three commands:

```
drift-start   →  snapshot starting point + Jira ticket
drift-status  →  read-only progress check
drift-sync    →  deep analysis + code review + Jira comment
```

Each command runs entirely inside the AI assistant's session, using the assistant's ability to read files, run shell commands, and reason about code with no truncation or character limits.

---

## Repo Structure

```
drift-skills/
├── README.md                           # Public-facing documentation
├── CLAUDE.md                           # This file — project context for AI assistants
├── claude-code/                        # Claude Code skills
│   ├── README.md                       # Claude Code-specific install + usage guide
│   ├── drift-start/SKILL.md            # /drift-start skill
│   ├── drift-sync/SKILL.md             # /drift-sync skill
│   └── drift-status/SKILL.md           # /drift-status skill
├── cursor/                             # Cursor rules
│   ├── README.md                       # Cursor-specific install + usage guide
│   ├── drift-start.mdc                 # drift-start rule
│   ├── drift-sync.mdc                  # drift-sync rule
│   └── drift-status.mdc               # drift-status rule
├── codex/                              # Codex skills
│   ├── README.md                       # Codex-specific install + usage guide
│   ├── drift-start/SKILL.md            # drift-start skill
│   ├── drift-sync/SKILL.md             # drift-sync skill
│   └── drift-status/SKILL.md          # drift-status skill
└── docs/
    └── DRIFT_DOCS_DETAILED.md          # Full technical reference (914 lines)
```

---

## Skill Format Differences

### Claude Code (`claude-code/`)
- Format: `SKILL.md` files inside named folders (`drift-start/SKILL.md`)
- Install location: `~/.claude/skills/`
- Frontmatter: `name`, `description`, `allowed-tools`, `user-invocable`, `argument-prompt`
- Invoked with slash commands: `/drift-start`, `/drift-sync`, `/drift-status`
- Arguments via `$ARGUMENTS` variable
- Session history file: `sessions.claude.json`
- Analysis label: `"claude-full-context"` / `"Claude analysis"` / `"Claude deep analysis"`

### Cursor (`cursor/`)
- Format: `.mdc` files (markdown with frontmatter) — flat, no subdirectories
- Install location: `~/.cursor/rules/`
- Frontmatter: `description`, `globs` (empty), `alwaysApply: false`
- Invoked by typing in agent mode chat: `drift-start BAS-10 implement auth`
- Arguments from the user's message text (no `$ARGUMENTS` variable)
- Session history file: `sessions.cursor.json`
- Analysis label: `"cursor-full-context"` / `"Cursor analysis"` / `"Cursor deep analysis"`

### Codex (`codex/`)
- Format: `SKILL.md` files inside named folders (same structure as Claude Code)
- Install location: `~/.agents/skills/`
- Frontmatter: `name`, `description` (no `allowed-tools` or `user-invocable`)
- Invoked via `/skills` or `$` mentions
- Arguments from the user's message text
- Session history file: `sessions.codex.json`
- Analysis label: `"codex-full-context"` / `"Codex analysis"` / `"Codex deep analysis"`

---

## How the Three Commands Work

### `drift-start`

**Purpose:** Mark the starting line for a session.

**Steps (all three tools follow the same logic):**

1. **Verify git repo** — `git rev-parse --git-dir`
2. **Initialize `.drift/`** on first run — creates `config.json`, `sessions.json`, adds `.drift/` to `.gitignore`
3. **Check for existing active session** — if one exists, asks to overwrite
4. **Parse input** — extracts ticket ID via `[A-Za-z][A-Za-z0-9]+-\d+` pattern, normalizes to uppercase. Everything else becomes `plannedWork`
5. **Capture git state** — `git rev-parse HEAD` (full SHA), `git rev-parse --abbrev-ref HEAD` (branch)
6. **Write session to config** — pushes to `sessions` array in `.drift/config.json`. **Capped at 20 sessions** — oldest trimmed from front if exceeded
7. **Jira fallback** — if no `plannedWork` text but Jira is configured, fetches ticket summary + description as plannedWork
8. **Jira credential setup** — checks all 6 credential sources, offers interactive setup if none found

**Session object shape:**
```json
{
  "startedAt": "2025-02-14T16:47:00.000Z",
  "startSha": "a1b2c3d4e5f6...",
  "ticket": "BAS-10",
  "plannedWork": "implement JWT authentication"
}
```

### `drift-status`

**Purpose:** Read-only snapshot. Writes nothing, changes nothing.

**Steps:**

1. **Read `.drift/config.json`** — get active session (last element of `sessions` array)
2. **Gather git data** — `git log --oneline`, `git diff --stat`, `git diff --name-only`, `git status --porcelain` (all relative to `startSha`)
3. **Display status** — session info, commits since start, files changed, uncommitted work
4. **Show session history** — reads `sessions.claude.json`, `sessions.cursor.json`, `sessions.codex.json`, `sessions.json` (shows last 5 from each)
5. **Suggest next steps** — mentions `drift-sync` and report file if it exists

### `drift-sync`

**Purpose:** Deep multi-phase analysis. The core of Drift.

**8 phases, executed sequentially:**

| Phase | Name | What it does |
|-------|------|-------------|
| 1 | Validate session | Read config, extract startSha/ticket/plannedWork |
| 1b | Jira fetch | Resolve credentials, fetch ticket AC (skipped with `--no-jira`) |
| 2 | Gather git data | Full diff, commit log, file lists (added/modified/deleted), uncommitted changes |
| 3 | Deep file analysis | Read full content of every changed file (not just diffs). Skip binary, skip >1000 lines, cap at top 20 if >100 files |
| 4 | Structured analysis | 7 sub-sections: what happened, decisions (SOUND/ACCEPTABLE/QUESTIONABLE), drift from plan, architecture impact, next session items, ticket comment, code review |
| 5 | Write CONTEXT.md | Living context in repo root. Latest session + up to 4 previous. **Migration**: renames `CONTEXT.claude.md` → `CONTEXT.md` if old file exists |
| 6 | Write session JSON | Prepends entry to tool-specific `sessions.*.json`. Capped at 50 entries |
| 7 | Post to Jira | Posts ADF comment to ticket (skipped with `--no-push` or `--no-jira`) |
| 8 | Report | Displays summary, writes `<ticket>-report.md` (lowercase, e.g. `bas-10-report.md`) or `session-report.md` |

**Flags:**
- `--no-push` — skip Jira comment posting only (still fetches ticket for analysis)
- `--no-jira` — skip all Jira integration (no fetch, no post)

---

## Code Review System (Built Into drift-sync Phase 4g)

Every sync reviews all changed files with three signal labels:

| Signal | Meaning | Action |
|--------|---------|--------|
| **CLEAN** | Good practice | None needed |
| **CONCERN** | Worth watching | May become a problem |
| **ISSUE** | Should be fixed | Creates real risk |

**Six review categories** (categories with no findings are omitted):

1. Approach validation
2. Hacky patterns (magic numbers, tight coupling, copy-paste)
3. Best practices (framework idioms, naming, error handling)
4. Architecture soundness (over/under-engineered?)
5. Security concerns (hardcoded secrets, auth gaps, injection)
6. Potential issues (race conditions, resource leaks, edge cases)

Each finding includes file path, line number, and recommendation. Every review ends with a 1-3 sentence overall assessment.

**CONTEXT.md** only includes CONCERN and ISSUE findings. The full review with CLEAN findings appears in `<ticket>-report.md`.

---

## Scoping Rules

These rules prevent false positives in drift analysis:

1. **Only evaluate against written AC** — if the AC doesn't mention tests, missing tests are not drift
2. **If it's not in the AC, it's not missing** — Drift never invents requirements
3. **Scope creep is flagged, not judged** — things built outside AC are surfaced, not marked wrong
4. **Future ticket work is out of scope** — labeled separately as "future work (out of scope)"
5. **No inferred requirements** — only what's literally written in the AC counts
6. **Progress % is concrete** — completed AC items / total AC items

---

## Output Files

### Written to repo root (committed, shared)

| File | Description |
|------|-------------|
| `CONTEXT.md` | Rolling context document. Latest session in full + up to 4 previous sessions summarized. Updated every sync |
| `<ticket>-report.md` | Human-readable session report (e.g. `bas-10-report.md`). Overwritten every sync. Good for PR descriptions |

### Written to `.drift/` (gitignored, never committed)

| File | Description |
|------|-------------|
| `config.json` | Active session state. `{initialized, sessions[]}`. Max 20 sessions |
| `sessions.json` | CLI tool session history |
| `sessions.claude.json` | Claude Code analysis history. Max 50 entries |
| `sessions.cursor.json` | Cursor analysis history. Max 50 entries |
| `sessions.codex.json` | Codex analysis history. Max 50 entries |
| `jira.json` | Jira credentials if saved interactively |
| `.env` | Optional drift-specific env vars |

---

## Data Model

### Session object (in `config.json`)

```json
{
  "startedAt": "ISO-8601 timestamp",
  "startSha": "full 40-char git SHA",
  "ticket": "BAS-10",
  "plannedWork": "description of planned work"
}
```

### Analysis entry (in `sessions.*.json`)

```json
{
  "date": "ISO-8601 timestamp",
  "ticket": "BAS-10",
  "startSha": "a1b2c3d4...",
  "endSha": "f4e5d6c7...",
  "summary": "What was built (3-5 sentences)",
  "decisions": ["HS256 over RS256 — SOUND: ..."],
  "drift": ["Added: password reset (not in AC)", "Missing: token refresh (AC #4)"],
  "architectureImpact": ["New middleware.py for JWT validation"],
  "forNextSession": ["Implement token refresh (AC #4)"],
  "ticketComment": "Full Jira comment text",
  "codeReview": {
    "findings": [
      {
        "signal": "CONCERN|ISSUE|CLEAN",
        "category": "security|approach|hacky|bestpractice|architecture|potential",
        "file": "backend/config.py",
        "finding": "description",
        "recommendation": "what to do"
      }
    ],
    "overall": "1-3 sentence assessment",
    "counts": { "clean": 3, "concern": 1, "issue": 1 }
  },
  "analysisMethod": "claude-full-context|cursor-full-context|codex-full-context",
  "filesAnalyzed": 12,
  "totalFilesChanged": 14,
  "jiraCommentPosted": true
}
```

### Jira credentials (in `jira.json`)

```json
{
  "url": "https://yourteam.atlassian.net",
  "email": "you@company.com",
  "token": "your-api-token"
}
```

---

## Jira Integration

### Credential Resolution Order (first complete set wins)

| Priority | Source |
|----------|--------|
| 1 | Shell env vars: `JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN` |
| 2 | `.env` in project root |
| 3 | `.drift/.env` |
| 4 | `~/.config/drift-nodejs/config.json` (global CLI config) |
| 5 | `.drift/jira.json` (saved interactively) |
| 6 | Interactive prompt |

The `.env` parser handles: `export` prefixes, quoted values, inline comments, trailing whitespace.

### What It Fetches

- Ticket `summary`, `description`, `status`
- Acceptance criteria found in order: custom AC field → checklist in description → subtask summaries

### What It Posts

Structured ADF comment with: progress %, what was done, drift from plan, code quality summary, next steps, blockers.

### Jira Comment Format

```
Drift Session Summary (<Tool> analysis):

Progress: ~70% of this ticket's acceptance criteria

What was done:
JWT auth implemented with login/register endpoints and middleware.

Drift from plan:
Added password reset endpoint not in AC — recommend separate ticket.

Code quality:
1 concern, 1 issue: rate limiting missing on login endpoint

Next steps:
- Implement token refresh (AC item #4)
- Add rate limiting to /login

Blockers:
- None
```

---

## Cross-Tool Compatibility

All three tool versions share the same `.drift/` directory and `CONTEXT.md`. You can switch between tools freely on the same project.

| Shared across all tools | Separate per tool |
|-------------------------|-------------------|
| `.drift/config.json` | `sessions.claude.json` |
| `.drift/jira.json` | `sessions.cursor.json` |
| `.drift/.env` | `sessions.codex.json` |
| `CONTEXT.md` | |
| `<ticket>-report.md` | |

`drift-status` in any tool shows session history from all sources (Claude Code, Cursor, Codex, CLI).

---

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| No changes since start | Reports "No changes" and stops without writing files |
| Binary files | Skipped, noted as "binary file, not analyzed" |
| Files over 1000 lines | Skipped, noted as "too large for inline analysis" |
| More than 100 changed files | Reads top 20 by lines changed, notes the rest |
| No `plannedWork` | "Drift from plan" section omitted entirely |
| Missing `startSha` (after rebase) | Falls back to `git merge-base`, fails gracefully |
| Jira fetch failure | Warns user, continues without ticket context |
| Jira comment failure | Warns user, still writes CONTEXT.md and report |
| Old `CONTEXT.claude.md` exists | Renamed to `CONTEXT.md` via `mv`, history preserved |
| Active session exists on drift-start | Asks to overwrite; if yes, replaces; if no, stops |

---

## CONTEXT.md Format

```markdown
# Project Context

> This file is auto-generated by drift (<tool> skill). It provides deep analysis of recent changes.
> Last updated: <ISO-8601 timestamp>
> Analysis method: <tool>-full-context (no truncation, multi-step file analysis)

# Latest Session

## Session: <date> (<ticket>)

**What happened:**
<3-5 sentences with file paths and approaches>

**Decisions made:**
<each rated SOUND / ACCEPTABLE / QUESTIONABLE>

**Drift from plan:**
<scope additions, reductions, approach changes>

**Architecture impact:**
<new patterns, dependencies, API changes>

**Code review:**
<CONCERN and ISSUE findings only — CLEAN omitted for brevity>
<1-3 sentence overall assessment>

**For next session:**
<remaining AC items, future work, TODOs from code>

# Previous Sessions

## Session: <date> (<ticket>)
<summarized previous sessions, up to 4>
```

---

## Design Decisions

- **Three commands, not one** — start is fast, status is safe (read-only), sync is deliberate (expensive). Combining them would make every status check trigger Jira calls.
- **20-session cap in config.json** — config is the active session store, not the archive. Full history lives in `sessions.*.json` (50 entries).
- **CONTEXT.md in repo root** — meant to be committed and shared. Hiding it in `.drift/` would defeat its purpose as persistent context.
- **Strict AC scoping** — prevents noise from invented requirements. Measures what you said you'd do vs what you did.
- **drift-status writes nothing** — provably read-only means no hesitation to run it frequently.
- **CLEAN findings in code review** — acknowledges good practices, not just problems. Builds trust in the review signal.

---

## Conventions When Editing Skills

- All three tool versions must stay functionally identical (same logic, same phases, same output format)
- Only the invocation mechanism, frontmatter, argument access, session history filename, and analysis labels differ between tools
- Ticket IDs are always normalized to uppercase in session data
- Report filenames use lowercase ticket ID (e.g. `bas-10-report.md`, not `BAS-10-report.md`)
- CONTEXT.md header says the tool name (e.g. "Claude Code skill" vs "Cursor rule" vs "Codex skill")
- Session JSON analysis method uses tool-specific labels: `claude-full-context`, `cursor-full-context`, `codex-full-context`
- When adding features or fixing bugs, update all three tool versions
- Test changes against the expected output format documented in `docs/DRIFT_DOCS_DETAILED.md`

---

## Installation (for users)

### Claude Code
```bash
cp -r claude-code/* ~/.claude/skills/
```

### Cursor
```bash
cp cursor/* ~/.cursor/rules/
```

### Codex
```bash
cp -r codex/* ~/.agents/skills/
```
