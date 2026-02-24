# Drift Skills

Instruction files for Claude Code, Cursor, and Codex that track coding sessions against Jira tickets.

---

## What Drift Is

Drift is a set of markdown instruction files that your AI assistant follows. You copy them into your tool's skills or rules directory, and they become available as commands. The AI assistant does all the work — reading files, running git commands, calling the Jira API, writing analysis — by following the instructions in the skill files.

There is nothing to install, no dependencies, no runtime. Just files that tell your AI assistant how to track sessions. Works in any git repository.

Three commands:

- **drift-start** — start tracking a session against a Jira ticket
- **drift-status** — see what changed so far (read-only, writes nothing)
- **drift-sync** — analyze everything, review the code, write CONTEXT.md, update Jira

---

## The Problem

**Scope creep goes unnoticed.** You open a ticket, start coding with an AI assistant, and two hours later you've built features nobody asked for because the AI kept suggesting "while we're here" improvements. Drift compares what you built against the ticket's written acceptance criteria and flags anything extra as a scope addition.

**Acceptance criteria get forgotten.** The ticket says one thing, your code says another, nobody knows until QA. Drift fetches the AC from Jira and uses it as the authoritative checklist for every analysis — it's always visible, not just something you checked at the start.

**Context evaporates between sessions.** You close the laptop and lose all the decisions, trade-offs, and next steps that weren't committed. Drift writes `CONTEXT.md` to the repo — a rolling document with the full analysis from your latest session plus up to 4 previous sessions. It survives laptop closes, tool switches, and AI memory resets.

---

## Install

Clone this repo:

```bash
git clone git@github.com:Hamad-Center/drift-skills.git
cd drift-skills
```

Then copy the files for your tool:

### Claude Code

```bash
mkdir -p ~/.claude/skills
cp -r claude-code/* ~/.claude/skills/
```

Invoke with slash commands: `/drift-start BAS-10`, `/drift-status`, `/drift-sync`

### Cursor

```bash
mkdir -p ~/.cursor/rules
cp cursor/* ~/.cursor/rules/
```

Invoke by typing in agent mode chat: `drift-start BAS-10`, `drift-status`, `drift-sync`

### Codex

```bash
mkdir -p ~/.agents/skills
cp -r codex/* ~/.agents/skills/
```

Invoke via `/skills` or `$` mentions: `drift-start BAS-10`, `drift-status`, `drift-sync`

**Prerequisites:** Git. Nothing else.

Each tool folder has its own README with detailed setup instructions.

---

## Quick Start

```
# Start tracking a Jira ticket
drift-start BAS-10

# Drift saves your git SHA and timestamp.
# If Jira is configured, it fetches the ticket's summary and description.
# You see a confirmation with branch, SHA, ticket, and planned work.

# ... write code with your AI assistant ...

# Check how much has changed (read-only, writes nothing)
drift-status

# ... keep coding ...

# Run the full analysis
drift-sync

# Drift reads every changed file in full, fetches the ticket's AC from Jira,
# compares what you built vs what the AC says, reviews the code, then:
#   - writes CONTEXT.md (rolling context document, committed to git)
#   - writes bas-10-report.md (session report, ready for a PR description)
#   - posts a progress comment to BAS-10 in Jira

# The session stays active. You can keep coding and run drift-sync again.
# It only ends when you start a new session:
drift-start BAS-11
```

---

## Command Reference

### `drift-start`

Starts a new session. Records the current git state and links it to a Jira ticket.

**Usage:**

```
drift-start BAS-10
drift-start BAS-10 implement JWT authentication
```

The ticket ID is required — Drift will ask for one if you don't provide it. The description after the ticket ID is optional.

**What it does:**

1. Verifies this is a git repo
2. On first run: creates `.drift/` directory, `config.json`, `sessions.json`, and adds `.drift/` to `.gitignore`
3. If an active session already exists: shows it and asks if you want to replace it. If you say yes, the old session is removed from the sessions array (not archived). If you say no, stops.
4. Parses your input: extracts the ticket ID (normalized to uppercase), everything else becomes `plannedWork`
5. Saves the session to `.drift/config.json`:

