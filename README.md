# Drift Skills

> AI-powered session tracking that connects your git changes to Jira tickets — available for Claude Code, Cursor, and Codex.

---

## The Problem

AI-assisted development is fast. Dangerously fast. You open a Jira ticket, start coding with an AI assistant, and two hours later you have 40 files changed, 12 commits, and a vague memory of what you were supposed to build.

Five things go wrong repeatedly:

1. **Scope creep goes unnoticed** — you build features nobody asked for because the AI kept suggesting "while we're here" improvements
2. **Acceptance criteria get forgotten** — the ticket says one thing, your code says another, nobody knows until QA
3. **Context evaporates between sessions** — you close the laptop and lose all the decisions, trade-offs, and next steps that weren't committed
4. **Hacky shortcuts slip through** — the AI picks the path of least resistance, not the right architecture
5. **Progress is impossible to track** — commits say "fix" and "wip", the PM has no idea what's actually done

Drift fixes all five.

---

## What Is Drift

Drift is a set of AI assistant rules that run inside your existing tools. Not a separate app, not a CI plugin, not a workflow change. Three commands:

```
drift-start   →  snapshot your starting point + Jira ticket
drift-status  →  read-only progress check
drift-sync    →  deep analysis + code review + Jira comment
```

---

## How It Works

### `drift-start` — Anchor the session

Before you write a single line of code:

```
drift-start BAS-10 implement JWT authentication
```

Drift records your git SHA, branch, timestamp, ticket, and plan. If Jira is configured, it fetches the ticket's acceptance criteria so there's a real checklist to measure against — not just your memory of what the ticket said.

### `drift-status` — Check without side effects

At any point during your session:

```
drift-status
```

Shows commits since start, files changed, diff stats, uncommitted work, and recent session history. Writes nothing. Changes nothing. Pure read-only mirror.

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
  Uncommitted:   yes - 2 files
```

### `drift-sync` — Full analysis before you ship

This is the core. Run it before opening a PR:

```
drift-sync
```

Drift reads the **full diff with no truncation** — every changed file in its entirety, every commit, related unchanged files for context. Then produces a structured analysis:

- **What was built** — concrete file paths and approaches, not vague summaries
- **Technical decisions** — what was chosen, what was skipped, rated SOUND / ACCEPTABLE / QUESTIONABLE
- **Drift from plan** — scope additions, scope reductions, approach changes vs the ticket's AC
- **Architecture impact** — new patterns, dependencies, API surface changes
- **Code review** — every changed file reviewed with CLEAN / CONCERN / ISSUE labels across 6 categories
- **Next session items** — remaining AC, TODOs found in code, tests needed, known edge cases

---

## What You Get

After every `drift-sync`, four things are produced:

| Output | What it is |
|---|---|
| **`CONTEXT.md`** | Living context document in the repo root. Updated every sync, keeps up to 4 previous sessions. Decisions, drift, code review findings, and next steps — survives laptop closes, team handoffs, and AI memory resets |
| **`<ticket>-report.md`** | Human-readable session report (e.g., `bas-10-report.md`). Overwritten every sync. Ready to paste into a PR description or share async |
| **Jira comment** | Auto-posted to the ticket: progress %, what was built, what drifted, code quality summary, and next steps. Your PM gets an update without you writing a word |
| **`sessions.*.json`** | Structured JSON history in `.drift/`. Max 50 entries. Machine-readable for tooling, auditing, or dashboards |

---

## Code Review System

Every `drift-sync` reviews all changed files with three signal labels:

| Signal | Meaning | Action |
|---|---|---|
| **CLEAN** | Good practice | None needed — acknowledges what's done well |
| **CONCERN** | Worth watching | May become a problem as codebase grows |
| **ISSUE** | Should be fixed | Creates real risk in security, correctness, or maintainability |

Findings are organized across six categories:

1. **Approach validation** — right solution for the problem?
2. **Hacky patterns** — magic numbers, tight coupling, copy-paste
3. **Best practices** — framework idioms, naming, error handling
4. **Architecture soundness** — right abstraction level? over/under-engineered?
5. **Security concerns** — hardcoded secrets, auth gaps, injection risks
6. **Potential issues** — race conditions, resource leaks, edge cases

Each finding includes a file path, line number, and recommendation. Every review ends with an overall assessment.

---

## Scoping Rules

Drift is strict about what counts as "drift" and what doesn't:

- **Only the written AC matters** — if the acceptance criteria don't mention tests, missing tests are not drift
- **Scope creep is flagged, not judged** — things you built that weren't in the AC are surfaced so you decide consciously
- **Future ticket work is out of scope** — work belonging to other tickets is labeled separately, never mixed in
- **No inferred requirements** — "obviously you need error handling" is not a scoping rule; only what's literally written counts
- **Progress % is concrete** — completed AC items / total AC items, nothing implied

---

## Jira Integration

Drift connects to Jira in two directions:

**Inbound (fetch):**
- On `drift-start` — fetches ticket summary, description, and acceptance criteria
- On `drift-sync` — fetches again to use AC as the authoritative checklist for analysis

**Outbound (post):**
- On `drift-sync` — posts a structured comment to the ticket:

```
Drift Session Summary:

