# Drift Skills

> AI-powered session tracking that connects your git changes to Jira tickets — available for Claude Code, Cursor, and Codex.

---

## The Problem

AI-assisted development is fast. Dangerously fast. You open a Jira ticket, start coding with an AI assistant, and two hours later you have 40 files changed, 12 commits, and a vague memory of what you were supposed to build.

Five things go wrong repeatedly:

1. **Scope creep goes unnoticed** — you build features nobody asked for because the AI kept suggesting "while we're here" improvements. Drift catches this by comparing what you built against the ticket's written acceptance criteria and flagging anything extra as a scope addition.

2. **Acceptance criteria get forgotten** — the ticket says one thing, your code says another, nobody knows until QA. Drift fetches the AC from Jira and uses it as the checklist for every analysis, so it's always visible.

3. **Context evaporates between sessions** — you close the laptop and lose all the decisions, trade-offs, and next steps that weren't committed. Drift writes `CONTEXT.md` to the repo — a rolling document that persists everything across sessions, tool switches, and AI memory resets.

4. **Hacky shortcuts slip through** — the AI picks the path of least resistance, not the right architecture. Drift runs a code review on every sync, checking for security gaps, hacky patterns, and architecture concerns.

5. **Progress is impossible to track** — commits say "fix" and "wip", the PM has no idea what's actually done. Drift posts a structured comment to the Jira ticket with concrete progress (e.g., "3 of 5 AC items done"), what was built, and what's next.

---

## What Is Drift

Drift is a set of instruction files that your AI assistant follows. You copy them into your tool's skills/rules directory, and they become available as commands. The AI assistant does all the work — reading files, running git commands, calling the Jira API, writing the analysis — by following the instructions in the skill files.

There's nothing to install, no dependencies, no runtime. Just markdown files that tell your AI assistant how to track sessions.

Three commands:

| Command | What it does | How you invoke it |
|---|---|---|
| `drift-start` | Start tracking a session against a Jira ticket | Claude Code: `/drift-start BAS-10` · Cursor: `drift-start BAS-10` · Codex: `drift-start BAS-10` |
| `drift-status` | See what changed so far (read-only, writes nothing) | Claude Code: `/drift-status` · Cursor: `drift-status` · Codex: `drift-status` |
| `drift-sync` | Analyze everything, review the code, update Jira | Claude Code: `/drift-sync` · Cursor: `drift-sync` · Codex: `drift-sync` |

In Claude Code these are slash commands (`/drift-start`). In Cursor you type the name in agent mode chat (`drift-start BAS-10`). In Codex you use `/skills` or `$` mentions. The behavior is identical across all three.

---

## How It Works

### 1. Start a session with `drift-start`

Before you write any code, tell Drift what ticket you're working on:

```
drift-start BAS-10
```

That's it. Just the ticket ID. The ticket ID is required — Drift will ask for one if you don't provide it.