```json
{
  "startedAt": "2025-02-14T16:47:00.000Z",
  "startSha": "a1b2c3d4e5f67890abcdef1234567890abcdef12",
  "ticket": "BAS-10",
  "plannedWork": "implement JWT authentication"
}
```

Four fields only: `startedAt`, `startSha`, `ticket`, `plannedWork`. The branch is not stored — it's read from git at display time.

6. If you provided a ticket but no description, and Jira is configured: fetches the ticket's summary and description from Jira and uses that as `plannedWork`
7. Sessions array is capped at 20. Oldest entries are trimmed when exceeded.
8. If Jira isn't configured yet: offers to collect your credentials interactively and saves them to `.drift/jira.json`

**Output:**

```
Session started
  Branch: feature/auth
  SHA:    a1b2c3d
  Ticket: BAS-10
  Plan:   implement JWT authentication
```

---

### `drift-status`

Read-only snapshot of the current session. Writes nothing, changes nothing, calls no APIs.

**Usage:**

```
drift-status
```

**What it shows:**

```
Drift session active

  Started:  2025-02-14T16:47:00Z
  Branch:   feature/auth
  SHA:      a1b2c3d → f4e5d6c
  Ticket:   BAS-10
  Plan:     implement JWT authentication

Changes since session start:
  Commits:       8
  Files changed: 14

  backend/auth.py          | 87 +++++++++
  backend/middleware.py    | 34 ++++
  backend/config.py        |  3 +-
  backend/router.py        | 15 ++-

Changed files:
  backend/auth.py
  backend/middleware.py
  backend/config.py
  backend/router.py

Uncommitted: yes - 2 files

Recent sessions (Claude Code):
  1. 2025-02-13 — BAS-09 — Scaffolded dashboard with full folder structure
  2. 2025-02-12 — BAS-08 — Added user model and async database layer

Recent sessions (CLI):
  1. 2025-02-11 — BAS-07 — Initial FastAPI setup with uvicorn
```

**Session history sources vary by tool:**

| Tool | What it reads |
|------|--------------|
| Claude Code | `sessions.claude.json` + `sessions.json` (2 sources) |
| Cursor | `sessions.cursor.json` + `sessions.claude.json` + `sessions.json` (3 sources) |
| Codex | `sessions.codex.json` + `sessions.claude.json` + `sessions.cursor.json` + `sessions.json` (4 sources) |

Shows the last 5 entries from each source. Suggests running `drift-sync` and checking the last report file if one exists.

---

### `drift-sync`

Deep analysis of everything that changed since the session started. This is the core of Drift.

**Usage:**

```
drift-sync
drift-sync --no-push
drift-sync --no-jira
```

**The 8 phases:**

**Phase 1 — Validate session.** Reads `.drift/config.json`, gets the active session (last element of the sessions array), extracts `startSha`, `ticket`, and `plannedWork`.

**Phase 1b — Fetch the ticket from Jira.** Calls the Jira API and does a deep AC extraction — looks for acceptance criteria in three places (first match wins): (1) a custom field containing "acceptance" in its name, (2) a checklist or bullet list in the ticket description, (3) subtask summaries. This AC becomes the authoritative checklist for the analysis. *Skipped if `--no-push` or `--no-jira` flag is set.*

**Phase 2 — Gather git data.** Full diff (not truncated), all commit messages, list of added/modified/deleted files, staged and unstaged uncommitted changes. Everything since the starting SHA.

**Phase 3 — Read every changed file in full.** Reads the complete current content of every changed file (not just diff chunks) so the analysis understands each file in context. Also reads related unchanged files for architecture context. Thresholds:

| Condition | Behavior |
|-----------|----------|
| Binary file | Skipped, noted as "binary file, not analyzed" |
| File over 1000 lines | Content skipped, noted as "too large for inline analysis" |
| More than 100 changed files | Reads top 20 by lines changed, notes the rest |
| File with more than 200 lines changed | Also examines the per-file diff individually |

**Phase 4 — Produce the structured analysis.** Seven sub-sections:

