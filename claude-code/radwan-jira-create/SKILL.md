---
name: radwan-jira-create
description: Create Jira tickets and epics from any format — markdown, lists, tables, or plain text
allowed-tools: Bash, Read, Write, Glob
user-invocable: true
argument-prompt: "Paste or describe the tickets to create. Any format works — markdown, bullet lists, tables, or plain text."
---

# radwan-jira-create

Create Jira issues (epics, stories, tasks, bugs) from free-form input. The input can be any format — structured markdown, bullet lists, tables, JSON, or plain English. You are an LLM — parse and understand the intent.

## Instructions

When the user invokes `/radwan-jira-create`, follow these steps:

### Step 1: Resolve Jira credentials

Check these sources **in order**. Use the first complete set (url, email, token):

1. **Shell env vars**: `$JIRA_URL`, `$JIRA_EMAIL`, `$JIRA_TOKEN`
2. **`.env`** in project root — parse as key=value (handle `export` prefix, quotes, inline comments)
3. **`.drift/.env`** — same parsing rules
4. **`.drift/jira.json`** — `{"url", "email", "token"}`

If no credentials found, ask the user to provide them. Offer to save to `.drift/jira.json`.

### Step 2: Resolve the Jira project

Check if `$ARGUMENTS` or the user's message specifies a project key. If not:

1. Check `.drift/jira-config.json` for a saved `projectKey`
2. Fetch all projects the user has access to:
   ```bash
   curl -s -u "<email>:<token>" "<url>/rest/api/3/project"
   ```
3. List them and ask the user to pick one
4. Save the choice to `.drift/jira-config.json`: `{"projectKey": "XX"}`

### Step 3: Discover project capabilities

Fetch the project's issue types and available fields once:

```bash
curl -s -u "<email>:<token>" "<url>/rest/api/3/project/<key>"
```

Extract available issue type names and IDs (Epic, Story, Task, Bug, Subtask, etc.).

For field availability, check editmeta on any existing issue or use:
```bash
curl -s -u "<email>:<token>" "<url>/rest/api/3/issue/createmeta/<key>/issuetypes/<typeId>"
```

Key fields to detect:
- `timetracking` → use `{"originalEstimate": "2h"}` for estimates
- `timeoriginalestimate` → use seconds (3600 = 1h) — fallback if timetracking fails
- `customfield_10016` → story points — last resort for estimates
- `labels` → for tagging
- `description` → always ADF format for API v3

### Step 4: Parse the user's input

Parse `$ARGUMENTS` to extract tickets. **Any format is valid.** Use your understanding to identify:

- **Epics** — grouping containers. Look for headers, "Epic" labels, or hierarchical structure
- **Tickets** — individual work items. Extract:
  - **Title** (required)
  - **Type** — Story, Task, Bug, QA, Spike, Optimization, etc. Map unsupported types to Task
  - **Estimate** — any time format: "2h", "1d", "3 days", "half day", "TBD". Convert to Jira format (Xh, Xd)
  - **Description/Summary** — any additional context
  - **Acceptance criteria** — bullet points, numbered lists, checkboxes
  - **Dependencies** — references to other tickets
  - **Labels/Tags** — any categorization hints
  - **Assignee** — if a person is mentioned
  - **Status** — if marked as done/complete, transition after creation
- **Parent/child relationships** — tickets nested under epics, indented items, etc.

Examples of formats you should handle:

**Markdown with headers:**
```
## Auth Epic
- Login endpoint (Story, 2h)
- Password reset (Task, 4h)
  - AC: reset email sent within 30s
  - AC: token expires after 1h
```

**Plain text:**
```
Create 3 tickets under the Auth epic:
1. Build login page - story, about 2 hours
2. Add OAuth support - task, 1 day, depends on login page
3. Write auth tests - QA task, half day
```

**Table:**
```
| Title | Type | Estimate |
| Login | Story | 2h |
| OAuth | Task | 1d |
```

