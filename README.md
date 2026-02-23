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

Drift is a set of AI assistant rules that run inside your existing tools — Claude Code, Cursor, or Codex. Not a separate app, not a CLI tool, not a CI plugin. You don't change your workflow. You just get three new commands:

```
drift-start   →  start tracking a session against a Jira ticket
drift-status  →  see what changed so far (read-only, writes nothing)
drift-sync    →  analyze everything, review the code, update Jira
```

---

## How It Works

### 1. Start a session with `drift-start`

Before you write any code, tell Drift what ticket you're working on:

```
drift-start BAS-10
```

That's it. Just the ticket ID.

**What happens behind the scenes:**
- Drift saves your current git SHA, branch, and timestamp as the starting snapshot
- If Jira is configured, it **automatically fetches the ticket's title, description, and acceptance criteria** from Jira — so it knows exactly what you're supposed to build
- The acceptance criteria become the checklist that everything is measured against later

You can optionally add a description if you want to override or supplement what's in Jira:

```
drift-start BAS-10 implement JWT authentication
```

But if Jira is connected, you usually don't need to — Drift already has the full ticket context.

**First time running Drift?** It creates a `.drift/` folder in your repo (automatically gitignored) to store session data and credentials. If Jira isn't configured yet, it asks for your credentials interactively and saves them locally.

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
```

It also shows recent session history from all tools (Claude Code, Cursor, Codex, CLI) so you can see what was done in previous sessions regardless of which tool was used.

---

### 3. Analyze and sync with `drift-sync`

This is the core of Drift. Run it when you're ready to review what you've built — typically before opening a PR:

```
drift-sync
```

**What actually happens (8 phases):**

**Phase 1 — Read the session.** Drift loads your starting snapshot (SHA, ticket, planned work) from `.drift/config.json`.

**Phase 1b — Fetch the ticket from Jira.** Drift calls the Jira API and pulls the ticket's current acceptance criteria. This becomes the authoritative checklist — not your memory of what the ticket said, but the actual written AC.

**Phase 2 — Gather everything from git.** Full diff (not truncated), all commit messages, list of added/modified/deleted files, staged and unstaged changes. Everything since your starting SHA.

**Phase 3 — Read every changed file in full.** This is the key difference from other tools. Drift doesn't just look at diff chunks — it reads the complete content of every changed file. For architecturally significant changes, it also reads related unchanged files for context (e.g., if you added a new route, it reads the router setup to understand how it fits). Binary files are skipped. Files over 1000 lines are skipped. If there are 100+ changed files, it reads the top 20 by lines changed.

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

- **Next session items** — remaining AC items, TODOs/FIXMEs found in code, tests needed, known edge cases

**Phase 5 — Write `CONTEXT.md`.** A living context document in the repo root (see [What You Get](#what-you-get)).

**Phase 6 — Save session history.** Appends structured JSON to `.drift/sessions.*.json` for future reference.

**Phase 7 — Post to Jira.** Automatically posts a structured comment to the ticket so your PM sees progress without you writing anything:

```
Drift Session Summary:

Progress: ~70% of this ticket's acceptance criteria

What was done:
JWT auth implemented with login/register endpoints and token
validation middleware. 3 of 5 AC items completed.

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

**Phase 8 — Write the report.** Saves a human-readable report as `bas-10-report.md` (ticket ID in lowercase). Ready to paste into a PR description.

---

### Flags

```
drift-sync --no-push    # run full analysis but don't post the Jira comment
drift-sync --no-jira    # skip all Jira integration (no fetch, no post)
```

Use `--no-push` when you want the analysis but aren't ready to update the ticket. Use `--no-jira` when working without Jira entirely — the analysis will use your `drift-start` description instead of ticket AC.

---

## What You Get

Every `drift-sync` produces four outputs:

### `CONTEXT.md` — Living context document

Written to the repo root. This is the file that solves the "what happened" problem. It contains:

- The full analysis from your latest session (what was built, decisions, drift, code review, next steps)
- Summarized history of up to 4 previous sessions

When you open your laptop Monday morning, or when a teammate picks up your branch, or when a new AI conversation starts with zero memory — `CONTEXT.md` is the answer to "what's going on in this repo." It's committed to git and shared with the team.

### `<ticket>-report.md` — Session report

Written to the repo root (e.g., `bas-10-report.md`). A clean, human-readable version of the analysis. Overwritten every sync. Useful as:
- A PR description (paste it in)
- An async standup update
- A handoff document

### Jira comment — Automatic ticket update

Posted directly to the Jira ticket. Your PM gets: progress percentage (completed AC items / total), what was built, what drifted from the plan, code quality summary, and next steps. They never have to ask "how's the ticket going" — the answer is already on the ticket.

### `sessions.*.json` — Structured history

Saved in `.drift/` (gitignored). Machine-readable JSON with every analysis run. Up to 50 entries. Includes the full analysis data: summary, decisions, drift, code review findings, file counts. Useful for tooling, auditing, or dashboards.

---

## Code Review

Every `drift-sync` reviews all changed files. This isn't a separate step — it's built into the analysis. Because Drift reads the full content of every changed file (not just diff chunks), it can evaluate code in context.

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

- **`CONTEXT.md`** — only CONCERN and ISSUE findings (keeps the document concise across sessions)
- **`<ticket>-report.md`** — all findings including CLEAN (the full picture)
- **Jira comment** — summary count only (e.g., "1 concern, 1 issue: rate limiting missing")