- **What happened** — 3-5 sentences with specific file paths, function names, and approaches
- **Technical decisions** — each rated SOUND (clearly right), ACCEPTABLE (reasonable, alternatives exist), or QUESTIONABLE (may cause problems)
- **Drift from plan** — scope additions (built but not in AC), scope reductions (AC items not done), approach changes. Only included if `plannedWork` was provided.
- **Architecture impact** — new patterns, dependency changes, API surface changes
- **Code review** — every changed file reviewed with signal labels:
  - **CLEAN** — good practice, no action needed
  - **CONCERN** — worth watching, may become a problem
  - **ISSUE** — should be fixed, creates real risk

  Organized across 6 categories: approach validation, hacky patterns, best practices, architecture soundness, security concerns, potential issues. Each finding includes file path, line number, and recommendation. Ends with a 1-3 sentence overall assessment.
- **Next session items** — remaining AC items, TODOs/FIXMEs/HACKs found in code, tests needed, edge cases
- **Ticket comment** — the structured text posted to Jira

**Phase 5 — Write CONTEXT.md.** Updates the rolling context document at the repo root. The current "Latest Session" content (if CONTEXT.md already exists) gets demoted to "Previous Sessions" and the new analysis takes its place. Up to 4 previous sessions are kept — older ones roll off (but remain in git history). Code review section includes only CONCERN and ISSUE findings — CLEAN is omitted to keep it concise across sessions. If `CONTEXT.claude.md` exists from an older version, it's renamed to `CONTEXT.md` first.

**Phase 6 — Save session history.** Prepends a structured JSON entry to `.drift/sessions.claude.json` (or `sessions.cursor.json` / `sessions.codex.json`). Newest entries first. Capped at 50 — oldest trimmed when exceeded.

**Phase 7 — Post to Jira.** Posts a structured comment to the ticket as Atlassian Document Format (ADF). *Skipped if `--no-push` or `--no-jira` flag is set.*

**Phase 8 — Write the report.** Displays the full analysis in the terminal and writes it to `<ticket>-report.md` (lowercase, e.g., `bas-10-report.md`) or `session-report.md` if no ticket. Overwritten on each sync. Previous ticket report files are not deleted.

**Session persistence:** `drift-sync` does **not** clear the active session. You can keep coding and run it again — the old "Latest Session" in CONTEXT.md gets demoted to "Previous Sessions" and the new analysis takes its place. The session only ends when you run `drift-start` for a new ticket.

**Flags:**

| Flag | Behavior |
|------|----------|
| `--no-push` | Skip all Jira integration — no credential resolution, no ticket fetch, no comment posted. Analysis uses `plannedWork` from config instead of ticket AC. |
| `--no-jira` | Identical to `--no-push`. Both flags exist for readability. |

There is currently no way to fetch AC from Jira but skip posting the comment. Both flags disable all Jira integration.

**Report output example:**

```
Drift sync complete (Claude deep analysis)

What happened:
  Implemented JWT authentication using python-jose with HS256 signing.
  Added login and register endpoints to router.py, middleware for token
  validation in middleware.py, and updated config.py with JWT settings.

Decisions:
  - SOUND: HS256 over RS256 — simpler for single-service architecture
  - ACCEPTABLE: Token in Authorization header vs httpOnly cookie

Drift from plan:
  - Added: password reset endpoint (not in AC — scope creep)
  - Missing: token refresh endpoint (AC item #4, not started)

Architecture impact:
  - New middleware layer in middleware.py for authenticated routes
  - python-jose dependency added to requirements.txt

Code review:
  3 clean | 1 concern | 1 issue
  - CONCERN: JWT secret has hardcoded fallback (config.py:13) — remove before deployment
  - ISSUE: Login endpoint has no rate limiting (router.py:41) — brute force risk
  Overall: Solid auth foundation. Rate limiting is the critical fix before merge.

For next session:
  - Implement token refresh endpoint (BAS-10 AC item #4)
  - Add rate limiting to /login
  - Move password reset to its own ticket

---
Files changed: 14 | Commits: 8 | Files analyzed: 12
CONTEXT.md updated
Jira: comment posted to BAS-10
```

