---
name: radwan-jira-get
description: Search, read, and list Jira tickets — backlog, sprints, epics, or specific issues
allowed-tools: Bash, Read, Write, Glob
user-invocable: true
argument-prompt: "What do you want to see? A ticket, the backlog, sprint, epic, or search for something."
---

# radwan-jira-get

Read and search Jira tickets. Supports viewing individual tickets, listing backlogs, browsing epics, searching by text, filtering by status/type/assignee, and showing sprint boards.

## Instructions

When the user invokes `/radwan-jira-get`, follow these steps:

### Step 1: Resolve Jira credentials

Check these sources **in order**. Use the first complete set (url, email, token):

1. **Shell env vars**: `$JIRA_URL`, `$JIRA_EMAIL`, `$JIRA_TOKEN`
2. **`.env`** in project root — parse as key=value (handle `export` prefix, quotes, inline comments)
3. **`.drift/.env`** — same parsing rules
4. **`.drift/jira.json`** — `{"url", "email", "token"}`

If no credentials found, ask the user.

### Step 2: Resolve the Jira project

Check `.drift/jira-config.json` for a saved `projectKey`. If not found:
- Try to infer from ticket keys in the input
- If still unknown, fetch projects and ask

### Step 3: Parse the query

Parse `$ARGUMENTS` to understand what the user wants. **Any format is valid:**

**Specific ticket:**
```
JH-47
```
```
show me the clustering edge case ticket
```

**Backlog:**
```
show backlog
```
```
what's in the backlog?
```

**Epic view:**
```
show EPIC 1 tickets
```
```
what's under the processing unit epic?
```

**Search:**
```
find all QA tickets
```
```
search for "scraper"
```

**Filtered views:**
```
show all tickets assigned to fares
```
```
what's done?
```
```
unassigned tickets
```
```
all bugs
```

**Sprint:**
```
current sprint
```
```
what's in the sprint?
```

**Summary/stats:**
```
project overview
```
```
how many tickets are done?
```

### Step 4: Execute the appropriate query

**Single ticket:**
```bash
curl -s -u "<email>:<token>" \
  "<url>/rest/api/3/issue/<KEY>?fields=summary,status,assignee,issuetype,parent,description,timetracking,labels,created,updated"
```

**Search/list (use JQL):**
```bash
curl -s -X POST -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"jql":"<query>","maxResults":100,"fields":["summary","status","assignee","issuetype","parent","timetracking","labels"]}' \
  "<url>/rest/api/3/search/jql"
```

Common JQL patterns:
- Backlog: `project=<KEY> AND sprint is EMPTY AND status != Done ORDER BY rank`
- Epic children: `project=<KEY> AND parent=<EPIC_KEY> ORDER BY rank`
- By type: `project=<KEY> AND issuetype=Story`
- By assignee: `project=<KEY> AND assignee="<accountId>"`
- By status: `project=<KEY> AND status="Done"`
- Unassigned: `project=<KEY> AND assignee is EMPTY`
- Text search: `project=<KEY> AND summary ~ "<text>"`
- Current sprint: `project=<KEY> AND sprint in openSprints()`
- All epics: `project=<KEY> AND issuetype=Epic`

**Sprint info:**
```bash
curl -s -u "<email>:<token>" "<url>/rest/agile/1.0/board/<boardId>/sprint?state=active"
```

**Board discovery:**
```bash
curl -s -u "<email>:<token>" "<url>/rest/agile/1.0/board?projectKeyOrId=<KEY>"
```

### Step 5: Format and display results

Format results clearly for the user. Adapt the format to what was queried:

**Single ticket — full detail:**
```
JH-47 — Incremental Clustering: Edge Case Hardening
  Type:     Task
  Status:   To Do
  Assignee: fares
  Epic:     JH-44 (EPIC 7 — Clustering Service)
  Estimate: 4d
  Labels:   clustering

  Description:
  Harden incremental update mode for edge cases...

  Acceptance Criteria:
  - Policy: when clusters split, merge, or retire
  - Stale cluster detection after N days
  - Stress tested: 50k article burst
```

**List view — compact table:**
```
JH Project Backlog (53 tickets)

EPIC 1 — Processing Unit (8 tickets)
  Key    Title                                  Type   Est  Status   Assignee
  JH-3   Detect Breaking News                   Story  2h   To Do    —
  JH-4   QA: Breaking News Classifier            Task   2h   To Do    —
  JH-5   Processing Unit: Consume from Queue     Story  1d   To Do    —
  ...

EPIC 2A — Article Scraper (1 open, 5 done)
  JH-12  Article Scraper — Push to Queue         Task   1d   To Do    —
  ...
```

**Stats view:**
```
JH Project Overview
  Total: 60 tickets across 12 epics
  Done: 7 (12%)
  To Do: 53
  Assigned: 7 | Unassigned: 53
  Total estimated: 500h (62d 4h)
```

**Adjust columns** based on what's relevant. If no assignees are set, drop that column. If all tickets are the same type, drop the type column.

### Step 6: Suggest follow-ups

Based on what was shown, suggest relevant actions:
- After viewing backlog: "Use `/radwan-jira-assign` to assign tickets or `/radwan-jira-create` to add new ones"
- After viewing a ticket: "Use `/radwan-jira-assign JH-47 to fares` to assign it"
- After searching: "Found 5 matches. Want to see full details on any of these?"

---

## Jira API Reference

### Search API (new endpoint — old /search is deprecated)
```bash
curl -s -X POST -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"jql":"<query>","maxResults":100,"fields":["summary","status","assignee","issuetype","parent","timetracking","labels"]}' \
  "<url>/rest/api/3/search/jql"
```

### Single issue
```bash
curl -s -u "<email>:<token>" "<url>/rest/api/3/issue/<KEY>?fields=summary,status,assignee,issuetype,parent,description,timetracking,labels,created,updated"
```

### Transitions (to check available statuses)
```bash
curl -s -u "<email>:<token>" "<url>/rest/api/3/issue/<KEY>/transitions"
```

### Board and sprint
```bash
# Find board
curl -s -u "<email>:<token>" "<url>/rest/agile/1.0/board?projectKeyOrId=<KEY>"
# Active sprint
curl -s -u "<email>:<token>" "<url>/rest/agile/1.0/board/<boardId>/sprint?state=active"
# Sprint issues
curl -s -u "<email>:<token>" "<url>/rest/agile/1.0/sprint/<sprintId>/issue?fields=summary,status,assignee"
```

### ADF description parsing
API v3 returns descriptions in ADF format. To extract plain text, recursively walk the `content` array and concatenate all `text` nodes. For display purposes, convert:
- `bulletList` → markdown bullet points
- `heading` → markdown headers
- `paragraph` → plain text with newlines
