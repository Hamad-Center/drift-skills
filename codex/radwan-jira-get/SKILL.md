---
name: radwan-jira-get
description: Search, read, and list Jira tickets — backlog, sprints, epics, or specific issues. Ask for what you want to see.
---

# radwan-jira-get

Read and search Jira tickets. View individual tickets, list backlogs, browse epics, search by text, filter by status/type/assignee, and show sprints.

## Instructions

### Step 1: Resolve Jira credentials

Check in order: shell env vars (`JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN`) → `.env` → `.drift/.env` → `.drift/jira.json`.

### Step 2: Parse the query

**Any format is valid:**
- `JH-47` — single ticket
- `show backlog` — all non-sprint tickets
- `EPIC 1 tickets` — epic children
- `QA tickets` — filter by type
- `assigned to fares` — filter by assignee
- `what's done?` — by status
- `project overview` — stats

### Step 3: Execute queries

**Single ticket:**
```bash
curl -s -u "<email>:<token>" "<url>/rest/api/3/issue/<KEY>?fields=summary,status,assignee,issuetype,parent,description,timetracking,labels"
```

**Search via JQL:**
```bash
curl -s -X POST -u "<email>:<token>" -H "Content-Type: application/json" \
  -d '{"jql":"<query>","maxResults":100,"fields":["summary","status","assignee","issuetype","parent","timetracking","labels"]}' \
  "<url>/rest/api/3/search/jql"
```

Common JQL:
- Backlog: `project=<KEY> AND sprint is EMPTY AND status != Done`
- Epic children: `project=<KEY> AND parent=<EPIC_KEY>`
- By type/assignee/status: standard JQL filters
- Text search: `summary ~ "<text>"`
- Current sprint: `sprint in openSprints()`

**Board/sprint:**
```bash
curl -s -u "<email>:<token>" "<url>/rest/agile/1.0/board?projectKeyOrId=<KEY>"
```

### Step 4: Format results

- Single ticket → full detail
- List → compact table grouped by epic
- Stats → counts by status, type, assignee

### Step 5: Suggest follow-ups

Point to `radwan-jira-assign` and `radwan-jira-create` where relevant.
