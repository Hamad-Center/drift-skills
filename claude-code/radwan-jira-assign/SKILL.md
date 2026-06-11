---
name: radwan-jira-assign
description: Assign Jira tickets to people from natural language
allowed-tools: Bash, Read, Write, Glob
user-invocable: true
argument-prompt: "Tell me which tickets to assign and to whom. Any format works."
---

# radwan-jira-assign

Assign Jira tickets to team members from free-form input. Understands natural language, ticket keys, names, and bulk operations.

## Instructions

When the user invokes `/radwan-jira-assign`, follow these steps:

### Step 1: Resolve Jira credentials

Check these sources **in order**. Use the first complete set (url, email, token):

1. **Shell env vars**: `$JIRA_URL`, `$JIRA_EMAIL`, `$JIRA_TOKEN`
2. **`.env`** in project root — parse as key=value (handle `export` prefix, quotes, inline comments)
3. **`.drift/.env`** — same parsing rules
4. **`.drift/jira.json`** — `{"url", "email", "token"}`

If no credentials found, ask the user.

### Step 2: Resolve the Jira project

Check `.drift/jira-config.json` for a saved `projectKey`. If not found:
- Try to infer from ticket keys in the input (e.g., "JH-47" → project "JH")
- If still unknown, fetch projects and ask

### Step 3: Parse the input

Parse `$ARGUMENTS` to extract assignment instructions. **Any format is valid:**

**Direct:**
```
JH-47, JH-21, JH-22 to fares
```

**Natural language:**
```
assign the TV scraper tickets to fares and the gemini ticket to radwan
```

**Bulk:**
```
assign all unassigned tickets in EPIC 1 to radwan
```

**Mixed:**
```
JH-10 to ahmed
the clustering tickets to fares
unassign JH-15
```

**By reference (not just ticket keys):**
```
assign the language bug ticket to radwan
assign all QA tickets to fares
```

Extract:
- **Tickets** — by key (JH-47), by title/description, by epic, by type, or by filter
- **Assignee** — by name. Resolve to Jira accountId via user search
- **Unassign** — if the user says "unassign", set assignee to null

### Step 4: Resolve ticket keys if needed

If the user references tickets by name/description rather than key, search for them:

```bash
curl -s -X POST -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"jql":"project=<KEY> AND summary ~ \"<search>\"","maxResults":10,"fields":["summary","assignee","parent"]}' \
  "<url>/rest/api/3/search/jql"
```

For epic-based queries:
```bash
curl -s -X POST -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"jql":"project=<KEY> AND parent=<EPIC_KEY>","maxResults":100,"fields":["summary","assignee"]}' \
  "<url>/rest/api/3/search/jql"
```

### Step 5: Resolve assignee names to account IDs

```bash
curl -s -u "<email>:<token>" "<url>/rest/api/3/user/search?query=<name>"
```

If multiple matches, show the options and ask the user to pick. Cache resolved names for the session.

### Step 6: Confirm before assigning

Show what will change:

```
Will assign:
  JH-47 (Incremental Clustering) → fares
  JH-21 (TV Scraper — Core) → fares
  JH-22 (TV Scraper — S3) → fares

Proceed? (y/n)
```

### Step 7: Assign via API

```bash
curl -s -X PUT -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"accountId":"<account_id>"}' \
  "<url>/rest/api/3/issue/<key>/assignee"
```

To unassign:
```bash
curl -s -X PUT -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"accountId":null}' \
  "<url>/rest/api/3/issue/<key>/assignee"
```

If assignment fails with "cannot be assigned", the user is not a project member. Tell the user:
> <name> can't be assigned tickets — they need to be added as a Member in Project Settings → Access.

### Step 8: Report results

```
Assigned 3 tickets:
  JH-47 → fares ✓
  JH-21 → fares ✓
  JH-22 → fares ✓
```

---

## Jira API Reference

### User search
```bash
curl -s -u "<email>:<token>" "<url>/rest/api/3/user/search?query=<name>"
```
Returns array of users with `accountId`, `displayName`, `active`.

### Assignee endpoint
```bash
curl -s -X PUT -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"accountId":"<id>"}' \
  "<url>/rest/api/3/issue/<key>/assignee"
```
Returns 204 on success.

### Search API (new endpoint)
```bash
curl -s -X POST -u "<email>:<token>" \
  -H "Content-Type: application/json" \
  -d '{"jql":"<query>","maxResults":100,"fields":["summary","assignee"]}' \
  "<url>/rest/api/3/search/jql"
```
