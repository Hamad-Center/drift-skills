---
name: drift-status
description: Quick read-only status check of the active drift session
allowed-tools: Bash, Read, Glob
user-invocable: true
---

# drift-status

Show a quick, read-only status of the active drift session — what's changed since it started.

## Instructions

This skill performs NO file writes. It only reads state and displays it.

### Step 1: Read session state

Read `.drift/config.json`. If it doesn't exist or `initialized` is false, tell the user:
```
No drift session found. Start one with /drift-start or drift start
```
And stop.

Get the active session — the **last element** of the `sessions` array. If the array is empty, tell the user there's no active session and stop.

### Step 2: Get change stats

Using the `startSha` from the active session, run these commands:

```bash
git log --oneline <startSha>..HEAD
```
→ Commits since session start

```bash
git diff --stat <startSha>
```
→ Diff statistics

```bash
git diff --name-only <startSha>
```
→ Changed files list

```bash
git rev-parse --abbrev-ref HEAD
```
→ Current branch

```bash
git rev-parse --short HEAD
```
→ Current SHA (short)

Also check for uncommitted changes:
```bash
git status --porcelain
```

### Step 3: Display status

Format the output clearly:

```
Drift session active

  Started:  <startedAt timestamp>
  Branch:   <current branch>
  SHA:      <startSha short> → <current SHA short>
  Ticket:   <ticket or "none">
  Plan:     <plannedWork or "none">

Changes since session start:
  Commits:       <N>
  Files changed: <N>

<diff stat output>

Changed files:
  <file 1>
  <file 2>
  ...

Uncommitted: <"yes - N files" or "none">
```

If there are no changes at all, say:
```
No changes since session start.
```

### Step 3b: Recent session history

If `.drift/sessions.claude.json` exists, read it and show the last 5 entries:

```
Recent sessions (Claude):
  1. <date> — <ticket or "no ticket"> — <summary truncated to 80 chars>
  2. ...
```

If `.drift/sessions.json` also exists, read it and show the last 5 entries:

```
Recent sessions (CLI):
  1. <date> — <ticket or "no ticket"> — <summary truncated to 80 chars>
  2. ...
```

If neither file exists or both are empty, omit this section.

### Step 4: Suggest next steps

At the end, remind the user:
- `/drift-sync` — analyze changes and write CONTEXT.md
- The last sync report is in `<ticket>-report.md` (or `session-report.md`) if it exists