**What happens behind the scenes:**
- Drift saves your current git SHA and timestamp as the starting snapshot (the branch is displayed but not stored — it's read from git at display time)
- If Jira is configured, it fetches the ticket's **summary and description** to use as context for the session
- This context is stored as the session's `plannedWork` — the baseline for measuring drift later

You can optionally add your own description if you want to be more specific than what's in Jira:

```
drift-start BAS-10 implement JWT authentication
```

If you provide a description, Drift uses that as the `plannedWork` instead of fetching from Jira. If you provide just the ticket ID and Jira is connected, it pulls the ticket details automatically. If Jira isn't configured, just add a short description of what you plan to do.

**Starting a new session when one is already active?** Drift shows you the current session and asks if you want to replace it. If you say yes, the old session is removed and the new one takes its place. (The old session's analysis is still preserved in `CONTEXT.md` and `sessions.*.json` if you ran `drift-sync` before starting a new one — only the active session pointer in `config.json` is replaced.)

> **How `config.json` sessions work:** The `.drift/config.json` file contains a `sessions` array. The **last element** is always the active session — the one `drift-status` and `drift-sync` operate on. Previous elements are historical session starts (kept for reference, capped at 20). There is only ever one active session at a time.

**First time running Drift?** It creates a `.drift/` folder in your repo (automatically added to `.gitignore`) to store session data and credentials. If Jira isn't configured yet, it offers to collect your credentials interactively and saves them locally in `.drift/jira.json` (never committed).

---

### 2. Check progress with `drift-status`

At any point during your session:

```
drift-status
```

This is a read-only snapshot. It writes nothing, changes nothing, calls no APIs. It just shows you where you are:

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

Uncommitted: yes - 2 files

Recent sessions (Claude Code):
  1. 2025-02-13 — BAS-09 — Scaffolded dashboard with full folder structure
  2. 2025-02-12 — BAS-08 — Added user model and async database layer
```

It also shows recent session history from previous analyses so you can see what was done in earlier sessions.

At the end, it suggests next steps — like running `drift-sync` or checking the last report file if one exists.

---

### 3. Analyze and sync with `drift-sync`

This is the core of Drift. Run it when you're ready to review what you've built — typically before opening a PR:

```
drift-sync
```

**What actually happens:**

**Phase 1 — Read the session.** Drift loads your starting snapshot (SHA, ticket, planned work) from `.drift/config.json`.

**Phase 1b — Fetch the ticket from Jira.** Drift calls the Jira API and pulls the ticket's current acceptance criteria. This is the deep fetch — it looks for AC in three places: a custom "acceptance criteria" field, a checklist in the ticket description, or subtask summaries. This AC becomes the authoritative checklist for the analysis. *(Skipped if `--no-push` or `--no-jira` flag is set.)*

> **Note:** This is different from the `drift-start` fetch. `drift-start` only grabs the ticket's summary and description as a fallback for your planned work description. `drift-sync` does the thorough AC extraction that drives the progress and drift measurements.

**Phase 2 — Gather everything from git.** Full diff (not truncated), all commit messages, list of added/modified/deleted files, staged and unstaged changes. Everything since your starting SHA.

**Phase 3 — Read every changed file in full.** This is the key difference from other tools. Drift doesn't just look at diff chunks — it reads the complete content of every changed file so the analysis understands each file in context, not just the lines that changed. For architecturally significant changes, it also reads related unchanged files for context (e.g., if you added a new route, it reads the router setup to understand how it fits). For files with more than 200 lines changed, it also examines the per-file diff individually for extra detail. Binary files are skipped. Files over 1000 lines are skipped (noted in the summary). If there are 100+ changed files, it reads the top 20 by lines changed and notes the rest.

**Phase 4 — Produce the analysis.** Using the full file contents, the complete diff, and the ticket's acceptance criteria, Drift produces a structured analysis:

- **What was built** — specific file paths, function names, and approaches. Not "implemented authentication" but "added `login()` and `register()` endpoints in `router.py`, JWT middleware in `middleware.py`, and `JWT_SECRET`/`JWT_ALGORITHM`/`JWT_EXPIRY_MINUTES` settings in `config.py`."

- **Technical decisions** — every significant choice, rated:
  - **SOUND** — clearly the right call given the constraints
  - **ACCEPTABLE** — reasonable, alternatives exist but this works
  - **QUESTIONABLE** — may cause problems, worth reconsidering

- **Drift from plan** — this is where Drift compares what you built against the ticket's acceptance criteria:
  - **Scope additions** — things you built that aren't in the AC (scope creep)
  - **Scope reductions** — AC items you haven't completed yet
  - **Approach changes** — where you deviated from what was planned

- **Code review** — every changed file reviewed (see [Code Review](#code-review) below)

- **Architecture impact** — new patterns, new dependencies, API surface changes

- **Next session items** — remaining AC items, TODOs/FIXMEs/HACKs found in code (scanned automatically), tests needed, known edge cases

**Phase 5 — Update `CONTEXT.md`.** This is the persistent memory layer. Drift writes (or updates) `CONTEXT.md` in the repo root with the full analysis from this session. If a `CONTEXT.md` already exists from a previous sync, the current "Latest Session" gets demoted to the "Previous Sessions" section and the new analysis takes its place. Up to 4 previous sessions are kept — older ones roll off. (Older sessions are still in git history if you need them.) The result is a single file that always contains the current state of the project plus recent history, so anyone (or any AI assistant) can read it and immediately understand what's been happening.

**Phase 6 — Save session history.** Prepends a structured JSON entry to `.drift/sessions.claude.json` (or `sessions.cursor.json` / `sessions.codex.json` depending on which tool you're using). Newest entries first. Capped at 50 entries — oldest are trimmed when exceeded.

**Phase 7 — Post to Jira.** Automatically posts a structured comment to the ticket so your PM sees progress without you writing anything. *(Skipped if `--no-push` or `--no-jira` flag is set.)*

```
Drift Session Summary:

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

**Phase 8 — Write the report.** Saves a human-readable report as `bas-10-report.md` (ticket ID in lowercase). This is the same analysis formatted as a standalone markdown document:

```markdown
# Drift Sync Report — BAS-10

## What happened
Implemented JWT authentication using python-jose with HS256 signing.
Added login and register endpoints to router.py, middleware for token
validation in middleware.py, and updated config.py with JWT settings.

## Decisions
- SOUND: HS256 over RS256 — simpler for single-service architecture
- ACCEPTABLE: Token in Authorization header vs httpOnly cookie

## Drift from plan
- Added: password reset endpoint (not in AC — scope creep)
- Missing: token refresh endpoint (AC item #4, not started)

## Code review
3 clean | 1 concern | 1 issue
- CONCERN: JWT secret has hardcoded fallback (config.py:13)
- ISSUE: Login endpoint has no rate limiting (router.py:41)
Overall: Solid auth foundation. Rate limiting is the critical fix.

## For next session
- Implement token refresh endpoint (AC item #4)
- Add rate limiting to /login
- Move password reset to its own ticket

---
Files changed: 14 | Commits: 8 | Files analyzed: 12
CONTEXT.md updated | Jira: comment posted to BAS-10
```

Ready to paste into a PR description or share with the team.

**Important:** `drift-sync` does **not** clear the active session. You can run it multiple times on the same session — useful if you keep coding after a sync and want to re-analyze. When you re-sync, the current "Latest Session" in `CONTEXT.md` gets demoted to "Previous Sessions" and the new analysis takes the "Latest Session" slot (so you may see the same ticket appear in both sections — that's expected). The session only ends when you run `drift-start` for a new ticket.

---

### Flags

| Flag | What it does |
|---|---|
| `--no-push` | Skip all Jira integration — no ticket fetch, no comment posted. The analysis uses your `drift-start` description instead of ticket AC. You still get CONTEXT.md, the report, and session JSON. |
| `--no-jira` | Same behavior as `--no-push`. Both flags exist for readability — use whichever reads more naturally to you. |

Use either flag when you want the analysis but don't need Jira involvement — e.g., working offline, tickets aren't set up yet, or you're not ready to post.

> **Note:** There is currently no way to fetch AC from Jira but skip posting the comment. Both flags disable all Jira integration (read and write). If you need the AC-based analysis without posting, run `drift-sync` normally and the Jira comment will post — you can always delete it from the ticket afterward.

---

## What You Get

Every `drift-sync` produces four outputs:

### `CONTEXT.md` — Living context document

Written to the repo root. **This is committed to git and shared with the team.** This is the file that solves the "I closed my laptop and forgot everything" problem.

**What's in it:**

Every time you run `drift-sync`, CONTEXT.md is updated with the full analysis from that session — what was built, every technical decision and its rating, what drifted from the ticket's AC, architecture impact, code review findings (CONCERN and ISSUE only — CLEAN findings are omitted to keep it concise across sessions), and what needs to happen next.

**How it updates over time:**

CONTEXT.md isn't a log that grows forever. It's a rolling document:

- The **"Latest Session"** section always contains the full analysis from your most recent sync
- When you sync again, the previous latest session gets moved down to **"Previous Sessions"** as a shorter summary
- Up to **4 previous sessions** are kept. Older sessions roll off the bottom (but are still in git history if you ever need them)

So after three syncs across tickets BAS-10, BAS-11, and BAS-12, your CONTEXT.md would look like:

```
# Project Context

> Last updated: 2025-02-16T14:00:00Z

# Latest Session

## Session: 2025-02-16 (BAS-12)

**What happened:**
Added dashboard filtering and search. Created FilterBar component
in components/FilterBar.tsx, added search API endpoint in api/search.py...

**Decisions made:**
- SOUND: Client-side filtering for small datasets, server-side for search...

**Drift from plan:**
- Missing: export to CSV (AC item #3, not started)

**Code review:**
- CONCERN: Search endpoint has no pagination (api/search.py:45)
Overall: Clean implementation. Add pagination before dataset grows.

**For next session:**
- Implement CSV export (AC item #3)
- Add pagination to search endpoint

# Previous Sessions

## Session: 2025-02-15 (BAS-11)
  What happened: Implemented user auth with JWT tokens...
  Key decisions: HS256 signing, token in Authorization header...
  Left off: token refresh endpoint, rate limiting on /login

## Session: 2025-02-14 (BAS-10)
  What happened: Scaffolded FastAPI project structure...
  Key decisions: async SQLAlchemy, Pydantic v2...
  Left off: all endpoints still stubs
```

**Why this matters:**

- You open your laptop Monday morning — read CONTEXT.md instead of doing archaeology on your own code
- A teammate picks up your branch — they read CONTEXT.md and know what's done, what's not, and what decisions were made
- A new AI conversation starts with zero memory — CONTEXT.md gives it full project context immediately
- It's committed to git — it survives laptop closes, tool switches, and AI memory resets

### `<ticket>-report.md` — Session report

Written to the repo root (e.g., `bas-10-report.md`). A clean, human-readable version of the full analysis. Code review findings list CONCERN and ISSUE individually, with CLEAN shown as a count (e.g., "3 clean | 1 concern | 1 issue"). Overwritten each time you sync the same ticket. Report files from previous tickets are not deleted — if you synced BAS-10 and then BAS-11, both `bas-10-report.md` and `bas-11-report.md` will exist.

Useful as:
- A PR description (paste it in)
- An async standup update
- A handoff document

### Jira comment — Automatic ticket update

Posted directly to the Jira ticket. Your PM gets: progress percentage (completed AC items / total AC items), what was built, what drifted from the plan, code quality summary, and next steps. They never have to ask "how's the ticket going" — the answer is already on the ticket.

### `sessions.*.json` — Structured history

Saved in `.drift/` (gitignored, never committed). Machine-readable JSON with every analysis run. Newest entries first. Up to 50 entries per tool — oldest are trimmed when exceeded. Each entry includes: summary, decisions, drift, architecture impact, next session items, code review findings with counts, files analyzed, and whether the Jira comment was posted. Each tool writes to its own file (`sessions.claude.json`, `sessions.cursor.json`, `sessions.codex.json`).

---

## Code Review

Every `drift-sync` reviews all changed files. This isn't a separate step — it's built into the analysis. Because Drift reads the full content of every changed file (not just diff chunks), it can evaluate code in context — understanding how a function fits in its module, whether patterns are used consistently, and whether new code aligns with the existing architecture.

### Signal labels

| Signal | Meaning | Example |
|---|---|---|
| **CLEAN** | Good practice, no action needed | `CLEAN: Async session lifecycle correctly handled in get_db() (database.py:9-17) — proper commit/rollback pattern` |
| **CONCERN** | Worth watching, may become a problem | `CONCERN: JWT secret loaded from env with hardcoded fallback (config.py:13) — must be removed before deployment` |
| **ISSUE** | Should be fixed, creates real risk | `ISSUE: Login endpoint has no rate limiting (router.py:41) — brute force attack surface` |

### What it checks

1. **Approach validation** — is this the right solution for the problem?
2. **Hacky patterns** — magic numbers, tight coupling, copy-paste duplication
3. **Best practices** — framework idioms, naming conventions, error handling
4. **Architecture soundness** — right level of abstraction? over-engineered? under-engineered?
5. **Security concerns** — hardcoded secrets, auth gaps, missing input validation, injection risks
6. **Potential issues** — race conditions, resource leaks, performance bottlenecks, edge cases

Each finding includes a file path, line number, and a specific recommendation. Every review ends with a 1-3 sentence overall assessment: the most important thing to fix, the most important thing done well, and an overall quality judgment.

### Where review findings appear

- **`CONTEXT.md`** — CONCERN and ISSUE findings only. CLEAN findings are omitted to keep the document concise as sessions accumulate — you don't need to scroll past 20 "this is fine" entries to find the two things that matter.
- **`<ticket>-report.md`** — CONCERN and ISSUE findings listed individually, CLEAN as a count. Same detail level as what you see in the terminal after sync.
- **Jira comment** — summary count only (e.g., "1 concern, 1 issue: rate limiting missing"). The ticket doesn't need the full review — just the signal.

---

## How Drift Measures Progress

Drift is strict about what counts as "drift" and how progress is measured. This is intentional — without strict rules, every analysis would be full of invented requirements like "you should have written tests" when the ticket never mentioned tests.

**The rules:**

- **Only the written AC matters.** Progress and drift are measured against the ticket's acceptance criteria as literally written. If the AC says "add login endpoint" and the endpoint exists, that item is done — even if it has no tests, no rate limiting, and no error handling. Those are separate concerns.

- **If it's not in the AC, it's not missing.** If the AC doesn't mention tests, missing tests are not a scope reduction. Drift never invents requirements.

- **Scope creep is flagged, not judged.** Things you built that weren't in the AC are listed as scope additions. They're not marked as wrong — sometimes extra work is good engineering. They're surfaced so you can make a conscious decision: keep it, ticket it out, or PR it separately.

- **Future ticket work is out of scope.** If you notice something for another ticket, it goes under "future work (out of scope)" — never mixed into this session's progress.

- **Progress % is concrete.** Completed AC items divided by total AC items. 3 of 5 done = 60%. Nothing implied, nothing estimated.

**What if the ticket's AC is vague or missing?** If Drift can't find clear acceptance criteria in the ticket (no AC field, no checklist in the description, no subtasks), it falls back to the `plannedWork` description from your `drift-start` command. The analysis will still run, but progress tracking will be less precise since there's no concrete checklist to measure against.

---

## Jira Integration

### How it connects

Drift talks to Jira in two directions:

**Reading:**
- On `drift-start` — fetches the ticket's **summary and description** as fallback context (only if you didn't provide your own description)
- On `drift-sync` — fetches the ticket again with a **deep AC extraction**. Looks for acceptance criteria in three places:
  1. A custom field containing "acceptance" in its name
  2. A checklist or bullet list in the description body
  3. Subtask summaries (if no AC field or description checklist exists)

**Writing:**
- On `drift-sync` — posts a structured comment to the ticket with progress, what was done, drift, code quality, next steps, and blockers

### Setting up credentials

You need three things: your Jira URL, email, and an API token.

| Variable | Example |
|---|---|
| `JIRA_URL` | `https://yourteam.atlassian.net` |
| `JIRA_EMAIL` | `you@company.com` |
| `JIRA_TOKEN` | Generate at https://id.atlassian.com/manage-profile/security/api-tokens |

Drift checks for credentials in this order (first complete set wins):

| Priority | Source | Notes |
|---|---|---|
| 1 | Environment variables | `export JIRA_URL=...` `JIRA_EMAIL=...` `JIRA_TOKEN=...` |
| 2 | `.env` in project root | Standard `KEY=value` format (supports `export` prefix, quoted values, inline comments) |
| 3 | `.drift/.env` | Drift-specific, keeps creds out of your main `.env` |
| 4 | `~/.config/drift-nodejs/config.json` | Global config (shared with the [drift CLI](https://github.com/Hamad-Center/drift) if you use it) |
| 5 | `.drift/jira.json` | Saved when you enter creds interactively |
| 6 | Interactive prompt | Drift asks during `drift-start` or `drift-sync` if nothing else is found |

If you've never set up Jira, the first time you run `drift-start` it will ask for your credentials and save them to `.drift/jira.json` (which is gitignored — your credentials never get committed).

### Running without Jira

Drift works fine without Jira. Use `--no-jira` on sync, or just don't configure credentials. The analysis will use whatever description you provided at `drift-start` instead of ticket AC. You still get CONTEXT.md, the report file, and the code review — just no Jira comment and no AC-based progress tracking.

---

## Install

First, clone this repo (or download the files):

```bash
git clone git@github.com:Hamad-Center/drift-skills.git
cd drift-skills
```

Then copy the skill files for your tool:

### Claude Code

```bash
mkdir -p ~/.claude/skills
cp -r claude-code/* ~/.claude/skills/
```

Invoke with slash commands:
```
/drift-start BAS-10
/drift-status
/drift-sync
/drift-sync --no-push
```

### Cursor

```bash
mkdir -p ~/.cursor/rules
cp cursor/* ~/.cursor/rules/
```

Invoke by typing in agent mode chat:
```
drift-start BAS-10
drift-status
drift-sync
drift-sync --no-jira
```

### Codex

```bash
mkdir -p ~/.agents/skills
cp -r codex/* ~/.agents/skills/
```

Invoke via `/skills` or `$` mentions:
```
drift-start BAS-10
drift-status
drift-sync
```

Each tool folder has its own README with detailed setup instructions and tool-specific notes.

**Prerequisites:** Git (Drift uses git commands extensively). No other dependencies — the skill files are just instructions that your AI assistant follows.

---

## Cross-Tool Compatibility

All three versions share the same `.drift/` directory, `CONTEXT.md`, and Jira credentials. You can switch between Claude Code, Cursor, and Codex on the same project without losing anything.

| Shared (works across all tools) | Separate (per tool) |
|---|---|
| `.drift/config.json` — active session state | `.drift/sessions.claude.json` — Claude Code analysis history |
| `.drift/jira.json` — Jira credentials | `.drift/sessions.cursor.json` — Cursor analysis history |
| `.drift/.env` — environment variables | `.drift/sessions.codex.json` — Codex analysis history |
| `CONTEXT.md` — latest analysis | |
| `<ticket>-report.md` — latest report | |

Running `drift-status` shows recent session history from previous analyses so you can see activity across sessions.

---

## What Gets Created in Your Repo

```
your-repo/
├── CONTEXT.md                   # Written by drift-sync (committed, shared with team)
├── bas-10-report.md             # Written by drift-sync (committed, shareable)
│
└── .drift/                      # Auto-gitignored on first drift-start
    ├── config.json              # Session state (max 20 sessions)
    ├── sessions.json            # Drift CLI session history
    ├── sessions.claude.json     # Claude Code analysis history (max 50)
    ├── sessions.cursor.json     # Cursor analysis history (max 50)
    ├── sessions.codex.json      # Codex analysis history (max 50)
    ├── jira.json                # Jira credentials (if entered interactively)
    └── .env                     # Optional drift-specific env vars
```

`CONTEXT.md` and report files are meant to be committed — they're the artifacts your team reads. Everything in `.drift/` is local-only and never committed.

> The `sessions.json` file (no tool suffix) is the session history for the [drift CLI tool](https://github.com/Hamad-Center/drift), a separate Node.js CLI that also reads the shared `.drift/` directory. The skills in this repo work independently of the CLI — you don't need it installed.

---

## Typical Workflow

```
# Pick up a Jira ticket
drift-start BAS-10

# Drift fetches the ticket from Jira automatically.
# You see: ticket title, description, and confirmation that the session started.

# ... code with your AI assistant ...

# Curious how much you've changed?
drift-status
# Shows commits, files changed, diff stats. Writes nothing.

# ... keep coding ...

# Done for now. Run the full analysis.
drift-sync

# Drift reads every changed file, fetches the ticket's AC from Jira,
# compares what you built vs what the AC says, reviews the code,
# writes CONTEXT.md, writes bas-10-report.md,
# and posts a progress comment to BAS-10 in Jira.

# Want to keep coding and re-analyze? Just run drift-sync again.
# The session stays active until you start a new one.

# Next ticket:
drift-start BAS-11

# Drift asks if you want to replace the BAS-10 session. You say yes.
# The new session begins. CONTEXT.md still has the BAS-10 analysis
# in its "Previous Sessions" section.
```

---

## Edge Cases

| Scenario | What happens |
|---|---|
| No changes since session start | Reports "No changes" and stops — no files written |
| Binary files in the diff | Skipped, noted as "binary file, not analyzed" |
| A changed file is over 1000 lines | File content skipped, noted as "too large for inline analysis" (diff is still analyzed) |
| More than 100 changed files | Reads the top 20 files by lines changed, notes the rest in the summary |
| No planned work and no Jira AC | "Drift from plan" section is omitted entirely — nothing to measure against |
| `startSha` no longer exists (after rebase) | Falls back to `git merge-base`, fails gracefully with a clear message if that also fails |
| Jira API returns an error | Warns you and continues the analysis without ticket context |
| Jira comment fails to post | Warns you but still writes CONTEXT.md and the report — you don't lose the analysis |
| Old `CONTEXT.claude.md` exists (pre-rename) | Automatically renamed to `CONTEXT.md`, previous session history preserved |
| You run `drift-start` with an active session | Shows the current session and asks if you want to replace it. If yes, the old session is removed from the active sessions list |
| You run `drift-sync` and keep coding | Session stays active. Run `drift-sync` again to re-analyze with the new changes |

---

## Repo Structure

```
drift-skills/
├── README.md                           # This file
├── CLAUDE.md                           # Project context for AI assistants
├── claude-code/                        # Claude Code skills
│   ├── README.md                       #   Setup + usage guide
│   ├── drift-start/SKILL.md
│   ├── drift-sync/SKILL.md
│   └── drift-status/SKILL.md
├── cursor/                             # Cursor rules
│   ├── README.md                       #   Setup + usage guide
│   ├── drift-start.mdc
│   ├── drift-sync.mdc
│   └── drift-status.mdc
├── codex/                              # Codex skills
│   ├── README.md                       #   Setup + usage guide
│   ├── drift-start/SKILL.md
│   ├── drift-sync/SKILL.md
│   └── drift-status/SKILL.md
└── docs/
    └── DRIFT_DOCS_DETAILED.md          # Full technical reference
```

---

## Full Documentation

For the complete technical reference — every phase of `drift-sync` explained in detail, the full data model with JSON schemas, all scoping rules with rationale, every edge case, and the design decisions behind each choice — see [DRIFT_DOCS_DETAILED.md](docs/DRIFT_DOCS_DETAILED.md).
