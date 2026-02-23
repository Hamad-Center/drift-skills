# Drift Skills for Codex

Track how your code drifts from plan. These skills run inside Codex as agent skills, connecting your git changes to Jira tickets with deep AI-powered analysis.

## What Is Drift

Drift is a session tracking system for AI-assisted development. It solves five problems that come with high-velocity AI coding:

1. **Invisible scope creep** — you build features nobody asked for and don't notice until code review
2. **Forgotten acceptance criteria** — the ticket says one thing, the code says another
3. **Context loss between sessions** — you close the laptop and lose everything that wasn't committed
4. **Hacky AI shortcuts** — the AI picks the path of least resistance, not the right architecture
5. **Untrackable progress** — commits say "fix" and "wip", nobody knows the real status

Drift fixes all five with three commands and zero changes to how you write code.

## Skills

### `drift-start` — Start a session

Records the current git SHA and session metadata. This is your starting point.

```
drift-start BAS-10 implement user auth
drift-start PROJ-123
```

**What it does:**
- Requires a Jira ticket ID (asks if not provided)
- Records git SHA, branch, timestamp, ticket, and planned work
- Auto-initializes `.drift/` directory and `.gitignore` on first run
- If no planned work text is given but Jira is configured, fetches the ticket's summary+description as planned work
- Caps session history at 20 entries to prevent unbounded growth

**Files created/updated:**
- `.drift/config.json` — active session state
- `.drift/sessions.json` — session history

### `drift-sync` — Analyze changes

Deep multi-step analysis of everything that changed since the session started. This is the core skill.

```
drift-sync
drift-sync --no-push    # skip all Jira integration (no fetch, no post)
drift-sync --no-jira    # same as --no-push
```

**What it does:**
1. Validates git repo and active session
2. Fetches Jira ticket context (including acceptance criteria)
3. Reads full diffs and file contents (no truncation — this is the key advantage over the CLI)
4. Produces structured analysis scoped to the ticket's AC:
   - What was built (concrete file paths, not vague summaries)
   - Technical decisions rated SOUND / ACCEPTABLE / QUESTIONABLE
   - Drift from plan (scope additions, scope reductions, approach changes)
   - Architecture impact (new patterns, dependencies, API changes)
   - Code review with CLEAN / CONCERN / ISSUE labels across 6 categories
   - Next session items (remaining AC, TODOs, tests needed)
5. Writes `CONTEXT.md` with full analysis (keeps up to 4 previous sessions)
6. Writes `.drift/sessions.codex.json` history (max 50 entries)
7. Posts a structured comment to the Jira ticket
8. Writes `<ticket>-report.md` (e.g., `bas-10-report.md`)

**Jira comment format:**
```
Drift Session Summary (Codex analysis):

Progress: ~80% of this ticket's acceptance criteria

What was done:
Dashboard scaffolded with full folder structure and all dependencies.

Drift from plan:
None

Code quality:
No concerns

Next steps:
- Verify dev server runs without errors
```

**Key design decision:** Progress, drift, and next steps are evaluated ONLY against what is explicitly written in the ticket's acceptance criteria. The skill never assumes or infers requirements that aren't written.

**Files created/updated:**
- `CONTEXT.md` — full analysis in the repo root
- `<ticket>-report.md` — session report (e.g., `bas-10-report.md`)
- `.drift/sessions.codex.json` — structured session history

### `drift-status` — Quick status check

Read-only snapshot of the active session. No file writes.

```
drift-status
```

**What it shows:**
- Session details (started, branch, SHA, ticket, plan)
- Change stats (commits, files changed, diff stat)
- Uncommitted changes
- Recent session history from Codex, Claude, Cursor, and CLI logs
- Next steps reminder with report file location

## Code Review System

Every `drift-sync` includes a code review of all changed files.

**Signal labels:**
- **CLEAN** — good practice, no action needed
- **CONCERN** — worth watching, may become a problem
- **ISSUE** — should be fixed, creates real risk

**Categories:**
1. Approach validation
2. Hacky patterns
3. Best practices
4. Architecture soundness
5. Security concerns
6. Potential issues

Each finding includes a file path, line number, and recommendation. The review ends with a 1–3 sentence overall assessment.

## Jira Integration

The skills connect to Jira in two directions:

**Inbound (fetch):** On `drift-start` and `drift-sync`, fetches the ticket's summary, description, acceptance criteria, and status. The AC becomes the authoritative checklist for measuring progress and drift.

**Outbound (post):** On `drift-sync`, posts a structured comment to the ticket with progress, what was built, code quality, and next steps.

