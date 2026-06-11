---
name: rahul-dev-bootstrap-devlog
description: Idempotent _devlog/ vault + docs/ scaffold setup for any project. Triggers on bootstrap devlog, set up devlog, enable session capture, scaffold devlog.
---

# /rahul-dev-bootstrap-devlog — Project Vault + Scaffold Setup

## Purpose
Make any project directory compatible with the global Obsidian-devlog workflow. Idempotent — safe to re-run. Does NOT modify existing files (skips if present).

## Prerequisites
- Current working directory is the project root (nearest `.git` or the directory the user intends as root).
- Canonical devlog spec exists at `C:\Users\Me\Desktop\GTC\hub\_devlog\CLAUDE.md`. Read it once to copy the schema.
- Root Obsidian vault exists at `C:\Users\Me\Desktop\Obsidian\`.

## Pre-flight (scoped contradiction scan — global rule 4)

This skill writes `.claude/settings.json` and `_devlog/CLAUDE.md`, which are CLAUDE.md-adjacent surfaces — scan fires per §15.9.

1. Enumerate CLAUDE.md chain: `<project>/CLAUDE.md` (if exists) → parent CLAUDE.mds → `~/.claude/CLAUDE.md`.
2. Skim top-level headings for conflicts on: Project Bootstrap rule, devlog scope, hook settings, Obsidian vault path.
3. If conflict found, emit: `CONFLICT: <path:line> vs <path:line> — precedence winner: <choice>`.
4. If ambiguous (two rules both satisfiable ≠ true), ASK before proceeding.

## Boundary (important)

`/rahul-dev-bootstrap-devlog` writes `_devlog/CLAUDE.md` (the devlog spec). It does **NOT** create the project-root `CLAUDE.md` (tech stack, repo structure, OVERRIDE blocks — that is Rahul's / `/init`'s job per global rule 73). If project-root `CLAUDE.md` is missing, print a reminder at the end of step 9: *"project-root CLAUDE.md missing — consider `/init` or create manually per global rule 73."*

## Steps (execute in order)

### 1. Detect project root
- If `$CLAUDE_PROJECT_DIR` is set, use it.
- Else use the git root: `git rev-parse --show-toplevel` — if that fails, use current PWD.
- Store as `$PROJECT_ROOT`. Derive `$PROJECT_NAME` as `basename "$PROJECT_ROOT"`.

### 2. Idempotency guard
- If `$PROJECT_ROOT/_devlog/CLAUDE.md` exists → print "devlog already initialised at $PROJECT_ROOT/_devlog — nothing to do" and exit.

### 3. Create `_devlog/` scaffold
Create these directories (mkdir -p, idempotent):
- `_devlog/inbox/`
- `_devlog/sources/{sessions,decisions,references}/`
- `_devlog/daily/`
- `_devlog/wiki/{context,architecture,features,sessions,decisions,patterns}/`
- `_devlog/scripts/`

### 4. Write canonical spec
Copy `C:\Users\Me\Desktop\GTC\hub\_devlog\CLAUDE.md` to `$PROJECT_ROOT/_devlog/CLAUDE.md`, with these substitutions on the first 30 lines:
- `GTC Hub` → `$PROJECT_NAME`
- `Gene Therapy Consultancy` → `(set by $PROJECT_NAME)`
- `GTC/hub/_devlog/` → `$PROJECT_ROOT/_devlog/`

Keep the rest of the spec verbatim — operations (SESSION, INGEST, QUERY, LINT, PLAYBOOK) are universal.

### 5. Write starter files

`_devlog/index.md`:
```markdown
# $PROJECT_NAME — Devlog Index
## Wiki pages
<!-- populated by SESSION operation -->
## Recent sessions
<!-- newest first -->
```

`_devlog/log.md`:
```markdown
# $PROJECT_NAME — Operation Log (newest first)
```

`_devlog/wiki/context/current-state.md`:
```markdown
# $PROJECT_NAME — Current State
> Last updated: <today> by /rahul-dev-bootstrap-devlog

## Phase
Just bootstrapped. No features shipped yet.

## What's Working
- (none — fresh project)

## In Progress / WIP
- (none)

## Broken / Known Issues
- (none known)

## Where to Start
1. Read this file first.
2. Read `_devlog/CLAUDE.md` for devlog operations.
3. Run `/rahul-dev-start-new-project` to plan your first feature.
```

### 6. Write project-local `.claude/settings.json`
If `$PROJECT_ROOT/.claude/settings.json` does not exist, create it:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|Bash|NotebookEdit",
        "hooks": [{
          "type": "command",
          "command": "bash ~/.claude/hooks/universal-capture.sh",
          "timeout": 10
        }]
      }
    ]
  }
}
```
If it exists, do NOT overwrite. Tell the user: "Project already has .claude/settings.json — merge the hook manually or let me show the diff."

### 7. Scaffold runbook directories
```
docs/plans/        (empty)
docs/audit/        (empty)
docs/rollback/     (empty)
memory/            (empty)
```
Plus `memory/project_snapshot.md`:
```markdown
# $PROJECT_NAME — Project Snapshot
> Re-entry document. Read this to understand project state at-a-glance.

## Working
<!-- [YYYY-MM-DD] Feature — one-line description → ref: file.md -->

## In Progress
<!-- -->

## Broken
<!-- -->
```

### 8. Register in root Obsidian vault
Create `C:\Users\Me\Desktop\Obsidian\projects\$PROJECT_NAME.md`:
```markdown
# $PROJECT_NAME

**Path:** `$PROJECT_ROOT`
**Devlog:** `$PROJECT_ROOT/_devlog/`
**Current state:** [[../../$PROJECT_ROOT/_devlog/wiki/context/current-state|current-state.md]]
**Bootstrapped:** <today>

## Recent sessions
<!-- appended by "session done" operation -->
```

Append to `C:\Users\Me\Desktop\Obsidian\index.md` under "Registered projects":
```markdown
- [[projects/$PROJECT_NAME]] — `$PROJECT_ROOT` (bootstrapped <today>)
```

### 9. Confirm and guide
Print:
```
✅ Devlog bootstrapped at $PROJECT_ROOT/_devlog/
✅ Registered in root Obsidian at Desktop/Obsidian/projects/$PROJECT_NAME.md
✅ PostToolUse hook active — every Edit/Write/Bash now logs to _devlog/inbox/

Next steps:
  → Plan a feature:   /rahul-dev-start-new-project
  → Or harden:        /rahul-dev-harden-project start
  → Between phases:   /compact to free context, then continue
```

## Failure modes
- **Canonical spec missing:** fall back to writing a minimal `_devlog/CLAUDE.md` that says "See GTCwikiObsidian canonical spec" and warn the user.
- **Not a git repo:** still works — just uses PWD as project root. Warn that rollback will be manual.
- **Permission error writing to Desktop/Obsidian/:** skip registration, print warning, continue.

## What this skill does NOT do
- Install the global PostToolUse hook in `~/.claude/settings.json` — that is a one-time setup done separately.
- Start a session / open Obsidian — the user does that.
- Write to `_devlog/wiki/` other than `current-state.md` stub — that's for SESSION operation.