**Terse:**
```
Epic: Payments
- Stripe integration 4h story
- Webhook handler 2h task
- Payment tests 1d QA
```

### Step 5: Confirm before creating

Show the user a summary of what will be created:

```
Will create in <PROJECT_KEY>:

Epic: Auth Epic
  1. Login endpoint (Story, 2h)
  2. Password reset (Task, 4h) — 2 AC items
  3. Auth tests (Task, 4h)

Total: 1 epic + 3 tickets

Proceed? (y/n)
```

If the user made any ambiguous choices, call them out. For example:
- "No type specified for 'Webhook handler' — defaulting to Task"
- "QA mapped to Task (not available as issue type)"
- "No estimate for 'Auth tests' — will skip estimate field"

### Step 6: Create issues via API

Create in dependency order: **epics first**, then child tickets.

**Create an issue:**
```bash
curl -s -X POST -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '<payload>' \
  "<url>/rest/api/3/issue"
```

**Payload structure:**
```json
{
  "fields": {
    "project": {"key": "<PROJECT_KEY>"},
    "summary": "<title>",
    "issuetype": {"id": "<type_id>"},
    "parent": {"key": "<epic_key>"},
    "description": <ADF document>,
    "labels": ["<label>"]
  }
}
```

**Build ADF description** from AC, summary, and dependencies:
- Summary text → paragraph
- Acceptance criteria → heading + bulletList
- Dependencies → heading + paragraph

**Set estimates** after creation (separate PUT call):
```bash
curl -s -X PUT -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"fields":{"timetracking":{"originalEstimate":"<estimate>"}}}' \
  "<url>/rest/api/3/issue/<key>"
```

If `timetracking` fails (400), fall back to `customfield_10016` (story points) with hours as the value.

**Assign** if an assignee was specified:
```bash
curl -s -X PUT -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"accountId":"<account_id>"}' \
  "<url>/rest/api/3/issue/<key>/assignee"
```

To resolve a name to an accountId:
```bash
curl -s -u "<email>:<token>" "<url>/rest/api/3/user/search?query=<name>"
```

**Transition done tickets** after creation:
```bash
# Get transitions
curl -s -u "<email>:<token>" "<url>/rest/api/3/issue/<key>/transitions"
# Find "Done" transition ID, then:
curl -s -X POST -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"transition":{"id":"<done_id>"}}' \
  "<url>/rest/api/3/issue/<key>/transitions"
```

### Step 7: Report results

Show a clear summary:

```
Created 1 epic + 3 tickets:

Epic: PROJ-10 — Auth Epic
  PROJ-11 — Login endpoint (Story, 2h)
  PROJ-12 — Password reset (Task, 4h)
  PROJ-13 — Auth tests (Task, 4h) → Done

All estimates set. 0 failures.
```

If any failed, show the error and the ticket that failed so the user can fix it.

---

## Jira API Reference

### Issue types
Common types and typical mappings:
- Epic, Story, Task, Bug — usually available
- QA, Spike, Optimization — usually not available, map to Task
- Subtask — available but requires a parent task (not epic)

### ADF (Atlassian Document Format)
API v3 requires ADF for description fields:
```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {"type": "paragraph", "content": [{"type": "text", "text": "Summary text"}]},
    {"type": "heading", "attrs": {"level": 3}, "content": [{"type": "text", "text": "Acceptance Criteria"}]},
    {"type": "bulletList", "content": [
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "AC item"}]}]}
    ]}
  ]
}
```

### Search API (new endpoint — old /search is deprecated)
```bash
curl -s -X POST -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"jql":"project=<KEY>","maxResults":100,"fields":["summary"]}' \
  "<url>/rest/api/3/search/jql"
```

### Estimate field priority
1. `timetracking.originalEstimate` — preferred ("2h", "1d")
2. `customfield_10016` — story points (number) — fallback

### Time format
- Hours: "1h", "2h", "4h", "6h"
- Days: "1d" (= 8h), "2d", "3d", "4d"
- Jira default: 1d = 8h, 1w = 5d
