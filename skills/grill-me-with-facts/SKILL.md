---
name: grill-me-with-facts
description: Scan any repo for factual contradictions across ALL authoritative sources (git, memory, CLAUDE.md chain, Obsidian, Graphify, GitHub, code comments, plans, schemas), surface them one-at-a-time for interactive resolution, then propagate the winning values everywhere via a self-closing verify loop. Self-improves over runs via authority-weights.json. Use when Rahul wants to reconcile conflicting information across the project knowledge base.
---

> **Cloud sessions:** State files (`authority-weights.json`, `contradiction-history.jsonl`) are loaded from the skill directory if present. On cloud VMs without a local hub clone, Obsidian vault and NotebookLM checks are skipped gracefully.

<what-to-do>

Run in the mode indicated by the invocation:

- **No args** → AUDIT mode: scan all sources, surface contradictions one at a time, write a snapshot.
- **`--propagate <snapshot-path>`** → PROPAGATE mode: apply all resolutions from the snapshot, then loop until re-audit is clean.
- **`--scan-only`** → SCAN ONLY mode: run the full scan phase, print all candidates ranked by severity, then stop. No interactive loop, no snapshot written. Use for a quick read before a merge or a meeting.
- **`--focus <package>`** → FOCUSED AUDIT: AUDIT scoped to a single package (e.g. `--focus email-nurture`). Only scans sources in/under `packages/<package>/` plus shared context files (`CLAUDE.md`, `context/`, `current-state.md`, memory). Faster for targeted work.
- **`--live`** → LIVE CHECK mode: run CLI tools to verify the deployed/running state matches what docs claim. Separate from doc-vs-doc audit — see the Live Check section below.

Ask zero clarifying questions before starting. Dive into the scan immediately and surface what you find.

</what-to-do>

<supporting-info>

---

## AUDIT mode — 3 phases

### Phase 1: SCAN (automated, run silently)

Read every available source. Adapt to what exists — if a source is absent, skip it and note it
in the final snapshot as "not scanned — not found."

**Source priority table** (highest trust first):

| Source | How to access | Trust |
|--------|--------------|-------|
| Git log + diff | `git fetch origin main 2>/dev/null` first, then `git log --oneline -20`; `git diff origin/main...HEAD` | Highest — production truth (fetch mandatory — stale `origin/main` gives wrong diff) |
| GitHub issues/PRs (open) | `gh issue list --state open --limit 20`; `gh pr list --state open` | High — team decisions |
| GitHub PR/issue comments | `gh api repos/<owner>/<repo>/issues/<N>/comments` for high-signal open items | Medium-High |
| CLAUDE.md chain | Walk CWD → root, read every CLAUDE.md encountered | High |
| All package CLAUDE.mds | `glob packages/*/CLAUDE.md` — read ALL of them, not a spot-check | High |
| Memory files | `~/.claude/projects/<hash>/memory/*.md` + `MEMORY.md` index | High |
| Current-state / devlog | `_devlog/wiki/context/current-state.md`; `_devlog/log.md` last 50 lines | High |
| Facts / briefing cards | `docs/projects/*/facts.jsonl`; `docs/projects/*/briefing-card.md`; `docs/projects/*/_meta.json` | High |
| AS push state | Read `.clasp-push-state` in each AS project dir → `git diff <commit>..HEAD -- <dir>/ --stat` | Medium-High — non-empty = code changed since last push |
| Obsidian vault | `~/Desktop/Obsidian/` — daily notes, wiki/, strategy/, business/ | High — often more current than repo |
| Graphify articles | `graphify-out/articles/*.md` if present — LLM-extracted file summaries | Medium-High — surfaces drift invisible to raw reads |
| NotebookLM | `~/.notebooklm-venv/Scripts/notebooklm.exe --json list`; query most relevant notebook | Medium |
| Skills index integrity | Compare paths listed in `SKILLS_INDEX.txt` + `SKILLS_REFERENCE.md` against `ls ~/.claude/skills/` | Medium — flags listed-but-deleted or exists-but-unlisted skills |
| Plans | `~/.claude/plans/*.md`; `docs/plans/*.md` | Medium — often stale post-completion |
| Package docs | `CONTEXT.md`; `CONTEXT-MAP.md`; `README.md` | Medium |
| Code comments | `grep -r "TODO\|FIXME\|NOTE\|HACK\|deprecated" --include="*.ts" --include="*.js" --include="*.mjs" -l` → read the flagged lines | Low-Medium |
| Supabase schema | `shared/supabase-schema.sql`; `supabase/migrations/` (latest); `shared/schema.ts` | Low-Medium — ground truth for data model |

**Always load authority weights first** (if they exist):
`~/.claude/skills/grill-me-with-facts/authority-weights.json`