---

## How Drift Measures Progress

Drift is strict about what counts as "drift" and how progress is measured. This is intentional — without strict rules, every analysis would be full of invented requirements like "you should have written tests" when the ticket never mentioned tests.

**The rules:**

- **Only the written AC matters.** Progress and drift are measured against the ticket's acceptance criteria as literally written. If the AC says "add login endpoint" and the endpoint exists, that item is done — even if it has no tests, no rate limiting, and no error handling. Those are separate concerns.

- **If it's not in the AC, it's not missing.** If the AC doesn't mention tests, missing tests are not a scope reduction. Drift never invents requirements.

- **Scope creep is flagged, not judged.** Things you built that weren't in the AC are listed as scope additions. They're not marked as wrong — sometimes extra work is good engineering. They're surfaced so you can make a conscious decision: keep it, ticket it out, or PR it separately.

- **Future ticket work is out of scope.** If you notice something for another ticket, it goes under "future work (out of scope)" — never mixed into this session's progress.

- **Progress % is concrete.** Completed AC items divided by total AC items. Nothing implied, nothing estimated.

---

## Jira Integration

### How it connects

Drift talks to Jira in two directions:

**Reading (on `drift-start` and `drift-sync`):**
Fetches the ticket's summary, description, status, and acceptance criteria. The AC is found in this order:
1. A custom field containing "acceptance" in its name
2. A checklist or bullet list in the description body
3. Subtask summaries (if no AC field or description checklist exists)

**Writing (on `drift-sync`):**
Posts a structured comment to the ticket with progress, what was done, drift, code quality, next steps, and blockers.

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
| 2 | `.env` in project root | Standard `KEY=value` format |
| 3 | `.drift/.env` | Drift-specific, keeps creds out of your main `.env` |
| 4 | `~/.config/drift-nodejs/config.json` | Global config (shared with drift CLI if you use it) |
| 5 | `.drift/jira.json` | Saved when you enter creds interactively |
| 6 | Interactive prompt | Drift asks during `drift-start` or `drift-sync` if nothing else is found |

If you've never set up Jira, the first time you run `drift-start` it will ask for your credentials and save them to `.drift/jira.json` (which is gitignored — your credentials never get committed).

### Running without Jira

Drift works fine without Jira. Use `--no-jira` on sync, or just don't configure credentials. The analysis will use whatever description you provided at `drift-start` instead of ticket AC. You still get CONTEXT.md, the report file, and the code review — just no Jira comment and no AC-based progress tracking.

---

## Install

### Claude Code

```bash
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
cp -r codex/* ~/.agents/skills/
```

Invoke via `/skills` or `$` mentions:
```
drift-start BAS-10
drift-status
drift-sync
```

Each tool folder has its own README with detailed setup instructions and tool-specific usage notes.

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

Running `drift-status` in any tool shows session history from all sources, so you always see the full picture.

---

## What Gets Created in Your Repo

```
your-repo/
├── CONTEXT.md                   # Written by drift-sync (committed, shared with team)
├── bas-10-report.md             # Written by drift-sync (committed, shareable)
│
└── .drift/                      # Auto-gitignored on first drift-start
    ├── config.json              # Session state (max 20 sessions)
    ├── sessions.json            # CLI session history
    ├── sessions.claude.json     # Claude Code analysis history (max 50)
    ├── sessions.cursor.json     # Cursor analysis history (max 50)
    ├── sessions.codex.json      # Codex analysis history (max 50)
    ├── jira.json                # Jira credentials (if entered interactively)
    └── .env                     # Optional drift-specific env vars
```

`CONTEXT.md` and the report file are meant to be committed — they're the artifacts your team reads. Everything in `.drift/` is local-only and never committed.

---

## Typical Workflow

```
# Pick up a Jira ticket
drift-start BAS-10

# Drift fetches the ticket from Jira automatically.
# You see: ticket title, AC items, and confirmation that the session started.

# ... code with your AI assistant ...

# Curious how much you've changed?
drift-status

# ... keep coding ...

# Done for now. Run the full analysis.
drift-sync

# Drift reads every changed file, compares against the ticket's AC,
# reviews the code, writes CONTEXT.md, writes bas-10-report.md,
# and posts a progress comment to BAS-10 in Jira.

# Next ticket:
drift-start BAS-11

# The previous session is archived. A new session begins.
```

---

## Edge Cases

| Scenario | What happens |
|---|---|
| No changes since session start | Reports "No changes" and stops — no files written |
| Binary files in the diff | Skipped, noted as "binary file, not analyzed" |
| A changed file is over 1000 lines | File content skipped, noted as "too large for inline analysis" (diff is still analyzed) |
| More than 100 changed files | Reads the top 20 files by lines changed, notes the rest in the summary |
| No planned work description | "Drift from plan" section is omitted entirely — nothing to measure against |
| `startSha` no longer exists (after rebase) | Falls back to `git merge-base`, fails gracefully with a clear message if that also fails |
| Jira API returns an error | Warns you and continues the analysis without ticket context |
| Jira comment fails to post | Warns you but still writes CONTEXT.md and the report — you don't lose the analysis |
| Old `CONTEXT.claude.md` exists from before | Automatically renamed to `CONTEXT.md`, previous session history preserved |
| You run `drift-start` with an active session | Shows the current session and asks if you want to replace it |

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