### Credentials

You need three values:

| Variable | Description |
|----------|-------------|
| `JIRA_URL` | e.g., `https://yourteam.atlassian.net` |
| `JIRA_EMAIL` | Your Atlassian account email |
| `JIRA_TOKEN` | API token from https://id.atlassian.com/manage-profile/security/api-tokens |

### Where to configure

Checked in this order (first complete set wins):

| Priority | Source | How to set |
|---|---|---|
| 1 | Shell env vars | `export JIRA_URL=...`, `JIRA_EMAIL=...`, `JIRA_TOKEN=...` |
| 2 | `.env` in project root | Standard key=value format |
| 3 | `.drift/.env` | Drift-specific, keeps creds out of your main `.env` |
| 4 | `~/.config/drift-nodejs/config.json` | Written by `drift config` CLI command |
| 5 | `.drift/jira.json` | Written when you provide creds interactively |
| 6 | Interactive prompt | The skill asks during `drift-start` or `drift-sync` |

The `.env` parser handles `export` prefixes, quoted values, inline comments, and trailing whitespace.

## Output Files

| File | Location | Lifecycle | What it contains |
|---|---|---|---|
| `CONTEXT.md` | repo root | Updated every sync, keeps 4 previous sessions | Full analysis: decisions, drift, architecture, code review, next steps |
| `<ticket>-report.md` | repo root | Overwritten every sync | Human-readable session report, ready for PR descriptions |
| `sessions.codex.json` | `.drift/` | Prepended every sync, max 50 entries | Machine-readable JSON session history |

`CONTEXT.md` and the report file are meant to be committed. `.drift/` is gitignored.

## File Structure

```
<repo root>/
├── CONTEXT.md                   # Latest analysis (committed, shared)
├── bas-10-report.md             # Latest report (committed, shared)
│
└── .drift/                      # Auto-gitignored, never committed
    ├── config.json              # Session state (max 20 sessions)
    ├── sessions.json            # CLI session history
    ├── sessions.claude.json     # Claude Code session history (max 50)
    ├── sessions.cursor.json     # Cursor session history (max 50)
    ├── sessions.codex.json      # Codex session history (max 50)
    ├── jira.json                # Jira credentials (if saved interactively)
    └── .env                     # Optional drift-specific env vars
```

## Installation

These skills are installed globally at `~/.agents/skills/`:

```
~/.agents/skills/
  drift-start/SKILL.md
  drift-sync/SKILL.md
  drift-status/SKILL.md
  README.md
```

They work in any git repository. Invoke via `/skills` or `$` mentions in the Codex CLI:

```
drift-start BAS-10 implement auth
drift-status
drift-sync
drift-sync --no-jira
```

## Edge Cases

| Scenario | Behavior |
|---|---|
| No changes since start | Reports "No changes" and stops without writing files |
| Binary files | Skipped, noted as "binary file, not analyzed" |
| Files over 1000 lines | Skipped, noted as "too large for inline analysis" |
| More than 100 changed files | Reads top 20 by lines changed, notes the rest |
| No `plannedWork` | "Drift from plan" section omitted entirely |
| Missing `startSha` (after rebase) | Falls back to `git merge-base`, fails gracefully |
| Jira fetch failure | Warns user, continues without ticket context |
| Jira comment failure | Warns user, still writes CONTEXT.md and report |
| Old `CONTEXT.claude.md` exists | Renamed to `CONTEXT.md` via `mv`, history preserved |

## Compatibility with Claude Code and Cursor

Drift is also available as Claude Code skills (`~/.claude/skills/`) and Cursor rules (`~/.cursor/rules/`). All three versions share the same `.drift/` directory and `CONTEXT.md` output.

| File | Shared across tools? |
|---|---|
| `.drift/config.json` | Yes — all tools read/write session state |
| `.drift/jira.json` | Yes — credentials work everywhere |
| `CONTEXT.md` | Yes — overwritten by whichever tool syncs last |
| `<ticket>-report.md` | Yes — overwritten by whichever tool syncs last |
| `sessions.claude.json` | Claude Code only |
| `sessions.cursor.json` | Cursor only |
| `sessions.codex.json` | Codex only |

`drift-status` shows session history from all four sources (Claude, Cursor, Codex, CLI).

## Typical Workflow

```
drift-start BAS-10 scaffold dashboard project
  ... do work ...
drift-status                    # check progress (read-only)
  ... do more work ...
drift-sync                      # analyze + post to Jira
drift-start BAS-11 next task    # start new session
```
