---
name: drift-start
description: Start a drift session — record current git SHA and session metadata to .drift/config.json
allowed-tools: Bash, Read, Write, Glob
user-invocable: true
argument-prompt: "Provide a ticket ID and/or description of planned work (e.g., PROJ-123 implement user auth)"
---

# drift-start

Record the current git state and session metadata to begin tracking how code drifts from plan.

## Instructions

When the user invokes `/drift-start`, follow these steps exactly:

### Step 1: Verify git repository

Run `git rev-parse --git-dir` to confirm this is a git repository. If not, tell the user and stop.

### Step 2: Initialize .drift/ directory

Check if `.drift/config.json` exists. If not:
- Create the `.drift/` directory
- Write `.drift/config.json` with: `{"initialized": true, "sessions": []}`
- Write `.drift/sessions.json` with content: `[]`
- Handle `.gitignore`:
  - If `.gitignore` exists and already contains `.drift` — do nothing
  - If `.gitignore` exists but does NOT contain `.drift` — append `\n# drift session data\n.drift/\n` to the file
  - If `.gitignore` does not exist — create it with content: `# drift session data\n.drift/\n`

### Step 3: Check for active session

Read `.drift/config.json` and check if the `sessions` array has any elements. The active session is the **last element** of the array.

If an active session exists:
- Show the user the existing session details (started at, SHA, ticket, planned work)
- Ask if they want to overwrite it (which will pop the last session and push a new one)
- If they decline, stop

### Step 4: Parse arguments from $ARGUMENTS

Parse `$ARGUMENTS` to extract:
- **Ticket ID**: Match the pattern `[A-Za-z][A-Za-z0-9]+-\d+` (case-insensitive — e.g., PROJ-123, proj-123, Ab-1). Normalize to UPPERCASE. **Required** — if no ticket ID is found in `$ARGUMENTS`, ask the user to provide one before continuing. They cannot skip this.
- **Planned work**: Everything in `$ARGUMENTS` that isn't the ticket ID. Trim whitespace. This is optional.

Examples:
- `PROJ-123 implement user authentication` → ticket=`PROJ-123`, plannedWork=`implement user authentication`
- `proj-123 implement user authentication` → ticket=`PROJ-123`, plannedWork=`implement user authentication`
- `implement user authentication` → no ticket found → ask user for ticket ID
- `PROJ-123` → ticket=`PROJ-123`, plannedWork=null
- (empty) → no ticket found → ask user for ticket ID

### Step 5: Get git state

Run these commands:
- `git rev-parse HEAD` → startSha
- `git rev-parse --abbrev-ref HEAD` → branch

### Step 6: Write session to config

Read `.drift/config.json`, then push a new session onto the `sessions` array:

```json
{
  "startedAt": "<ISO 8601 timestamp>",
  "startSha": "<full SHA>",
  "ticket": "<ticket ID or omit if none>",
  "plannedWork": "<planned work text or omit if none>"
}
```

The `SessionState` interface fields are: `startedAt` (string), `startSha` (string), `ticket` (optional string), `plannedWork` (optional string). Do not add extra fields.

Write the updated config back to `.drift/config.json` with 2-space indentation.

Keep a maximum of 20 sessions in the array. If the array exceeds 20 entries after pushing, remove the oldest entries (from the beginning of the array) to bring it back to 20. This prevents config.json from growing unbounded.

### Step 6b: Fetch Jira ticket as plannedWork fallback

If ALL of the following are true:
- A `ticket` was set in Step 4
- No `plannedWork` was parsed from `$ARGUMENTS`
- Jira credentials are available (using the resolution order in the **Jira Credential Resolution** section below)

Then fetch the Jira ticket:
```bash
curl -s -u "<email>:<token>" -H "Content-Type: application/json" "<url>/rest/api/3/issue/<ticket>"
```

Extract `summary` and `description` (plain text from the ADF body). Set `plannedWork` to `"<summary>\n<description>"` and update the session in `.drift/config.json`.

This matches the CLI behavior: when no planned work text is provided but a ticket exists and Jira is configured, the ticket's summary+description becomes the plannedWork.

If the fetch fails (non-200 status, network error, etc.), continue without — plannedWork stays null.

### Step 7: Report to user

Display a clear summary:
```
Session started
  Branch: <branch>
  SHA:    <short SHA (first 7 chars)>
  Ticket: <ticket or "none">
  Plan:   <planned work or "none">
```

If a ticket was provided, check whether Jira credentials are available using the resolution order defined in the **Jira Credential Resolution** section below.

If Jira is configured, mention that `/drift-sync` will fetch ticket context and post a summary comment.

If Jira is NOT configured, ask the user if they want to set it up now:
> Would you like to configure Jira integration for this project?
> When configured, `/drift-sync` will fetch ticket context and post session summaries as comments.
>
> I'll need:
> 1. Jira URL (e.g., https://yourteam.atlassian.net)
> 2. Jira email
> 3. Jira API token (from https://id.atlassian.com/manage-profile/security/api-tokens)

If the user provides credentials, save them to `.drift/jira.json`:
```json
{"url": "<url>", "email": "<email>", "token": "<token>"}
```
If the user declines, that's fine — continue without Jira.

---

## Jira Credential Resolution

When Jira credentials are needed, check these sources **in order**. Use the first source that provides all three values (url, email, token). Stop checking once a complete set is found.

### 1. Environment variables (shell)
```bash
echo "$JIRA_URL"; echo "$JIRA_EMAIL"; echo "$JIRA_TOKEN"
```
Use these if all three are set and non-empty in the current shell environment.

### 2. `.env` file (project root)
Read `.env` in the repo root. Look for `JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN`.

Parse `.env` as key=value pairs with these rules:
- Ignore blank lines and lines starting with `#`
- Strip optional `export ` prefix (e.g., `export JIRA_URL=...` → `JIRA_URL=...`)
- Strip surrounding quotes from values (single or double: `JIRA_URL="https://..."` → `https://...`)
- Trim trailing whitespace from values
- Ignore inline comments after values (e.g., `JIRA_URL=https://foo.net # prod` → `https://foo.net`)

Use if all three keys are present and non-empty.

### 3. `.drift/.env` file (drift-specific)
Same format and parsing rules as source 2 above, but read from `.drift/.env`. This keeps Jira creds out of the project's main `.env` file.

### 4. Drift CLI global config
Read `~/.config/drift-nodejs/config.json` if it exists. Keys are `jiraUrl`, `jiraEmail`, `jiraToken`.
This is where `drift config --jira-url/--jira-email/--jira-token` saves credentials.

### 5. `.drift/jira.json` (local project config)
Read `.drift/jira.json` if it exists. Format: `{"url": "...", "email": "...", "token": "..."}`.
This is where the skill saves credentials when the user provides them interactively.

### 6. Ask the user
If none of the above sources have complete credentials, ask the user. If they provide them, save to `.drift/jira.json`. If they decline, skip Jira.

Remind the user they can run `/drift-status` to check progress and `/drift-sync` to analyze when done.
