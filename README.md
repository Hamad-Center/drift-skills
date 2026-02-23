# Drift Skills

> AI-powered session tracking that connects your git changes to Jira tickets — available for Claude Code, Cursor, and Codex.

## What Is Drift

Drift tracks how your code drifts from plan during AI-assisted development. Three commands, zero workflow changes:

```
drift-start   →  snapshot your starting point + Jira ticket
drift-status  →  read-only progress check
drift-sync    →  deep analysis + code review + Jira comment
```

### What Problems Does It Solve

- **Scope creep** — flags everything you built that wasn't in the ticket's acceptance criteria
- **Forgotten AC items** — compares your code against the ticket's actual requirements, item by item
- **Context loss** — writes `CONTEXT.md` with decisions, drift, and next steps that survive across sessions
- **Hacky shortcuts** — reviews every changed file with CLEAN / CONCERN / ISSUE labels
- **Invisible progress** — auto-posts structured updates to Jira with progress %, quality summary, and next steps

## Quick Install

### Claude Code

```bash
cp -r claude-code/* ~/.claude/skills/
```

Invoke with slash commands:
```
/drift-start BAS-10 implement user auth
/drift-status
/drift-sync
```

### Cursor

```bash
cp cursor/* ~/.cursor/rules/
```

Invoke by typing in agent mode chat:
```
drift-start BAS-10 implement user auth
drift-status
drift-sync
```

### Codex

```bash
cp -r codex/* ~/.agents/skills/
```

Invoke via `/skills` or `$` mentions:
```
drift-start BAS-10 implement user auth
drift-status
drift-sync
```

## Repo Structure

```
drift-skills/
├── README.md                           # This file
│
├── claude-code/                        # Claude Code skills
│   ├── README.md                       # Claude Code specific docs
│   ├── drift-start/SKILL.md
│   ├── drift-sync/SKILL.md
│   └── drift-status/SKILL.md
│
├── cursor/                             # Cursor rules
│   ├── README.md                       # Cursor specific docs
│   ├── drift-start.mdc
│   ├── drift-sync.mdc
│   └── drift-status.mdc
│
├── codex/                              # Codex skills
│   ├── README.md                       # Codex specific docs
│   ├── drift-start/SKILL.md
│   ├── drift-sync/SKILL.md
│   └── drift-status/SKILL.md
│
└── docs/                               # Documentation
    ├── DRIFT_DOCS.md                   # Overview documentation
    ├── DRIFT_DOCS_DETAILED.md          # Full technical reference
    └── drift-pitch.md                  # Presentation / pitch deck
```

## Tool Compatibility

All three versions share the same `.drift/` directory and `CONTEXT.md`. You can switch between tools freely on the same project.

| What's shared | What's separate |
|---|---|
| `.drift/config.json` (session state) | `sessions.claude.json` (Claude Code history) |
| `.drift/jira.json` (credentials) | `sessions.cursor.json` (Cursor history) |
| `CONTEXT.md` (latest analysis) | `sessions.codex.json` (Codex history) |
| `<ticket>-report.md` (latest report) | |

## Output Files

After every `drift-sync`:

| File | Location | What it is |
|---|---|---|
| `CONTEXT.md` | repo root | Living context document — latest session + up to 4 previous sessions |
| `<ticket>-report.md` | repo root | Session report — overwritten every sync, ready for PR descriptions |
| `sessions.*.json` | `.drift/` | Structured JSON history — max 50 entries, machine-readable |
| Jira comment | posted via API | Progress %, what was built, drift, code quality, next steps |

## Jira Integration

Drift connects to Jira in two directions:

- **Inbound:** Fetches ticket summary, description, and acceptance criteria on `drift-start` and `drift-sync`
- **Outbound:** Posts a structured session summary as a Jira comment on `drift-sync`

Credentials are resolved in order: env vars → `.env` → `.drift/.env` → global config → `.drift/jira.json` → interactive prompt.

Use `drift-sync --no-jira` to skip all Jira integration.

## Flags

```
drift-sync --no-push    # run full analysis, skip posting the Jira comment
drift-sync --no-jira    # skip all Jira integration (no fetch, no post)
```

## Documentation

| Document | What it covers |
|---|---|
| [DRIFT_DOCS.md](docs/DRIFT_DOCS.md) | Overview — problems, how it works, output files, installation |
| [DRIFT_DOCS_DETAILED.md](docs/DRIFT_DOCS_DETAILED.md) | Full technical reference — every phase, data model, scoping rules, edge cases, design decisions |
| [drift-pitch.md](docs/drift-pitch.md) | Storytelling presentation — the vibe coder who pushes to prod without checking |

## Typical Workflow

```
drift-start BAS-10 scaffold dashboard project
  ... code with AI ...
drift-status                    # check progress (read-only)
  ... keep coding ...
drift-sync                      # full analysis + Jira update
drift-start BAS-11 next task    # new session
```