Progress: ~70% of this ticket's acceptance criteria

What was done:
JWT auth implemented with login/register endpoints and middleware.

Drift from plan:
Added password reset endpoint not in AC — recommend separate ticket.

Code quality:
1 issue: login endpoint has no rate limiting (router.py:41)

Next steps:
- Implement token refresh (AC item #4)
- Add rate limiting to /login
```

### Credentials

| Variable | Description |
|---|---|
| `JIRA_URL` | e.g., `https://yourteam.atlassian.net` |
| `JIRA_EMAIL` | Your Atlassian account email |
| `JIRA_TOKEN` | API token from https://id.atlassian.com/manage-profile/security/api-tokens |

Checked in this order (first complete set wins):

| Priority | Source |
|---|---|
| 1 | Shell env vars: `JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN` |
| 2 | `.env` in project root |
| 3 | `.drift/.env` (drift-specific, keeps creds separate) |
| 4 | `~/.config/drift-nodejs/config.json` (global CLI config) |
| 5 | `.drift/jira.json` (saved when you provide creds interactively) |
| 6 | Interactive prompt (asks during `drift-start` or `drift-sync`) |

The `.env` parser handles `export` prefixes, quoted values, inline comments, and trailing whitespace.

---

## Flags

```
drift-sync --no-push    # run full analysis, skip posting the Jira comment
drift-sync --no-jira    # skip all Jira integration (no fetch, no post)
```

---

## Quick Install

### Claude Code

```bash
cp -r claude-code/* ~/.claude/skills/
```

Invoke with slash commands:
```
/drift-start BAS-10 implement user auth
/drift-status
/drift-sync
```

### Cursor

```bash
cp cursor/* ~/.cursor/rules/
```

Invoke by typing in agent mode chat:
```
drift-start BAS-10 implement user auth
drift-status
drift-sync
```

### Codex

```bash
cp -r codex/* ~/.agents/skills/
```

Invoke via `/skills` or `$` mentions:
```
drift-start BAS-10 implement user auth
drift-status
drift-sync
```

---

## Tool Compatibility

All three versions share the same `.drift/` directory and `CONTEXT.md`. You can switch between tools freely on the same project.

| Shared across all tools | Separate per tool |
|---|---|
| `.drift/config.json` (session state) | `sessions.claude.json` (Claude Code history) |
| `.drift/jira.json` (credentials) | `sessions.cursor.json` (Cursor history) |
| `CONTEXT.md` (latest analysis) | `sessions.codex.json` (Codex history) |
| `<ticket>-report.md` (latest report) | |

`drift-status` in any tool shows session history from all four sources (Claude Code, Cursor, Codex, and CLI).

---

## File Structure

```
<your repo>/
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

---

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

---

## Repo Structure

```
drift-skills/
├── README.md                           # This file
├── claude-code/                        # Claude Code skills
│   ├── README.md
│   ├── drift-start/SKILL.md
│   ├── drift-sync/SKILL.md
│   └── drift-status/SKILL.md
├── cursor/                             # Cursor rules
│   ├── README.md
│   ├── drift-start.mdc
│   ├── drift-sync.mdc
│   └── drift-status.mdc
├── codex/                              # Codex skills
│   ├── README.md
│   ├── drift-start/SKILL.md
│   ├── drift-sync/SKILL.md
│   └── drift-status/SKILL.md
└── docs/
    └── DRIFT_DOCS_DETAILED.md          # Full technical reference
```

---

## Documentation

For the full technical reference — every phase of `drift-sync`, the complete data model, all scoping rules, every edge case, and the design decisions behind each choice — see [DRIFT_DOCS_DETAILED.md](docs/DRIFT_DOCS_DETAILED.md).

---

## Typical Workflow

```
drift-start BAS-10 scaffold dashboard project
  ... code with AI ...
drift-status                    # check progress (read-only)
  ... keep coding ...
drift-sync                      # full analysis + Jira update
drift-start BAS-11 next task    # new session
```