---

## Output Files

### `CONTEXT.md`

Lives at the repo root. Committed to git, shared with the team. This is the file that answers "what happened in this repo."

Updated every `drift-sync`. Contains the full analysis from the latest session and summaries of up to 4 previous sessions. Code review section includes only CONCERN and ISSUE findings (CLEAN omitted for brevity).

```markdown
# Project Context

> This file is auto-generated by drift (Claude Code skill).
> It provides deep analysis of recent changes.
> Last updated: 2025-02-16T14:00:00Z
> Analysis method: claude-full-context (no truncation, multi-step file analysis)

# Latest Session

## Session: 2025-02-16 (BAS-12)

**What happened:**
Added dashboard filtering and search. Created FilterBar component
in components/FilterBar.tsx, added search API endpoint in api/search.py,
and wired up client-side filtering for small datasets.

**Decisions made:**
- SOUND: Client-side filtering for datasets under 100 rows, server-side
  for search — avoids unnecessary API calls for small lists
- ACCEPTABLE: Used query params for search instead of POST body

**Drift from plan:**
- Missing: export to CSV (AC item #3, not started)

**Architecture impact:**
- New FilterBar component added to shared components
- Search endpoint added to api/search.py

**Code review:**
- CONCERN: Search endpoint has no pagination (api/search.py:45) —
  will cause performance issues as dataset grows
Overall: Clean implementation. Add pagination before dataset scales.

**For next session:**
- Implement CSV export (AC item #3)
- Add pagination to search endpoint

# Previous Sessions

---

## Session: 2025-02-15 (BAS-11)

**What happened:**
Implemented user auth with JWT tokens. Added login/register endpoints,
token validation middleware, and JWT configuration settings.

**Decisions made:**
- SOUND: HS256 signing for single-service architecture
- ACCEPTABLE: Token in Authorization header

**For next session:**
- Token refresh endpoint
- Rate limiting on /login

---

## Session: 2025-02-14 (BAS-10)

**What happened:**
Scaffolded FastAPI project structure with async SQLAlchemy, Pydantic v2
response wrappers, and base repository pattern.

**For next session:**
- All endpoints still stubs — implement real logic
```

### `<ticket>-report.md`

Written to the repo root. Filename uses the ticket ID in lowercase (e.g., `bas-10-report.md`). If no ticket, `session-report.md`. Overwritten each time you sync the same ticket. Report files from previous tickets are not deleted — `bas-10-report.md` and `bas-11-report.md` can coexist.

Contains the same content as the terminal output from Phase 8. Code review shows CONCERN and ISSUE findings individually, CLEAN as a count only (e.g., "3 clean | 1 concern | 1 issue").

### `.drift/sessions.*.json`

Structured JSON history in `.drift/` (gitignored). Each tool writes to its own file: `sessions.claude.json`, `sessions.cursor.json`, `sessions.codex.json`. Newest entries at the front. Capped at 50 entries.

Each entry contains:

```json
{
  "date": "2025-02-14T16:47:00.000Z",
  "ticket": "BAS-10",
  "startSha": "a1b2c3d4...",
  "endSha": "f4e5d6c7...",
  "summary": "Implemented JWT authentication using python-jose...",
  "decisions": ["HS256 over RS256 — SOUND: simpler key management"],
  "drift": ["Added: password reset (not in AC)", "Missing: token refresh (AC #4)"],
  "architectureImpact": ["New middleware.py for JWT validation"],
  "forNextSession": ["Implement token refresh (AC #4)"],
  "ticketComment": "Drift Session Summary (Claude analysis):\n\nProgress: ...",
  "codeReview": {
    "findings": [
      {
        "signal": "CONCERN",
        "category": "security",
        "file": "backend/config.py",
        "finding": "JWT secret has hardcoded fallback",
        "recommendation": "Remove fallback, raise on missing env var"
      }
    ],
    "overall": "Solid auth foundation. Rate limiting is the critical gap.",
    "counts": { "clean": 3, "concern": 1, "issue": 1 }
  },
  "analysisMethod": "claude-full-context",
  "filesAnalyzed": 12,
  "totalFilesChanged": 14,
  "jiraCommentPosted": true
}
```

