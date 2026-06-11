---
name: radwan-jira-assign
description: Assign Jira tickets to people from natural language. Say which tickets and who to assign them to.
---

# radwan-jira-assign

Assign Jira tickets to team members from free-form input. Understands natural language, ticket keys, names, and bulk operations.

## Instructions

### Step 1: Resolve Jira credentials

Check in order: shell env vars (`JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN`) → `.env` → `.drift/.env` → `.drift/jira.json`.

### Step 2: Parse the input

**Any format is valid:**
- `JH-47, JH-21 to fares` — direct keys
- `assign the TV scraper tickets to fares` — by description
- `all QA tickets to radwan` — by type
- `unassign JH-15` — remove assignee

### Step 3: Resolve tickets and people

Search for tickets by name if no key given:
```bash
curl -s -X POST -u "<email>:<token>" -H "Content-Type: application/json" \
  -d '{"jql":"project=<KEY> AND summary ~ \"<text>\"","fields":["summary","assignee"]}' \
  "<url>/rest/api/3/search/jql"
```

Resolve names: `curl -s -u "<email>:<token>" "<url>/rest/api/3/user/search?query=<name>"`

### Step 4: Confirm, then assign

```bash
curl -s -X PUT -u "<email>:<token>" -H "Content-Type: application/json" \
  -d '{"accountId":"<id>"}' "<url>/rest/api/3/issue/<key>/assignee"
```

Unassign: `{"accountId":null}`. If "cannot be assigned" → user needs project Member role.

### Step 5: Report results
