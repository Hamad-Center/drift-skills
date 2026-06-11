---
name: radwan-jira-create
description: Create Jira tickets and epics from any format — markdown, lists, tables, or plain text. Provide the tickets to create in any format.
---

# radwan-jira-create

Create Jira issues (epics, stories, tasks, bugs) from free-form input. The input can be any format — structured markdown, bullet lists, tables, JSON, or plain English. You are an LLM — parse and understand the intent.

## Instructions

When the user invokes `radwan-jira-create`, follow these steps:

### Step 1: Resolve Jira credentials

Check in order: shell env vars (`JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN`) → `.env` → `.drift/.env` → `.drift/jira.json`. Use first complete set. If none found, ask the user.

### Step 2: Resolve the Jira project

Check `.drift/jira-config.json` for `projectKey`. If not found, fetch projects and ask. Save choice.

### Step 3: Discover project capabilities

Fetch issue types from `<url>/rest/api/3/project/<key>`. Map unsupported types (QA, Spike, Optimization) to Task.

Detect estimate fields: try `timetracking.originalEstimate` first, fall back to `customfield_10016`.

### Step 4: Parse the user's input

**Any format is valid.** Extract:
- Epics (grouping containers)
- Tickets: title, type, estimate, description, acceptance criteria, dependencies, labels, assignee, status
- Parent/child relationships

### Step 5: Confirm before creating

Show summary and ask for confirmation. Call out ambiguous choices.

### Step 6: Create via API

Create epics first, then children. Use:
- `POST <url>/rest/api/3/issue` — create
- `PUT <url>/rest/api/3/issue/<key>` — set estimates via `timetracking.originalEstimate`
- `PUT <url>/rest/api/3/issue/<key>/assignee` — assign (resolve names via `/user/search?query=<name>`)
- `POST <url>/rest/api/3/issue/<key>/transitions` — transition done tickets

Descriptions use ADF format (API v3). Build from summary + AC + dependencies.

### Step 7: Report results

Show created tickets with Jira keys, titles, and any failures.

---

## API Reference

- **Search:** `POST <url>/rest/api/3/search/jql` with `{"jql":"...","fields":[...]}`
- **ADF format** required for descriptions
- **Time format:** "1h", "2h", "1d" (1d = 8h)
- **Estimate priority:** timetracking.originalEstimate → customfield_10016