---

## Jira Integration

### What it fetches

**On `drift-start`** (only if you didn't provide a description and Jira is configured):
Fetches the ticket's summary and description to use as the session's `plannedWork`.

**On `drift-sync`** (Phase 1b):
Deep AC extraction — looks for acceptance criteria in three places (first match wins):
1. A custom field containing "acceptance" in its name
2. A checklist or bullet list in the ticket description body
3. Subtask summaries

This AC becomes the authoritative checklist for measuring progress and drift.

### What it posts

On `drift-sync` (Phase 7), posts a structured comment to the ticket:

```
Drift Session Summary (Claude analysis):

Progress: 60% of this ticket's acceptance criteria (3 of 5 items)

What was done:
JWT auth implemented with login/register endpoints and token
validation middleware.

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

Progress is concrete: completed AC items divided by total AC items.

### Setting up credentials

You need: Jira URL, email, and API token (from https://id.atlassian.com/manage-profile/security/api-tokens).

Drift checks for credentials in this order (first complete set wins):

| Priority | Source | Notes |
|----------|--------|-------|
| 1 | Environment variables | `JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN` |
| 2 | `.env` in project root | Standard `KEY=value` format |
| 3 | `.drift/.env` | Drift-specific, keeps creds out of your main `.env` |
| 4 | `~/.config/drift-nodejs/config.json` | Global config (shared with drift CLI if you use it) |
| 5 | `.drift/jira.json` | Saved when you enter creds interactively |
| 6 | Interactive prompt | Drift asks during `drift-start` or `drift-sync` |

On first run, if no credentials are found, Drift offers to collect them interactively and saves them to `.drift/jira.json` (gitignored — never committed).

### Running without Jira

Drift works without Jira. Use `--no-jira` or `--no-push` on sync, or don't configure credentials. The analysis uses your `drift-start` description instead of ticket AC. You still get CONTEXT.md, the report file, the code review, and session history — just no Jira comment and no AC-based progress tracking.

---

## Scoping Rules

Drift is strict about what counts as "drift." Without strict rules, every analysis would be full of invented requirements.

1. **Only the written AC matters.** Progress and drift are measured against the ticket's acceptance criteria as literally written. If the AC says "add login endpoint" and it exists, that item is done — even without tests or error handling.
2. **If it's not in the AC, it's not missing.** Drift never invents requirements. If the AC doesn't mention tests, missing tests are not a scope reduction.
3. **Scope creep is flagged, not judged.** Things built outside the AC are listed as scope additions — not marked wrong, just surfaced so you decide consciously.
4. **Future ticket work is out of scope.** Work for other tickets goes under "future work (out of scope)" — never mixed into this session's progress.
5. **Progress is concrete.** Completed AC items divided by total AC items. 3 of 5 = 60%.

**When AC is vague or missing:** If Drift can't find clear acceptance criteria (no AC field, no description checklist, no subtasks), it falls back to the `plannedWork` text from `drift-start`. The analysis still runs but progress tracking is less precise.

---

## File Structure

```
your-repo/
├── CONTEXT.md                   # Written by drift-sync (committed, shared)
├── bas-10-report.md             # Written by drift-sync (committed)
│
└── .drift/                      # Auto-gitignored on first drift-start
    ├── config.json              # Active session state (max 20 sessions)
    ├── sessions.json            # Drift CLI session history
    ├── sessions.claude.json     # Claude Code analysis history (max 50)
    ├── sessions.cursor.json     # Cursor analysis history (max 50)
    ├── sessions.codex.json      # Codex analysis history (max 50)
    ├── jira.json                # Jira credentials (if entered interactively)
    └── .env                     # Optional drift-specific env vars
```

`CONTEXT.md` and report files are committed to git — they're the artifacts your team reads. Everything in `.drift/` is local-only and never committed.

---

## Cross-Tool Compatibility

All three tool versions share `.drift/`, `CONTEXT.md`, and Jira credentials. You can switch between tools on the same project.

### Shared files

| File | Used by |
|------|---------|
| `.drift/config.json` | All tools (active session state) |
| `.drift/jira.json` | All tools (Jira credentials) |
| `.drift/.env` | All tools (env vars) |
| `CONTEXT.md` | Overwritten by whichever tool syncs last |
| `<ticket>-report.md` | Overwritten by whichever tool syncs last |

### What differs

| | Claude Code | Cursor | Codex |
|---|---|---|---|
| **Install location** | `~/.claude/skills/` | `~/.cursor/rules/` | `~/.agents/skills/` |
| **File format** | `SKILL.md` in folders | `.mdc` flat files | `SKILL.md` in folders |
| **Invocation** | `/drift-start BAS-10` | `drift-start BAS-10` | `drift-start BAS-10` |
| **Session history file** | `sessions.claude.json` | `sessions.cursor.json` | `sessions.codex.json` |
| **drift-status reads** | Own + CLI (2 sources) | Own + Claude + CLI (3 sources) | All four files (4 sources) |
| **Analysis label** | `claude-full-context` | `cursor-full-context` | `codex-full-context` |

Behavior is identical across all three. Only invocation, file format, and labels differ.

---

## Edge Cases

| Scenario | What happens |
|----------|----------|
| No changes since session start | Reports "No changes" and stops — no files written |
| Binary files in the diff | Skipped, noted as "binary file, not analyzed" |
| File over 1000 lines | Content skipped, noted as "too large for inline analysis" |
| More than 100 changed files | Reads top 20 by lines changed, notes the rest |
| File with 200+ lines changed | Also examines the per-file diff individually |
| No planned work and no Jira AC | "Drift from plan" section omitted entirely |
| `startSha` missing after rebase | Falls back to `git merge-base`, fails gracefully if that also fails |
| Jira API returns an error | Warns you, continues analysis without ticket context |
| Jira comment fails to post | Warns you, still writes CONTEXT.md and report |
| Old `CONTEXT.claude.md` exists | Renamed to `CONTEXT.md` automatically, history preserved |
| `drift-start` with active session | Shows current session, asks to replace (old is removed, not archived) |
| Re-running `drift-sync` | Old Latest Session demoted to Previous Sessions, new analysis takes its place |

---

## Design Decisions

**Why three commands instead of one?** `drift-start` needs to be fast (no overhead at session start). `drift-status` needs to be safe (read-only, no risk). `drift-sync` is expensive (reads many files, calls Jira). Combining them would mean every status check triggers a Jira call.

**Why cap sessions at 20 in config.json?** Config is the active session store, not the archive. Full analysis history lives in `sessions.*.json` (50 entries). Keeping config small prevents it from growing after months of use.

**Why does CONTEXT.md live in the repo root?** It's meant to be committed and shared. When a teammate clones the repo or a new AI conversation starts, CONTEXT.md should be the first thing they read. Hiding it in `.drift/` would defeat that purpose.

**Why strict AC scoping?** Evaluating against "what a complete implementation would look like" produces noise. Drift's job is to compare what you said you'd do against what you did. Strict scoping means findings are grounded in what was agreed upon, not assumptions.

**Why does drift-status write nothing?** If it had side effects, you'd hesitate to run it. Making it provably read-only means it can be run at any time without risk.

**Why include CLEAN findings in code review?** Purely negative reviews are harder to trust. Acknowledging good practices confirms the analysis looked at the full picture and rewards patterns worth repeating.

---

## Full Documentation

For the complete technical reference — every phase of `drift-sync` in full detail, the complete data model, all scoping rules with rationale, and the design decisions behind each choice — see [DRIFT_DOCS_DETAILED.md](docs/DRIFT_DOCS_DETAILED.md).

---

## Acknowledgments

This project was inspired by [Drift](https://github.com/Mersall/drift) by [@Mersall](https://github.com/Mersall). Thank you for the original idea and foundation.

---

## License

This project is licensed under the [MIT License](LICENSE).
