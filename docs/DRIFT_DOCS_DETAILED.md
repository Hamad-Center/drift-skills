# Drift — Full Technical Documentation

> AI-powered session tracking that connects your git changes to Jira tickets, with deep code analysis and automatic context preservation.

---

## Table of Contents

1. [The Problem](#the-problem)
2. [What Drift Is](#what-drift-is)
3. [Core Concepts](#core-concepts)
4. [Command Reference](#command-reference)
   - [drift-start](#drift-start)
   - [drift-status](#drift-status)
   - [drift-sync](#drift-sync)
5. [The Analysis Engine](#the-analysis-engine)
6. [Code Review System](#code-review-system)
7. [Output Files](#output-files)
8. [Jira Integration](#jira-integration)
9. [Data Model](#data-model)
10. [Scoping Rules](#scoping-rules)
11. [Flags & Options](#flags--options)
12. [File Structure](#file-structure)
13. [Installation](#installation)
14. [Tool Compatibility](#tool-compatibility)
15. [Edge Cases](#edge-cases)
16. [Design Decisions](#design-decisions)

---

## The Problem

AI-assisted development has a speed problem — not too slow, too fast.

When you pair with Claude or Cursor, you can go from zero to 40 changed files in two hours. That velocity is the point. But it creates four failure modes that compound on each other:

### Failure Mode 1: Invisible Scope Creep

You open a ticket: *"Add a login endpoint."* You start coding, the AI suggests related improvements, one thing leads to another, and by the time you're done you've also added password reset, email verification, a user settings model, and a helper utility you'll probably need later.

None of that was in the ticket. You didn't notice you were doing it because each step felt like a natural next step. The ticket says one thing. The code says another. Nobody knows until a PM or QA reads both at the same time.

### Failure Mode 2: Forgotten Acceptance Criteria

Tickets have acceptance criteria. Real, specific, agreed-upon requirements. By the time you're deep in the code, those criteria have faded out of working memory. You finish the session feeling confident — you solved the problem — and you ship. Then someone reads the AC and notices you missed item 3 of 5.

The gap isn't negligence. It's that there was no mechanism to keep the AC visible throughout the session. You checked it at the start and never again.

### Failure Mode 3: Context Loss Between Sessions

You close the laptop Friday at 5. You open it Monday at 9. The code is there. The ticket is there. What's missing is everything in between: why you chose this approach over that one, what you were in the middle of, what the next three steps were, what you noticed but decided to punt on.

AI assistants have no persistent memory. Your git log says `fix`, `fix2`, `wip`, `trying something`. There is no CONTEXT.md. You spend 45 minutes doing archaeology on your own code before you can write a single new line.

### Failure Mode 4: Invisible Quality Gaps

Moving fast with AI assistance means you're accepting code faster than you can critically evaluate it. Security gaps, missing error handling, hardcoded values, missing rate limiting — they get merged because no one stopped to look. Not because the review was skipped maliciously, but because the review was "it looks right and the vibe is good."

---

## What Drift Is

Drift is a set of AI assistant rules — not a standalone app, not a CLI tool, not a CI plugin. It runs inside **Claude Code** and **Cursor** as first-class commands. You don't install anything separately. You don't change your workflow. You add three commands to your existing sessions.

```
drift-start   →  mark the starting line
drift-status  →  read-only progress check
drift-sync    →  deep analysis + Jira update
```

Each command runs entirely inside the AI assistant's session. It uses the assistant's ability to read files, run shell commands, and reason about code — with no truncation, no character limits, no summarization of summaries.

**What makes it different from the drift CLI:**

The drift CLI tool truncates diffs at 8,000 characters. For most real sessions, this means the CLI sees a fraction of the actual changes. Drift reads the full diff, the full content of every changed file, and related unchanged files for context. This is the core advantage — the analysis is based on complete information.

---

## Core Concepts

### Session

A session is a bounded unit of work. It starts with `drift-start` and ends implicitly when you run `drift-start` again for the next ticket. A session records:
- When it started
- What git SHA you were at when it started
- What ticket you were working on
- What you said you planned to do

Everything Drift measures — progress, drift, code quality — is relative to this starting snapshot.

### Active Session

The last element of the `sessions` array in `.drift/config.json`. Drift always operates on the active session. There's only ever one.

### Drift

"Drift" specifically means the gap between what was written in the ticket's acceptance criteria and what was actually implemented. Not the gap between a developer's intuition and the code. Not the gap between best practice and reality. The literal gap between what the AC text says and what git diff shows.

This definition matters. It means Drift never invents requirements. If the AC doesn't mention tests, missing tests are not drift. If the AC says "create folder structure" and the folders exist, that AC item is done even if the folders are empty.

### Ticket Context

When Jira is configured, `drift-start` fetches the full ticket at session start and `drift-sync` fetches it again during analysis. The ticket's acceptance criteria become the authoritative checklist for everything that follows. Without ticket context, Drift falls back to the `plannedWork` text you provided at session start.

---

## Command Reference

### drift-start

**Purpose:** Mark the starting line for a new session.

**Invocation:**
```
/drift-start <TICKET-ID> <optional planned work description>
```

**Examples:**
```
/drift-start BAS-10 implement JWT authentication
/drift-start BAS-10
/drift-start PROJ-123 refactor user repository layer
```

**What it does, step by step:**

**1. Verify git repository**
Runs `git rev-parse --git-dir`. If this isn't a git repo, stops immediately.

**2. Initialize `.drift/` on first run**
If `.drift/config.json` doesn't exist yet:
- Creates the `.drift/` directory
- Writes `config.json` with `{"initialized": true, "sessions": []}`
- Writes `sessions.json` with `[]`
- Adds `.drift/` to `.gitignore` (creates `.gitignore` if it doesn't exist)

The `.drift/` directory is always gitignored. Session data, credentials, and analysis history never get committed.

**3. Check for an existing active session**
If the `sessions` array already has entries, shows the current active session and asks if you want to overwrite it. If you say no, stops. If you say yes, pops the last session and proceeds.

**4. Parse your input**
Extracts the ticket ID using the pattern `[A-Za-z][A-Za-z0-9]+-\d+` (e.g., `BAS-10`, `PROJ-123`, `AB-1`). Normalizes to uppercase. Everything else in your message becomes `plannedWork`.

Ticket ID is **required**. If you don't provide one, Drift asks for it before continuing.

**5. Capture git state**
```bash
git rev-parse HEAD         → startSha (full 40-char SHA)
git rev-parse --abbrev-ref HEAD  → branch name
```

**6. Write session to config**
Pushes a new session object onto the array and writes it back:
```json
{
  "startedAt": "2025-02-14T16:47:00.000Z",
  "startSha": "a1b2c3d4e5f6...",
  "ticket": "BAS-10",
  "plannedWork": "implement JWT authentication"
}
```
The array is capped at 20 sessions. If adding this session pushes it over 20, the oldest entries are trimmed from the front.

**7. Jira fallback for plannedWork**
If you provided a ticket but no planned work description, and Jira credentials are available, Drift fetches the ticket and sets `plannedWork` to the ticket's `summary + description`. This means even if you type just `/drift-start BAS-10`, the session still has rich planned work context pulled from the actual ticket.

**8. Jira credential setup**
Checks all credential sources (see [Jira Integration](#jira-integration)). If Jira is configured, tells you sync will post a comment. If not, offers to set it up interactively.

**Output:**
```
Session started
  Branch: feature/auth
  SHA:    a1b2c3d
  Ticket: BAS-10
  Plan:   implement JWT authentication
```

---

### drift-status

**Purpose:** Read-only snapshot of the current session. See what's changed, how much, and what's coming.

**Invocation:**
```
/drift-status
```

**What it does, step by step:**

**1. Read session state**
Reads `.drift/config.json`. If it doesn't exist or is uninitialized, tells you to run `drift-start` and stops.

Gets the active session (last element of `sessions` array). If the array is empty, tells you there's no active session.

**2. Gather git data**
Using the `startSha` from the active session:
```bash
git log --oneline <startSha>..HEAD    # commits since start
git diff --stat <startSha>            # diff statistics
git diff --name-only <startSha>       # list of changed files
git rev-parse --abbrev-ref HEAD       # current branch
git rev-parse --short HEAD            # current SHA
git status --porcelain                # uncommitted changes
```

**3. Display status**
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
  ...

Uncommitted: yes - 2 files
```

If nothing has changed since the session started, says so and stops.

**4. Session history**
Reads `.drift/sessions.cursor.json`, `.drift/sessions.claude.json`, and `.drift/sessions.json` if they exist, and shows the last 5 entries from each:
```
Recent sessions (Cursor):
  1. 2025-02-14 — BAS-09 — Scaffolded dashboard with full folder structure...
  2. 2025-02-13 — BAS-08 — Added user model and async database layer...

Recent sessions (CLI):
  1. 2025-02-12 — BAS-07 — Initial FastAPI setup with uvicorn...
```

**5. Suggest next steps**
```
- drift-sync — analyze changes and write CONTEXT.md
- The last sync report is in bas-10-report.md if it exists
```

**Important:** `drift-status` writes nothing. No files are created or modified. It is purely observational.

---

### drift-sync

**Purpose:** Deep multi-phase analysis of everything that changed since the session started. Writes CONTEXT.md, posts to Jira, produces a report file.

**Invocation:**
```
/drift-sync
/drift-sync --no-push
/drift-sync --no-jira
```

`drift-sync` runs 8 phases in sequence. Here's exactly what happens in each:

---

#### Phase 1 — Validate session

Confirms this is a git repo. Reads `.drift/config.json`, verifies `initialized: true`, gets the active session. Extracts `startSha`, `startedAt`, `ticket`, `plannedWork`. Stops with a clear message if any of these are missing or invalid.

---

#### Phase 1b — Resolve Jira credentials and fetch ticket context

**Skipped if:** No ticket on the session, or `--no-jira` / `--no-push` flag present.

Resolves credentials in order (see [Jira Integration](#jira-integration)). If no credentials are found, offers to collect them interactively and save to `.drift/jira.json`.

Once credentials are available, fetches the ticket:
```bash
curl -s -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  "<url>/rest/api/3/issue/<ticket>"
```

Extracts `ticketContext`:
- `summary` — ticket title
- `description` — full text from the ADF body
- `acceptanceCriteria` — checked in this order: custom AC field → checklist in description body → subtask summaries
- `status` — current workflow status

This `ticketContext`, especially the AC, becomes the authoritative measure for everything in Phase 4.

---

#### Phase 2 — Gather git data

Runs a comprehensive set of git commands using `startSha`:

```bash
git diff --name-only <startSha>               # all changed files
git diff --stat <startSha>                    # lines added/removed per file
git log --oneline <startSha>..HEAD            # commit messages
git diff <startSha>                           # full diff (not truncated)
git diff --name-only --diff-filter=A <startSha>  # added files only
git diff --name-only --diff-filter=M <startSha>  # modified files only
git diff --name-only --diff-filter=D <startSha>  # deleted files only
git diff --cached --name-only                 # staged uncommitted
git diff --name-only                          # unstaged changes
```

Committed and uncommitted changes are both included. Staged and unstaged changes are tracked separately and noted in the analysis.

---

#### Phase 3 — Deep file analysis

**This is the key difference from the CLI.**

For each changed file:
1. **Skip binary files** — detected via `git diff --numstat` (binary files show `-` for add/delete counts). Noted as "binary file, not analyzed."
2. **Skip files over 1000 lines** — noted as "too large for inline analysis."
3. **Read the full content** of every remaining file — not a diff, the complete current file.

If there are more than 100 changed files, runs `git diff --numstat` to get line counts and reads only the top 20 by total lines changed. The rest are noted in the summary.

For architecturally significant changes (new modules, API changes, config changes, new dependencies), also reads related unchanged files that provide context — e.g., if a new route was added, reads the router setup file to understand how it fits.

For files with more than 200 lines changed, also examines the per-file diff individually: `git diff <startSha> -- <file>`.

The result is a complete picture of the codebase state — not a grep, not a truncated diff, but the actual content of every changed file read in full.

---

#### Phase 4 — Structured analysis

This is the output of all the reasoning, built from everything gathered in Phases 2 and 3. Seven sub-sections:

**4a. What happened**
3–5 sentences. What was actually built. Specific file paths, function names, approaches. Concrete, not vague.

**4b. Technical decisions**
Every significant architectural or implementation decision. Rated:
- **SOUND** — clearly the right choice given the constraints
- **ACCEPTABLE** — reasonable, alternatives exist
- **QUESTIONABLE** — may cause problems, worth reconsidering

Each decision includes what was chosen, what wasn't chosen, and which files reflect it.

**4c. Drift from plan**
Only included if `plannedWork` was provided. Three categories:
- **Scope additions** — built things not in the AC
- **Scope reductions** — AC items not yet completed
- **Approach changes** — planned one way, implemented another

Evaluated strictly against the ticket's written AC (see [Scoping Rules](#scoping-rules)).

**4d. Architecture impact**
- New patterns introduced
- Dependency changes (added packages, removed packages)
- API surface changes (new endpoints, changed interfaces, breaking changes)
- New files/modules and their role
- Changes that affect other parts of the codebase

**4e. For next session**
Organized by scope:
1. **Remaining ticket work** — unfinished AC items from this ticket, with file paths
2. **Future work (out of scope)** — things noticed that belong to other tickets, labeled clearly as out of scope

Plus: TODOs found in code (scans changed files for `TODO`, `FIXME`, `HACK`), tests that need writing, known edge cases.

**4f. Ticket comment**
The structured Jira comment (see [Jira Integration](#jira-integration)).

**4g. Code & architecture review**
See [Code Review System](#code-review-system).

---

#### Phase 5 — Write CONTEXT.md

Writes `CONTEXT.md` in the repository root. See [Output Files](#output-files) for the full format.

**Migration rule:** If `CONTEXT.md` doesn't exist but `CONTEXT.claude.md` does (from older sessions before the rename), renames the old file first using `mv`, then proceeds. This preserves previous session history across the filename change.

---

#### Phase 6 — Write session history JSON

Prepends a new entry to `.drift/sessions.claude.json` (Claude Code) or `.drift/sessions.cursor.json` (Cursor). Keeps a maximum of 50 entries. See [Data Model](#data-model) for the full structure.

---

#### Phase 7 — Post to Jira

**Skipped if:** No ticket, no credentials, or `--no-push` / `--no-jira` flag.

Posts the ticket comment from Phase 4f as an Atlassian Document Format (ADF) comment:
```bash
curl -s -w "\n%{http_code}" -u "<email>:<token>" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{ "body": { "type": "doc", "version": 1, ... } }' \
  "<url>/rest/api/3/issue/<ticket>/comment"
```

Checks the HTTP status code. 2xx → `jiraCommentPosted: true`. Failure → warns the user but does not stop the sync. The analysis is always written regardless of whether Jira succeeds.

---

#### Phase 8 — Report to user

Displays a formatted summary in the chat and writes the same content as `<ticket>-report.md` (e.g., `bas-10-report.md`) or `session-report.md` if there's no ticket. Overwrites any existing report file.

```
Drift sync complete (Claude deep analysis)

What happened:
  Implemented JWT authentication with login, register, and
  token validation middleware...

Decisions:
  - SOUND: HS256 over RS256 — simpler for single-service architecture
  - ACCEPTABLE: Token in Authorization header vs httpOnly cookie

Drift from plan:
  - Added: password reset endpoint (not in AC)
  - Missing: token refresh (AC item #4)

Code review:
  3 clean | 1 concern | 1 issue
  - CONCERN: JWT secret has hardcoded fallback (config.py:13)
  - ISSUE: Login endpoint has no rate limiting (router.py:41)
  Overall: Solid auth foundation. Rate limiting gap is the
  critical fix before merge.

For next session:
  - Implement token refresh endpoint (AC item #4)
  - Add rate limiting to /login
  - Remove or ticket-out the password reset endpoint

---
Files changed: 14 | Commits: 8 | Files analyzed: 12
CONTEXT.md updated
Jira: comment posted to BAS-10
```

---

## The Analysis Engine

What makes `drift-sync` different from a simple diff review is the depth of what it reads and the structure of what it produces.

**Full context, not summaries.** Every changed file is read in its entirety. Not line-by-line diff chunks — the full current state of the file. This means the analysis can reason about how a function fits in its module, whether a pattern is used consistently, whether a newly added dependency is already used elsewhere.

**Related unchanged files.** When a new route is added, the analysis also reads the router configuration to understand how the route is mounted. When a new model is added, it reads the migration setup. Architecture analysis requires seeing the parts that didn't change, not just the parts that did.

**Commit history as narrative.** The `git log --oneline` output is read as a story — 8 commits over two hours tells you something about how the implementation evolved. Lots of small fixes after an initial commit suggests iteration on a hard problem. One big commit suggests either very focused work or a compressed history.

**Scoped to the ticket, not the codebase.** The analysis doesn't try to evaluate the entire codebase. It evaluates the delta — what changed — against what the ticket asked for. This is intentional. Drift is not a general code quality tool. It's a session tracking tool.

---

## Code Review System

Every `drift-sync` includes a code review of all changed files. It uses three signal labels:

### CLEAN
Good practice. No action needed. Explicitly included to acknowledge what's done well, not just surface problems.

> `- CLEAN: Async session lifecycle correctly handled in get_db() (database.py:9-17) — proper commit/rollback pattern`

### CONCERN
Not wrong, but worth watching. May become a problem as the codebase grows or requirements change. No immediate action required, but worth keeping in mind.

> `- CONCERN: JWT secret loaded from env with hardcoded fallback (config.py:13) — not dangerous locally, but must be removed before any deployment`

### ISSUE
Should be fixed. Creates real risk in security, correctness, or maintainability. These are the findings that warrant a fix before merging.

> `- ISSUE: Login endpoint has no rate limiting (router.py:41) — brute force attack surface; add slowapi or equivalent before deploy`

### Review Categories

Findings are organized into six categories. Any category with no findings is omitted:

1. **Approach validation** — Was this the right way to solve the problem? Are there simpler alternatives that were missed?
2. **Hacky patterns** — Magic numbers, tight coupling, workarounds, copy-paste duplication, quick fixes with known problems
3. **Best practices** — Framework idiom adherence, naming conventions, error handling patterns, standard library usage
4. **Architecture soundness** — Right level of abstraction? Over-engineered? Under-engineered? Speculative generalization?
5. **Security concerns** — Hardcoded secrets, auth gaps, missing input validation, injection risks, sensitive data exposure
6. **Potential issues** — Race conditions, missing error handling, performance bottlenecks, resource leaks, edge cases

### Overall Assessment

Every review ends with a required 1–3 sentence overall assessment: the single most important thing to fix, the single most important thing done well, and a quality judgment.

### What Goes in CONTEXT.md vs the Report

`CONTEXT.md` only includes CONCERN and ISSUE findings. CLEAN findings are omitted to keep the document concise across many sessions. The full review including CLEAN findings appears in the `<ticket>-report.md`.

---

## Output Files

### CONTEXT.md

A rolling context document that lives in the repo root. Updated by every sync. Contains the latest session in full and up to 4 previous sessions summarized. Designed to be the answer to "what happened in this repo and what's the current state."

```markdown
# Project Context

> This file is auto-generated by drift (Claude Code skill). It provides deep analysis of recent changes.
> Last updated: 2025-02-14T19:23:00Z
> Analysis method: claude-full-context (no truncation, multi-step file analysis)

# Latest Session

## Session: 2025-02-14 (BAS-10)

**What happened:**
Implemented JWT authentication using python-jose with HS256 signing.
Added login and register endpoints to router.py, middleware for token
validation in middleware.py, and updated config.py with JWT_SECRET,
JWT_ALGORITHM, and JWT_EXPIRY_MINUTES settings.

**Decisions made:**
- HS256 over RS256 — SOUND: single-service architecture doesn't need
  asymmetric keys; simpler key management
- Token in Authorization header over httpOnly cookie — ACCEPTABLE:
  appropriate for API clients; would need to change for browser-first auth

**Drift from plan:**
- Added: password reset endpoint (not in BAS-10 AC — scope creep)
- Missing: token refresh endpoint (AC item #4, not started)

**Architecture impact:**
- New middleware layer in middleware.py for all authenticated routes
- python-jose dependency added to requirements.txt
- JWT_SECRET env var now required for secure operation

**Code review:**
- CONCERN: JWT secret has hardcoded fallback (config.py:13) — remove
  before any deployment
- ISSUE: Login endpoint has no rate limiting (router.py:41) — brute
  force risk
Overall: Solid auth foundation with correct async patterns. Rate
limiting is the critical gap before this is production-ready.

**For next session:**
- Implement token refresh endpoint (BAS-10 AC item #4)
- Add rate limiting to /login endpoint
- Move password reset to its own ticket

# Previous Sessions

---

## Session: 2025-02-13 (BAS-09)

**What happened:**
Scaffolded the FastAPI project structure with async SQLAlchemy...
```

### `<ticket>-report.md`

A human-readable session report. Generated every sync, overwrites any previous version. Filename is the ticket ID in lowercase: `bas-10-report.md`. If no ticket, `session-report.md`.

Contains the same content as the terminal output from Phase 8, formatted as markdown. Useful as a PR description, async standup update, or handoff document.

### `.drift/sessions.claude.json` / `.drift/sessions.cursor.json`

Structured JSON history of every analysis run. See [Data Model](#data-model) for the schema.

---

## Jira Integration

### What it fetches

On `drift-start` (if Jira is configured):
- Ticket `summary`, `description`, `status`
- Acceptance criteria — found in this order:
  1. A custom field containing "acceptance" in its name
  2. A checklist or bullet list within the description
  3. Subtask summaries if no AC field or description checklist exists

This becomes the `plannedWork` if you didn't provide your own.

On `drift-sync`:
- Same fetch, used as the authoritative AC checklist for Phase 4

### What it posts

After Phase 4, posts a structured comment to the ticket:

```
Drift Session Summary (Claude analysis):

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

The comment is posted using Atlassian Document Format (ADF), the same format used by the official drift CLI.

### Credential Resolution

Checked in order. First complete set of `url`, `email`, and `token` wins:

| Priority | Source | How to set |
|---|---|---|
| 1 | Shell env vars | `export JIRA_URL=...`, `JIRA_EMAIL=...`, `JIRA_TOKEN=...` |
| 2 | `.env` in project root | `JIRA_URL=https://...` etc., standard key=value format |
| 3 | `.drift/.env` | Same format, drift-specific, keeps creds out of your main `.env` |
| 4 | `~/.config/drift-nodejs/config.json` | Written by `drift config --jira-url ...` CLI command |
| 5 | `.drift/jira.json` | Written by Drift when you provide creds interactively |
| 6 | Interactive prompt | Drift asks during `drift-start` or `drift-sync` |

The `.env` parser handles: `export` prefix (`export JIRA_URL=...` → `JIRA_URL=...`), quoted values (`"https://..."` → `https://...`), inline comments (`https://foo.net # prod` → `https://foo.net`), and trailing whitespace.

---

## Data Model

### `.drift/config.json`

```json
{
  "initialized": true,
  "sessions": [
    {
      "startedAt": "2025-02-14T16:47:00.000Z",
      "startSha": "a1b2c3d4e5f67890abcdef1234567890abcdef12",
      "ticket": "BAS-10",
      "plannedWork": "implement JWT authentication"
    }
  ]
}
```

- `initialized` — boolean, true once `.drift/` has been set up
- `sessions` — array, capped at 20 entries, oldest trimmed from front
- Active session is always the **last element** of the array

### `.drift/sessions.claude.json` / `.drift/sessions.cursor.json`

```json
[
  {
    "date": "2025-02-14T16:47:00.000Z",
    "ticket": "BAS-10",
    "startSha": "a1b2c3d4...",
    "endSha": "f4e5d6c7...",
    "summary": "Implemented JWT authentication using python-jose with HS256 signing...",
    "decisions": [
      "HS256 over RS256 — SOUND: simpler key management for single service",
      "Token in Authorization header — ACCEPTABLE: appropriate for API clients"
    ],
    "drift": [
      "Added: password reset endpoint (not in AC)",
      "Missing: token refresh (AC item #4)"
    ],
    "architectureImpact": [
      "New middleware.py for JWT validation on all authenticated routes",
      "python-jose added to requirements.txt"
    ],
    "forNextSession": [
      "Implement token refresh endpoint (BAS-10 AC item #4)",
      "Add rate limiting to /login",
      "Move password reset to separate ticket"
    ],
    "ticketComment": "Drift Session Summary (Claude analysis):\n\nProgress: ~70%...",
    "codeReview": {
      "findings": [
        {
          "signal": "CONCERN",
          "category": "security",
          "file": "backend/config.py",
          "finding": "JWT secret has hardcoded fallback",
          "recommendation": "Remove fallback, raise on missing env var"
        },
        {
          "signal": "ISSUE",
          "category": "security",
          "file": "backend/router.py",
          "finding": "Login endpoint has no rate limiting",
          "recommendation": "Add slowapi or equivalent before deploy"
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
]
```

Capped at 50 entries. Newest entries at the front of the array.

### `.drift/jira.json`

```json
{
  "url": "https://yourteam.atlassian.net",
  "email": "you@company.com",
  "token": "your-api-token"
}
```

Written when credentials are provided interactively. Never committed (`.drift/` is gitignored).

---

## Scoping Rules

The scoping rules exist to prevent false positives. Without them, every analysis would be full of "you didn't write tests" and "this feature is incomplete" — findings based on what the reviewer thinks should exist, not what was actually agreed.

**Rule 1: Only evaluate against written AC**
Progress, drift, and next steps are measured only against items explicitly written in the ticket's acceptance criteria. If the AC says "add login endpoint" and the login endpoint exists, that item is done — even if the endpoint has no tests, no rate limiting, and no error handling. Those are separate concerns measured against separate requirements.

**Rule 2: If it's not in the AC, it's not missing**
If the AC doesn't mention tests, missing tests are not scope reductions. If the AC doesn't mention documentation, missing docs are not drift. This seems strict, but it's correct — the alternative is Drift inventing requirements that were never agreed upon.

**Rule 3: Scope creep is flagged, not judged**
Things you built that weren't in the AC are listed as scope additions. They're not marked as wrong — sometimes scope creep is good engineering. They're just surfaced so you can make a deliberate decision about them: ticket them out, PR them separately, or accept them consciously.

**Rule 4: Future ticket work is out of scope**
If you're on BAS-10 and you notice something that belongs to BAS-11, that's not drift and it's not missing. It's labeled "future work (out of scope)" and listed separately. The session is scoped to this ticket.

**Rule 5: No inferred requirements**
"Obviously you need error handling" is not a scoping rule. "The AC says 'handle invalid credentials with 401'" is. Drift reasons from what's written, not from what's implied.

---

## Flags & Options

### `drift-sync --no-push`
Runs the full analysis — all 8 phases — but skips posting the Jira comment in Phase 7. CONTEXT.md is still written, the report file is still written, session history is still saved. Use when you want the analysis but don't want a half-finished session appearing in Jira.

### `drift-sync --no-jira`
Skips all Jira integration entirely — no credential resolution, no ticket fetch in Phase 1b, no comment in Phase 7. The analysis uses `plannedWork` from config instead of ticket AC. Use when working without Jira or when tickets aren't set up.

---

## File Structure

```
<repo root>/
├── CONTEXT.md                   # Living context document (written by drift-sync)
├── bas-10-report.md             # Session report, lowercase ticket ID (written by drift-sync)
│
└── .drift/                      # Auto-gitignored, never committed
    ├── config.json              # Session state — initialized flag + sessions array (max 20)
    ├── sessions.json            # CLI tool session history
    ├── sessions.claude.json     # Claude Code analysis history (max 50 entries)
    ├── sessions.cursor.json     # Cursor analysis history (max 50 entries)
    ├── jira.json                # Jira credentials if saved interactively
    └── .env                     # Optional drift-specific env vars (JIRA_URL etc.)
```

**gitignore handling:**
On first `drift-start`, `.drift/` is added to `.gitignore` automatically. If `.gitignore` already contains `.drift`, nothing changes. If `.gitignore` doesn't exist, it's created.

**CONTEXT.md and report files** are in the repo root and are **not** gitignored. They're meant to be committed and shared. They're the primary artifact of a drift session — the persistent context that survives laptop closes, team handoffs, and AI memory resets.

---

## Installation

### Claude Code

Place the skill files at `~/.claude/skills/`:

```
~/.claude/skills/
  drift-start/SKILL.md
  drift-sync/SKILL.md
  drift-status/SKILL.md
```

Skills are global — they work in any git repository opened with Claude Code. Invoked as slash commands:

```
/drift-start BAS-10 implement auth
/drift-status
/drift-sync
/drift-sync --no-jira
```

### Cursor

Place the rule files at `~/.cursor/rules/`:

```
~/.cursor/rules/
  drift-start.mdc
  drift-sync.mdc
  drift-status.mdc
```

Rules are global — they work in any git repository opened with Cursor. Invoked by typing the rule name in agent mode:

```
drift-start BAS-10 implement auth
drift-status
drift-sync
drift-sync --no-jira
```

Cursor matches rules by their `description` field and activates them when they're relevant to the message.

---

## Tool Compatibility

Both the Claude Code and Cursor versions of Drift are designed to coexist on the same repository. The shared layer and the separate layer:

| File | Shared? | Notes |
|---|---|---|
| `.drift/config.json` | Yes | Both tools read and write session state |
| `.drift/sessions.json` | Yes | CLI tool history, both display it |
| `.drift/jira.json` | Yes | Credentials work in both tools |
| `.drift/.env` | Yes | Env vars work in both tools |
| `CONTEXT.md` | Yes | Overwritten by whichever tool ran sync last |
| `<ticket>-report.md` | Yes | Overwritten by whichever tool ran sync last |
| `sessions.claude.json` | Claude Code only | Claude Code writes and reads this |
| `sessions.cursor.json` | Cursor only | Cursor writes and reads this |

`drift-status` in either tool shows session history from all three sources (Cursor, Claude Code, CLI) so the full picture is always visible regardless of which tool you use.

---

## Edge Cases

### No changes since session start
`drift-sync` detects this in Phase 2 and stops without writing any files. Reports "No changes since session start."

### Binary files
Detected via `git diff --numstat` (binary files show `-` for add/delete). Skipped in Phase 3 file reading. Noted in the summary as "binary file, not analyzed."

### Files over 1000 lines
Skipped in Phase 3 file reading. Noted as "too large for inline analysis." The diff is still analyzed, just not the full file content.

### More than 100 changed files
Phase 3 runs `git diff --numstat` to get per-file line counts and reads only the top 20 files by total lines changed. The remaining files are noted in the summary.

### Missing startSha
If the `startSha` no longer exists in git history (e.g., after a rebase or force push), Phase 2 falls back to `git merge-base HEAD <startSha>`. If that also fails, tells the user explicitly and stops.

### No plannedWork
The "Drift from plan" section is omitted entirely from all outputs — not included with "N/A" or empty content. No planned work means no drift to measure.

### Jira fetch failure
If the Jira API returns a non-200 response or a network error in Phase 1b, warns the user and continues the sync without ticket context. The analysis falls back to `plannedWork`.

### Jira comment failure
If posting the comment in Phase 7 fails, warns the user but does not stop the sync. CONTEXT.md and the report file are still written. `jiraCommentPosted` is set to `false` in the session JSON.

### Old CONTEXT.claude.md (migration)
If `CONTEXT.md` doesn't exist but `CONTEXT.claude.md` does (from older sessions before the filename was standardized), Phase 5 renames `CONTEXT.claude.md` to `CONTEXT.md` using `mv` before proceeding. Previous session history is preserved.

### Active session already exists when running drift-start
Shows the existing session details and asks if you want to overwrite. If yes, removes the last session from the array and proceeds with the new one. If no, stops without changes.

---

## Design Decisions

### Why three separate commands instead of one

`drift-start` needs to be fast and non-intrusive — running it at the start of a session should feel like no overhead. `drift-status` needs to be read-only so it can be run anytime without risk. `drift-sync` is expensive (reads many files, calls Jira) and should be deliberate. Combining them would mean every status check triggers a Jira call, or every sync creates a session. Separation keeps the commands predictable.

### Why sessions are capped at 20 in config.json

`config.json` is the active session store, not the archive. The complete history lives in `sessions.claude.json` and `sessions.cursor.json` (capped at 50). Keeping only 20 sessions in config.json prevents the file from growing to megabytes after months of use, while still giving `drift-status` enough history to show a meaningful recent activity view.

### Why CONTEXT.md stays in the repo root (not .drift/)

`.drift/` is gitignored. `CONTEXT.md` is meant to be committed and shared. When a teammate clones the repo, or when a new AI conversation starts, `CONTEXT.md` is the first thing they should read. Hiding it in `.drift/` would defeat its purpose as a persistent context layer.

### Why the analysis is scoped strictly to the ticket's AC

The alternative — evaluating against "what a complete implementation would look like" — produces noise. Drift's job isn't to review the entire codebase or set requirements. It's to compare what you said you'd do against what you did. Strict AC scoping means the findings are always grounded in something explicitly agreed upon, never in assumptions.

### Why drift-status writes nothing

Trust. If `drift-status` had side effects — even benign ones — developers would hesitate to run it frequently. Making it provably read-only (no file writes, no API calls) means it can be run at any time, including in the middle of a fragile session, without risk.

### Why CLEAN findings are included in code review

Purely negative reviews are harder to trust. If every finding is a problem, reviewers start to discount the signal. Explicitly acknowledging good practices does two things: it confirms that the analysis looked at the full picture, and it rewards the patterns worth repeating.