Use weights to pre-rank candidates: lower-weight source = likely stale = flag it in presentation.

**Always load contradiction history** (if it exists):
`~/.claude/skills/grill-me-with-facts/contradiction-history.jsonl`

Before surfacing any candidate, check history for a matching prior resolution:
- Same field + same two sources, and sources now agree → skip (already clean)
- Same field + same two sources, and sources still disagree → resurface (propagation failed or regressed)
- Skip patterns with confidence ≥ 0.8 resolved the same way ≥ 3 times → auto-apply, still log

**Contradiction detection heuristics** — look for ALL of these:

1. **Value mismatch** — same entity described with different values in 2+ sources
   - Dollar amounts, counts, version numbers, port numbers, table names, column names
   - e.g. "€25K" in a plan vs "€20K" in memory; "26 experts" in log vs "120+" in CLAUDE.md

2. **Status mismatch** — same feature marked differently across sources
   - "✅ Working" in current-state.md vs "WIP" or "broken" in a CLAUDE.md or plan
   - GitHub issue open but code/memory says it's fixed
   - `status: active` in a plan file where the feature shipped months ago

3. **Path/reference rot** — file paths or line refs that no longer exist
   - `file:line` refs in CLAUDE.md or memory that 404 against the current tree
   - Skill paths in SKILLS_INDEX.txt pointing to missing files
   - Import paths in comments referencing deleted modules

4. **Stale counts** — numbers that describe the same thing but disagree
   - Expert counts, member counts, PR numbers, issue counts

5. **Config drift** — same config key with different values in different files
   - Port numbers, API endpoints, environment variable names, thresholds

6. **GitHub vs repo divergence** — a PR/issue says X is done but the code or CLAUDE.md still describes old behavior

7. **Obsidian vs repo divergence** — a daily note or strategy doc states something that contradicts STRATEGY.md, current-state.md, or memory files

8. **Graphify surface** — a graphify article summary contradicts the actual file content (file changed after last graph build)

9. **Comment rot** — inline code comments that contradict the surrounding implementation or CLAUDE.md description

10. **Cross-memory conflicts** — two memory files asserting different values for the same fact (e.g., two pricing memories with different numbers)

11. **Reference resolution (deep path rot)** — grep the entire CLAUDE.md chain for backtick paths, `file:line` patterns, and `[[wiki/...]]` links. For each: resolve the path against the actual file tree (`Test-Path` on PowerShell; `[ -e ]` in bash). Flag any that don't resolve. This catches refs that look syntactically valid but point to moved or deleted files — invisible to heuristic #3's surface scan.

12. **Untracked doc files** — run `git status --porcelain` and filter lines starting with `??` for files with `.md`, `CLAUDE.md`, `CONTEXT.md`, `INDEX.md`, or `SKILL.md` in the name. Surface each as: "Untracked doc file `<path>` — not in git and not referenced in any CLAUDE.md scan list. Intentional?" These are documentation gaps a cold agent cannot discover from the repo.

After scanning, internally rank candidates: **clear value conflict > status mismatch > path rot > stale count > comment rot > untracked files**.

Deduplicate: if two candidates point to the same underlying disagreement, merge them into one.

---

### Phase 2: RESOLVE (interactive — one at a time)

Print the total count first: "Found N contradictions. Working through them one at a time."

For each candidate, present this block and wait for input before continuing:

```
─── CONTRADICTION [i] of [N] ────────────────────────────────
Field:     <topic / field name>
Source A:  <file:line or source name>
           "<value A>"
           (date: <YYYY-MM-DD if known>, trust: <source type>)
Source B:  <file:line or source name>
           "<value B>"
           (date: <YYYY-MM-DD if known>, trust: <source type>)

Recommendation: [A] ← newer + higher-trust source   (or explain why)

  [A] Use Source A's value
  [B] Use Source B's value
  [C] Neither — I'll type the correct value
  [S] Skip for now
  [Q] Stop and write snapshot with resolved so far
─────────────────────────────────────────────────────────────
```

Rules:
- Always give a recommendation with `←` and a brief reason (newer date, higher source trust, git agrees, etc.)
- Never silently pick without user confirmation
- Accept bare `a` / `b` / `c` / `s` / `q`
- For `[C]`: "Enter the correct value:" → confirm before recording
- After each resolution, immediately record in contradiction-history.jsonl (do not batch)
- After resolution, update authority-weights.json for both sources

If `>10` candidates: after the 10th, ask: "Continue? [Y/n] or [S] skip rest and write snapshot"

---

### Phase 3: SNAPSHOT (write immediately after resolve loop)

Write to `_devlog/inbox/truth-audit-YYYY-MM-DD-HHmm.md`
(fallback if no `_devlog/`: write to CWD as `truth-audit-YYYY-MM-DD.md`)

```markdown
---
title: Source Truth Audit
type: inbox
source: grill-me-with-facts
created: <ISO timestamp>
status: fresh
repo: <repo-name from git remote or CWD folder name>
contradictions_found: <N>
contradictions_resolved: <M>
contradictions_skipped: <K>
---

# Source Truth Audit — <repo> — <YYYY-MM-DD>

## Summary
- **Found:** N contradictions across <sources scanned count> sources
- **Resolved:** M | **Skipped:** K | **Auto-applied:** L (from history/weights)
- **Sources not scanned:** <list any that were absent>

## Resolved Contradictions

### 1. <field / topic>
- **Winner:** `"<winning value>"` — <Source B name>:<line>, date <YYYY-MM-DD>
- **Superseded:** `"<losing value>"` — <Source A name>:<line>
- **Propagation targets:**
  - [ ] `<file path>` line <N> — replace `"<old>"` with `"<new>"`
  - [ ] `<file path>` — field `<key>` — update value
  - [ ] `<file path>` — already correct, no change needed

### 2. ...

## Skipped / Open
- **<field>:** Source A (`"<value>"`) vs Source B (`"<value>"`) — skipped
```

**There is NO separate "Propagation Checklist" section.** The `[ ]` items under each contradiction's "Propagation targets" ARE the checklist. PROPAGATE mode reads directly from those items. One source of truth — no sync drift.

After writing: print the snapshot path and the propagate command.

---

## PROPAGATE mode — self-closing loop

**Entry:** `--propagate <snapshot-path>` — re-read the snapshot, execute the checklist.

### Loop structure (max 3 iterations)

**Step A — APPLY**

For each unchecked `[ ]` item under any `### N.` contradiction's **Propagation targets** block:
1. Read the target file with 5 lines of surrounding context (not just the target line)
2. Apply the change intelligently — understand context before writing, not regex-only
3. If the target file is missing: note "not found — skip" and move on
4. Tick the checkbox `[x]` in the snapshot file immediately after each change
5. **Verify the edit landed:** re-read the changed lines and confirm the new value is present. If not: mark `[!FAILED]` instead of `[x]` — do not silently continue
6. If the changed file is `_devlog/wiki/context/current-state.md`: also update its `updated:` frontmatter date to today (YYYY-MM-DD)
7. Append to `_devlog/inbox/propagation-log-<YYYY-MM-DD-HHmm>.md`:
   ```
   [HH:MM] CHANGED <file>:<line> — "<old>" → "<new>"
   [HH:MM] APPLY FAILED <file>:<line> — edit did not land, review manually
   ```

**Pruning check (after each apply):**
- Does another file now contain a redundant copy of the same fact?
- Is the redundant copy providing zero additional context vs. the canonical source?
- If yes: offer `"Redundant copy at <file:line>. Remove to reduce future drift? [Y/n]"`
- If confirmed: delete the redundant block and log the deletion

**Step B — SELF-REVIEW (silent re-audit)**

Re-run the full SCAN phase against the same source set.
- Load contradiction-history.jsonl — skip anything already resolved
- Load authority-weights.json — use to pre-filter low-signal candidates
- Produce a delta list: only contradictions that survived OR were introduced by propagation

**Step C — VERDICT**

```
0 delta candidates → print:
  "✅ Audit clean after [N] iteration(s). All contradictions resolved."
  List sources that were updated.

1+ delta candidates → print:
  "⚠️ [N] contradiction(s) remain after propagation."
  For each: show the condensed block (field + sources + values)
  "Resolve interactively? [Y/n]"
  → Y: run RESOLVE phase on deltas → update snapshot → continue loop
  → N: write partial completion report and stop
```

If max iterations (3) hit without clean: print
`"Stopped at max iterations. [N] contradictions unresolved — review manually."`

---

## LIVE CHECK mode (--live)

Verifies deployed/running state matches what docs claim. Runs real CLI commands — does NOT edit any files. Skip any check where the CLI is unavailable and note as "skipped — CLI not found."

Output: a table of checks with **PASS / FAIL / SKIP** per item. No interactive resolution.

### Checks to run

**1. Netlify env vars**
```bash
netlify env:list --filter website_v5 --json
```
- Compare `APPS_SCRIPT_URL` and `APPS_SCRIPT_SECRET` for `production` and `deploy-preview` contexts
- Compare against `netlify.toml` `[context.production.environment]` and `[context.deploy-preview.environment]`
- FAIL if any value mismatches; PASS if all match; SKIP if `netlify` CLI unavailable

**2. Apps Script push state**
For each `.clasp*.json` in `packages/email-nurture/expert-network/`:
```bash
# Read last-pushed commit from .clasp-push-state, then:
git diff <commit>..HEAD -- packages/email-nurture/expert-network/ --stat
```
- Empty diff → PASS (AS is in sync with git)
- Non-empty diff → FAIL: "AS out of sync — `clasp push` needed. Changed: `<file list>`"
- `.clasp-push-state` missing → SKIP

**3. Google Forms availability**
For each `formId` in `packages/email-nurture/expert-network/config.js`:
```bash
gws forms forms get --params '{"formId":"<id>"}' | node -e "
  process.stdin.resume(); let d='';
  process.stdin.on('data', c => d += c);
  process.stdin.on('end', () => {
    const f = JSON.parse(d);
    console.log(f.formId, f.responderUri ? 'OPEN' : 'CLOSED', f.info?.title || '');
  })"
```
- All forms return `OPEN` → PASS
- Any form `CLOSED` or request errors → FAIL: list which form IDs
- `gws` CLI unavailable → SKIP

**4. current-state.md freshness**
- Read `updated:` from `_devlog/wiki/context/current-state.md` frontmatter
- If > 14 days ago → FAIL: "current-state.md not updated in >2 weeks — may be stale"
- Otherwise → PASS

---

## Authority weights — self-improving

File: `~/.claude/skills/grill-me-with-facts/authority-weights.json`

Create if absent:
```json
{
  "source_authority": {},
  "skip_patterns": [],
  "last_updated": null
}
```

**After each user resolution**, update:
```json
{
  "source_authority": {
    "facts.jsonl": { "wins": 0, "losses": 0, "weight": 0.5 },
    "memory/*.md":  { "wins": 0, "losses": 0, "weight": 0.5 },
    "~/.claude/plans/*.md": { "wins": 0, "losses": 0, "weight": 0.5 },
    "obsidian/wiki/*": { "wins": 0, "losses": 0, "weight": 0.5 }
  },
  "skip_patterns": [],
  "last_updated": "<ISO>"
}
```

Weight formula: `wins / (wins + losses)` — treat 0/0 as 0.5 (unknown trust). Floor at 0.05 so low-trust sources are still surfaced occasionally.

**skip_patterns** — add entry when same field + same resolution occurs ≥3 times with consistent winner:
```json
{ "pattern": "expert count", "resolved_to": "supabase live count", "confidence": 0.9, "times_seen": 4 }
```

On next SCAN: if a candidate matches a skip_pattern with confidence ≥ 0.8 → auto-apply, log as
`"Auto-applied from history (pattern: '<pattern>', confidence: X)"` in the snapshot.

---

## Contradiction history — deduplication

File: `~/.claude/skills/grill-me-with-facts/contradiction-history.jsonl`

Append one line per resolution (never delete, never rewrite):
```json
{"id":"<uuid>","date":"<ISO>","field":"payment terms","source_a":"facts.jsonl:3ca89f7a","source_b":"memory/project_gtc.md:L12","winner":"source_b","winning_value":"single invoice one week after project start","repo":"gtc-hub"}
```

On each SCAN, before surfacing a candidate:
1. Find matching entries by `field` + `source_a` + `source_b` (or reversed)
2. If match found AND sources now agree → skip (clean)
3. If match found AND sources still disagree → resurface with note: `"Previously resolved <date> — regression detected"`
4. If no match → surface normally

---

## Edge cases

| Situation | Handling |
|-----------|----------|
| No `_devlog/` in repo | Write snapshot to CWD; propagation log also to CWD |
| 0 contradictions found | Print "No contradictions found. Sources appear consistent." No snapshot written. |
| >20 candidates | After 10th, offer "Continue [Y/n] or [S]kip rest" |
| Propagation target missing | Note in snapshot as "file not found — skip"; continue |
| User quits mid-session (`Q`) | Write partial snapshot immediately; print propagate command for what's resolved |
| authority-weights.json missing | Create with defaults; don't block |
| contradiction-history.jsonl missing | Create empty; don't block |
| Obsidian not installed / vault not found | Skip; note "Obsidian vault not scanned — not found at ~/Desktop/Obsidian" |
| NotebookLM CLI not available | Skip; note "NotebookLM not scanned — CLI not found" |
| Graphify output not present | Skip; note "Graphify not scanned — no graphify-out/ directory" |
| --live CLI unavailable | Mark that check SKIP; continue with remaining checks |
| Edit applied but verify re-read fails | Mark `[!FAILED]`; log to propagation log; continue with remaining items |

---

## What this skill is NOT

- Not `/grill-with-docs` — that grills design decisions and terminology. This grills factual values.
- Not `content/contradiction-check` — that checks LinkedIn posts against prior posts.
- Not `project-intel reconcile` — that is scoped to one project code. This is repo-wide.

If the user invokes this and the issue is actually a terminology/design decision, say:
"This looks like a design decision rather than a factual inconsistency — consider `/grill-with-docs` instead."

</supporting-info>
