# Claude Code — Production Hardening Runbook
### For making existing, working code production-ready

> **Use this when:** you already have code that works, and now need quality, security, reliability, and operability.
>
> **How to use:** Work top-to-bottom. Every phase contains everything you need — a single Settings block at the top, then Why/Handoff/Paste/Checks/Gate. No cross-referencing required.

---

## Mindset Shift

**Building:** "Does it work?"
**Hardening:** "What happens when it breaks at 2am, when a stranger attacks it, when traffic 10x's, when the person who wrote it is unreachable?"

Hardening is about **removing unknowns**, not adding features.

---

## Universal Rules (apply to every phase)

- **Opus** when being wrong is expensive (data loss, breach, irreversible). **Sonnet** otherwise.
- If output is saved to a file → `/clear` is safe after saving. The file becomes the context.
- Never `/clear` mid-debug or mid-conversation-alignment.
- Read-only phases = same worktree parallel-safe. Writing phases in parallel = sub-worktrees + merge.
- Every phase ends with a gate. Don't move on until you can answer "yes."

---

## Quick Reference — Slash Commands

| Phase | Skill / Command | Model |
|---|---|---|
| Start (any phase) | `/harden-project start` or `/harden-project phase <n>` | per phase |
| Phase 0 (Snapshot) | `/harden-project phase 0` | Sonnet |
| Phase 1 (Inventory) | `/harden-project phase 1` | Sonnet |
| Phase 2 (Baseline) | `/harden-project phase 2` | Sonnet |
| Phase 3a/b/c (Security) | `/harden-project phase 3` (spawns 3 terminals) | Opus |
| Phase 4 (Quality) | `/harden-project phase 4` → `/simplify` | Sonnet |
| Phase 5 (Tests) | `/harden-project phase 5` → `/superpowers:test-driven-development` | Opus→Sonnet |
| Phase 6 (Observability) | `/harden-project phase 6` | Sonnet |
| Phase 7 (Config/Secrets) | `/harden-project phase 7` | Opus |
| Phase 8 (Deploy/Rollback) | `/harden-project phase 8` | Sonnet |
| Phase 9 (Data/Backup) | `/harden-project phase 9` | Opus |
| Phase 10 (Cost) | `/harden-project phase 10` | Sonnet |
| Phase 11 (Docs) | `/harden-project phase 11` | Sonnet |
| Phase 12 (Final) | `/harden-project phase 12` → `/phase-gate final` | Opus |
| 1-day subset | `/harden-project 1-day` | per phase |
| Between any two phases | `/handoff` → `/clear` → `/resume` (recommended before 3, 5, 9, 12) | — |

**Per-phase gate:** every phase automatically invokes `/phase-gate <name>` — the founder checklist is required before advancing.
**Progress brief at any time:** `/founder-brief`.

---

# 🟢 PHASE 0 — Snapshot & Safety Net (do this FIRST)

### ⚙️ Settings
- **Interface:** Claude Code (terminal) — or Claude Cowork for GUI pairing on this setup phase
- **Model:** Sonnet 4.6 — mechanical git work
- **Effort:** low
- **Permission mode:** Accept edits — runs git init/tag/push
- **Worktree checkbox:** ☑ ON — creates the `hardening` worktree this phase
- **Directory:** project root (on main until worktree is made)
- **Environment:** Local
- **Files to attach:** none
- **Commands/Skills:** none
- **Connectors:** GitHub (to push the tag)
- **Plugins:** superpowers
- **/clear:** before ❌ (first phase), during ❌, after ✅ — tag + worktree persist on disk
- **/compact:** ✅ safe / rarely needed
- **Parallel-safe:** ❌ serial only — everything depends on this

**Why this setup:** Mechanical git work, no judgment required. Must run first because every later phase assumes you have a rollback point and a worktree to operate in.

### ✅ Command to paste:
```
PHASE 0 — Snapshot & Safety Net.

Before we harden anything, I need a safety net.

1. Confirm we're in a git repo. If not, initialise one and commit the current state with message "pre-hardening baseline — known-working state".
2. Create a git tag: pre-hardening-2026-04-15
3. Push to remote (GitHub) so there's an offsite copy.
4. Create a worktree called "hardening" so we never touch main during this process.

Walk me through each step and confirm before destructive operations.
```

### 👀 Founder manual checks after Phase 0:
- [ ] Open GitHub in browser → confirm the `pre-hardening-2026-04-15` tag exists
- [ ] Run `git log -1` and read the commit message out loud
- [ ] Run `git worktree list` → confirm you see the `hardening` worktree

### 🚦 GATE 0
> **I have a remote-backed snapshot I could restore in 5 minutes if everything goes wrong.**

---

# 🟢 PHASE 1 — Inventory

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Sonnet 4.6
- **Effort:** medium
- **Permission mode:** Accept edits — writes `inventory.md` only, read-only on source
- **Worktree checkbox:** ☑ ON — `hardening` (same as Phase 0)
- **Directory:** `hardening` worktree
- **Environment:** Local
- **Files to attach:** none
- **Commands/Skills:** none
- **Connectors:** none
- **Plugins:** superpowers
- **/clear:** before ✅ (Phase 0 output is a git tag), during ✅ only if scan returns 500+ items, after ✅ — output saved to file
- **/compact:** ✅ safe mid-scan on huge codebases
- **Parallel-safe:** ❌ serial only — every later phase reads `inventory.md`

**Why this setup:** You can't harden what you can't see. A single read-only scan writes a durable inventory file that all later phases depend on.

### Handoff — read before starting:
*(nothing — this is the first real work phase)*

### ✅ Command to paste:
```
PHASE 1 — Inventory. Starting fresh session in the `hardening` worktree.

Do a full inventory of this codebase. Give me a Founder Brief with:

1. Languages, frameworks, and their versions
2. External services used (databases, APIs, auth providers, payment, email, storage)
3. Every environment variable the code references (even if not currently set)
4. Deploy targets (Render, Netlify, Vercel, etc.) and what's configured where
5. Domains / DNS records involved
6. Entry points: every URL, CLI command, or cron job that kicks off code
7. Any secrets / API keys found in the codebase (DO NOT print values — just file paths and variable names)

Save the output as /docs/audit/inventory.md
```

### 👀 Founder manual checks after Phase 1:
- [ ] Open `/docs/audit/inventory.md` in a text editor
- [ ] Read every service listed — do you recognise them all? Flag anything you don't
- [ ] Every API key found → confirm it's in env vars, not in the code itself

### 🚦 GATE 1
> **A new engineer could read this file and know what this system is made of.**

---

# 🟢 PHASE 2 — Baseline Behavior

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Sonnet 4.6
- **Effort:** medium
- **Permission mode:** Accept edits — writes `baseline.md`, runs the app for demos
- **Worktree checkbox:** ☑ ON — `hardening`
- **Directory:** `hardening` worktree
- **Environment:** Local
- **Files to attach:** `/docs/audit/inventory.md`
- **Commands/Skills:** none
- **Connectors:** whatever the app depends on (Supabase/Stripe/etc.)
- **Plugins:** superpowers
- **/clear:** before ✅, during ❌ (during demos you want context), after ✅ — output saved
- **/compact:** ✅ safe between feature demos / ❌ don't mid-demo
- **Parallel-safe:** ❌ serial only — Phases 5 and 12 depend on this

**Why this setup:** "Works on my machine" is not a baseline. You need a written, demo-verified list of what currently works BEFORE any changes — otherwise you won't know what regressions your hardening introduced.

### Handoff — read before starting:
- `/docs/audit/inventory.md`

### ✅ Command to paste:
```
PHASE 2 — Baseline Behavior. Starting fresh session in `hardening` worktree.

First, read /docs/audit/inventory.md so you know what features exist.

Then for each user-facing feature:
1. List the feature (plain English)
2. Walk me through the exact clicks / commands to demo it
3. Tell me what the success case looks like
4. Tell me what the most likely failure case is (wrong input, timeout, empty state)

Then: I will execute each one while you watch. Stop me if I skip anything. Record which features actually work, which don't, and which are half-done. Save as /docs/audit/baseline.md
```

### 👀 Founder manual checks after Phase 2:
- [ ] Pick 3 features from `baseline.md` and actually click through them yourself
- [ ] Write "✅ I watched this work" next to each one in the file
- [ ] Anything you couldn't demo → mark BROKEN and add to priority list

### 🚦 GATE 2
> **I have a written list of what works TODAY, verified by live demo, not assumption.**

---

# 🟢 PHASE 3 — Security Audit (3a, 3b, 3c)

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Opus 4.6 — breach risk; worth running 3a and 3c twice (Sonnet + Opus) and diffing
- **Effort:** high
- **Permission mode:** Plan mode — all three sub-phases are READ-ONLY, docs-only output
- **Worktree checkbox:** ☑ ON — `hardening` (same worktree for all three terminals)
- **Directory:** `hardening` worktree
- **Environment:** Local
- **Files to attach:** `/docs/audit/inventory.md`, `/docs/audit/baseline.md`
- **Commands/Skills:** none (3b may shell out to `npm audit` / `pip-audit` / `govulncheck`)
- **Connectors:** none
- **Plugins:** superpowers
- **/clear:** before ✅ each sub-phase, during ❌, after ✅ — each outputs its own file
- **/compact:** ✅ safe if one audit is returning huge output mid-run
- **Parallel-safe:** ✅ with siblings [3a, 3b, 3c] — read-only, each writes to a different file

**Why this setup:** Read-only scans cannot conflict — safe to run three terminals in the same worktree. Security is the one place where being wrong is expensive enough to justify Opus. Split because each scan uses different tools and should focus on one concern.

### Handoff — each terminal reads before starting:
- `/docs/audit/inventory.md`
- `/docs/audit/baseline.md`

### ✅ Terminal A — Secrets scan (Phase 3a):
```
PARALLEL CONTEXT: You are Agent A of 3 running Phase 3 security audit.
- Your task: 3a — secrets scan ONLY
- Sibling B is running 3b (dependency audit) in another terminal
- Sibling C is running 3c (OWASP review) in another terminal
- Your output file: /docs/audit/security-secrets.md
- DO NOT write to: security-deps.md or security-owasp.md
- READ-ONLY on source files — do not modify code.

First read /docs/audit/inventory.md for context.

Then scan the entire codebase (including git history if possible) for:
- Hardcoded API keys, passwords, tokens
- Database connection strings with credentials
- Private keys or certificates
- .env files accidentally committed
- Debug/admin endpoints without auth

List each finding with file:line. Redact the secret value. Save to /docs/audit/security-secrets.md
```

### ✅ Terminal B — Dependency audit (Phase 3b):
```
PARALLEL CONTEXT: You are Agent B of 3 running Phase 3 security audit.
- Your task: 3b — dependency audit ONLY
- Sibling A: 3a (secrets scan) / Sibling C: 3c (OWASP review)
- Your output: /docs/audit/security-deps.md
- READ-ONLY.

Run dependency audits: `npm audit` / `pnpm audit` / `pip-audit` / `govulncheck` (whichever applies).

Summarise: critical vulns, high vulns, fix path for each, upgrade strategy least likely to break. Save to /docs/audit/security-deps.md
```

### ✅ Terminal C — OWASP review (Phase 3c):
```
PARALLEL CONTEXT: You are Agent C of 3 running Phase 3 security audit.
- Your task: 3c — OWASP review ONLY
- Sibling A: 3a (secrets) / Sibling B: 3b (deps)
- Your output: /docs/audit/security-owasp.md
- READ-ONLY.

First read /docs/audit/inventory.md + baseline.md.

Review against OWASP Top 10:
- Injection (SQL, command, XSS)
- Broken authentication / session management
- Sensitive data exposure (logs, error messages, URLs)
- Broken access control (can user A see user B's data?)
- Security misconfiguration (CORS, headers, defaults)
- Unvalidated inputs at every boundary
- Rate limiting on public endpoints

Founder Brief: top 5 risks ranked by (impact × likelihood). Save to /docs/audit/security-owasp.md
```

### 👀 Founder manual checks after Phase 3:
- [ ] Open all three security files, read the "top risks" sections
- [ ] Any finding labelled CRITICAL → STOP. Fix before anything else.
- [ ] For any "won't fix" decisions → write WHY in a comment (future you will ask)
- [ ] Check: are any secrets you *thought* were in env vars actually in code?

### 🚦 GATE 3
> **Every secret is in env vars. No critical/high vulns remain. Top 5 risks are either fixed or have a written exception from me.**

---

# 🟢 PHASE 4 — Code Quality & Simplification

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Sonnet 4.6 — pattern-matching cleanups
- **Effort:** medium
- **Permission mode:** Accept edits — modifies arbitrary code files
- **Worktree checkbox:** ☑ ON — `hardening` (SERIAL, single writer)
- **Directory:** `hardening` worktree
- **Environment:** Local
- **Files to attach:** `/docs/audit/inventory.md`, `/docs/audit/baseline.md`, `/docs/audit/security-*.md`
- **Commands/Skills:** `/simplify`
- **Connectors:** none
- **Plugins:** superpowers
- **/clear:** before ✅, during ✅ safe if reviewing 20+ files, after ⚠️ only once simplify + baseline demo both pass
- **/compact:** ✅ safe mid-review / ❌ don't mid-edit
- **Parallel-safe:** ❌ serial only — writes to arbitrary files

**Why this setup:** Simplification touches many files unpredictably — parallel writers would collide. Run serial in the main hardening worktree. Don't clear until baseline demo confirms nothing broke.

### Handoff — read before starting:
- `/docs/audit/inventory.md`
- `/docs/audit/baseline.md`
- `/docs/audit/security-*.md` (don't clean up code that's about to be rewritten for security)

### ✅ Command to paste:
```
PHASE 4 — Simplify. Fresh session, same `hardening` worktree.

First read: inventory.md, baseline.md, all three security-*.md files.

/simplify

Review the entire codebase. Flag:
1. Duplicated logic across files
2. Dead code (unreferenced functions, commented-out blocks, unused imports)
3. Premature abstractions (interfaces with one implementation)
4. Gold-plating (error handling for impossible cases, unused config options)
5. Inconsistent patterns

Give me a ranked list. Save as /docs/audit/quality.md.

Then do ONLY the cleanups that are LOW RISK and HIGH VALUE. Leave risky refactors for later.
After EACH cleanup, re-run the relevant baseline demo and confirm it still works.
```

### 👀 Founder manual checks after Phase 4:
- [ ] `git diff` → scroll through it. Do you understand every change?
- [ ] Re-run EVERY baseline demo from `baseline.md`. All pass?
- [ ] If AI said "nothing to simplify" → push back once. Usually there is something.

### 🚦 GATE 4
> **Baseline demo still passes. Code is visibly simpler. I understand what was removed.**

---

# 🟢 PHASE 5 — Test Coverage

### ⚙️ Settings
- **Interface:** Claude Code (terminal) — or Claude Cowork for the Opus strategy discussion
- **Model:** Opus 4.6 for strategy → switch to Sonnet 4.6 for writing
- **Effort:** high (strategy) → medium (writing)
- **Permission mode:** Plan mode (strategy) → Accept edits (writing tests)
- **Worktree checkbox:** ☑ ON — `hardening` (SERIAL; test failure = real bug)
- **Directory:** `hardening` worktree
- **Environment:** Local
- **Files to attach:** `/docs/audit/baseline.md`, `/docs/audit/security-owasp.md`, `/docs/audit/quality.md`
- **Commands/Skills:** `/superpowers:test-driven-development` when switching to Sonnet
- **Connectors:** none (mocks for external APIs)
- **Plugins:** superpowers
- **/clear:** before ✅, during ❌ mid-debug, after ⚠️ only once test suite committed and passing
- **/compact:** ✅ safe if test writing runs >1 hour / ❌ don't mid-failure-investigation
- **Parallel-safe:** ❌ serial only — a failing test means a real bug; one agent should reason about it

**Why this setup:** Choosing WHAT to test is an architectural decision — use Opus. Writing tests once decided is mechanical — switch to Sonnet. If a test fails on existing code, you just found a real bug; don't clear while debugging.

### Handoff — read before starting:
- `/docs/audit/baseline.md` (the tests must cover what's in baseline)
- `/docs/audit/security-owasp.md` (auth/permission tests are required)
- `/docs/audit/quality.md` (don't test code you're about to delete)

### ✅ Command to paste (Opus first):
```
PHASE 5 — Test Strategy. Fresh session, `hardening` worktree, Opus.

Read: baseline.md, security-owasp.md, quality.md.

Recommend the MINIMUM test coverage that would catch the most damaging regressions:
1. Happy-path user journeys from baseline.md
2. Every payment / billing / money-handling path (zero tolerance)
3. Every auth / permission boundary
4. Every external API integration (mock the API, test our code)
5. Data migrations if any

Skip: UI pixel tests, trivial getters, framework behavior.

Save the test plan as /docs/audit/test-plan.md. Wait for my approval before writing tests.
```

Then **switch to Sonnet** with `/model`:
```
PHASE 5 — Test Writing. Same worktree, now Sonnet.

Read /docs/audit/test-plan.md and implement the tests exactly as planned.

Confirm all pass. If any fail → STOP and tell me — a failing test on existing code means we just found a real bug. Do not "fix" it by changing the test.
```

### 👀 Founder manual checks after Phase 5:
- [ ] Run the test suite YOURSELF with the exact command (not "the AI ran it")
- [ ] Deliberately break something small (change a string) → a test should fail
- [ ] Revert → tests pass again. This proves tests actually test things.

### 🚦 GATE 5
> **Tests exist for money-paths, auth, and baseline journeys. All pass. Failures (if any) were investigated as real bugs, not "fixed" away.**

---

# 🟢 PHASE 6 — Observability

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Sonnet 4.6
- **Effort:** medium
- **Permission mode:** Accept edits — adds log statements and `/health` route
- **Worktree checkbox:** ☑ ON — `hardening-observability` (sub-worktree) if parallel with 7/8/10, else `hardening`
- **Directory:** observability sub-worktree (or `hardening`)
- **Environment:** Local
- **Files to attach:** `/docs/audit/inventory.md`, `/docs/audit/security-owasp.md`
- **Commands/Skills:** none
- **Connectors:** Sentry (or equivalent error tracker), uptime monitor
- **Plugins:** superpowers
- **/clear:** before ✅, during ✅ between "add logs" and "add /health" sub-tasks, after ✅ once live
- **/compact:** ✅ safe between sub-tasks
- **Parallel-safe:** ✅ with siblings [Phase 7, Phase 8, Phase 10] — but each needs its own sub-worktree

**Why this setup:** Observability modifies source files (adds log statements, `/health` route). If parallel with 7 (config) or 8 (deploy), use sub-worktrees to avoid conflicts. Merge back serially with baseline demo between each merge.

### Handoff — read before starting:
- `/docs/audit/inventory.md` (to know which external calls exist)
- `/docs/audit/security-owasp.md` (to ensure logs don't leak PII)

### Sub-worktree setup (if going parallel):
From `hardening` worktree:
```
git worktree add ../hardening-observability hardening
```
Then `cd` into `hardening-observability` and start a fresh session there.

### ✅ Command to paste:
```
PHASE 6 — Observability. Fresh session.

PARALLEL CONTEXT (delete these lines if running serial): You are Agent A of up to 4.
- Your worktree: hardening-observability (confirm with `git branch --show-current`)
- Scope: Phase 6 ONLY — logging + /health endpoint
- Sibling worktrees may exist: hardening-config (Phase 7), hardening-deploy (Phase 8), hardening (Phase 10 cost audit, read-only)
- You MAY modify: any source file for logging, add /health route
- You MUST NOT touch: .env files, config files, /docs/runbook/deploy.md
- Commit prefix: "observability:"

First read: inventory.md, security-owasp.md.

Then add structured logging where missing:
1. Every external API call: request + response status + duration
2. Every auth event: login, logout, permission denial
3. Every error path: with context (user ID, request ID, input hash)
4. Every state transition that matters (order created, payment captured, email sent)

Use the existing logger. Do NOT log secrets or PII.

Then set up error tracking (Sentry or equivalent) if not present. Document where I access the dashboard.

Then add a /health endpoint returning: app version, DB reachable, external services reachable, uptime. Wire to an uptime monitor. Tell me where I'll get alerted.
```

### 👀 Founder manual checks after Phase 6:
- [ ] Open your error tracker (Sentry) in the browser
- [ ] Trigger an error on purpose (hit a nonexistent URL) → does it show up in the dashboard?
- [ ] Confirm you received the alert email/SMS
- [ ] Hit `/health` in the browser — returns healthy?

### 🚦 GATE 6
> **I know where to look when something is wrong. I get pinged when prod is down.**

---

# 🟢 PHASE 7 — Configuration & Secrets Hygiene

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Opus 4.6 — subtle leaks (logs, errors, URL params) need careful review
- **Effort:** high
- **Permission mode:** Accept edits — modifies `.env`, config files
- **Worktree checkbox:** ☑ ON — `hardening-config` (sub-worktree) if parallel with 6/8/10, else `hardening`
- **Directory:** config sub-worktree (or `hardening`)
- **Environment:** Local
- **Files to attach:** `/docs/audit/inventory.md`, `/docs/audit/security-secrets.md`
- **Commands/Skills:** none
- **Connectors:** none
- **Plugins:** superpowers
- **/clear:** before ✅, during ✅ after half the hardcoded values have been moved, after ✅ once `.env.example` committed
- **/compact:** ✅ safe mid-pass
- **Parallel-safe:** ✅ with siblings [Phase 6, Phase 8, Phase 10] — different files

**Why this setup:** Opus because missing even one leaked secret is a real breach. Owns `.env`, `.env.example`, and config files — isolated from Phase 6 (code logging) and Phase 8 (deploy scripts). Safe in parallel with its own worktree.

### Handoff — read before starting:
- `/docs/audit/inventory.md` (list of env vars)
- `/docs/audit/security-secrets.md` (known leaks to fix)

### ✅ Command to paste:
```
PHASE 7 — Config/Secrets. Fresh session, Opus model.

PARALLEL CONTEXT: You are Agent B (if running parallel).
- Worktree: hardening-config
- Scope: Phase 7 ONLY — config and secrets hygiene
- Sibling worktrees: hardening-observability (Phase 6), hardening-deploy (Phase 8)
- You MAY modify: .env files, config files, anything that reads env vars
- You MUST NOT touch: logging code, /health endpoint, deploy docs
- Commit prefix: "config:"

First read: inventory.md, security-secrets.md.

Then:
1. Every hardcoded URL, threshold, timeout, magic number → move to env var or config file
2. Every secret → confirm it's in env vars only, never in code or logs
3. Create .env.example with every required variable: name + purpose + example format (no real values)
4. Document how to get each secret (e.g. "Stripe: dashboard.stripe.com → API keys → Restricted key")

Save as /docs/runbook/configuration.md.
```

### 👀 Founder manual checks after Phase 7:
- [ ] Grep the codebase yourself for "sk-", "api_key", "password" → any hits in source?
- [ ] Copy `.env.example` to `.env.test`, fill in dummy values, try to run the app
- [ ] Anything missing from `.env.example`? → not documented properly

### 🚦 GATE 7
> **A new developer could set up this project in under 30 minutes using only /docs.**

---

# 🟢 PHASE 8 — Deploy & Rollback

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Sonnet 4.6 — documentation + scripting
- **Effort:** medium
- **Permission mode:** Bypass permissions — briefly, only for the rollback simulation (destructive commands) then back to Accept edits
- **Worktree checkbox:** ☑ ON — `hardening-deploy` if parallel, else `hardening`
- **Directory:** deploy sub-worktree (or `hardening`)
- **Environment:** Local + SSH / Remote Control if deploy target is a remote server
- **Files to attach:** `/docs/audit/inventory.md`, `/docs/runbook/configuration.md`
- **Commands/Skills:** none
- **Connectors:** Render / Netlify / Vercel / GitHub Actions as the deploy target dictates
- **Plugins:** superpowers
- **/clear:** before ✅, during ❌ mid-rollback-simulation, after ✅ once rollback doc is on disk
- **/compact:** ✅ safe mid-doc-writing
- **Parallel-safe:** ✅ with siblings [Phase 6, Phase 7, Phase 10] — different files

**Why this setup:** Mostly documentation, sometimes adds a deploy script. Low conflict risk. Sub-worktree only if genuinely parallel; otherwise serial.

### Handoff — read before starting:
- `/docs/audit/inventory.md` (deploy targets)
- `/docs/runbook/configuration.md` (env vars needed at deploy)

### ✅ Command to paste:
```
PHASE 8 — Deploy & Rollback. Fresh session.

PARALLEL CONTEXT (if applicable): Agent C.
- Worktree: hardening-deploy
- Scope: Phase 8 — deploy/rollback docs only
- Siblings: hardening-observability (6), hardening-config (7)
- You MAY modify: /docs/runbook/deploy.md, deploy scripts
- You MUST NOT touch: app source code, .env files
- Commit prefix: "deploy:"

First read: inventory.md, configuration.md.

Then document the deploy pipeline:
1. How code gets from my machine to production (step by step)
2. Is there a staging environment? If not, recommend.
3. Roll back a bad deploy — ONE command.
4. Blast radius of a bad deploy (downtime? data loss? stuck users?)
5. Are deploys automated on git push or manual?

If rollback isn't a one-liner, make it one. Save as /docs/runbook/deploy.md.

Then SIMULATE a rollback on this worktree (don't touch prod) — walk me through with real commands.
```

### 👀 Founder manual checks after Phase 8:
- [ ] **ACTUALLY roll back** in staging (or a throwaway env)
- [ ] Time yourself — under 5 minutes?
- [ ] Could a hired dev do it from docs alone? Test on a real person.

### 🚦 GATE 8
> **I have executed (or simulated) a rollback. I could do it at 2am without help.**

---

# 🟢 PHASE 9 — Data & Backup

### ⚙️ Settings
- **Interface:** Claude Code (terminal) — or Claude Cowork for a careful GUI walk-through of the restore
- **Model:** Opus 4.6 — data loss is irreversible
- **Effort:** high
- **Permission mode:** Bypass permissions — briefly, backup/restore commands touch prod-equivalent data; switch back after
- **Worktree checkbox:** ☑ ON — `hardening` (SERIAL; parallel work from 6/7/8 must be merged first)
- **Directory:** `hardening` worktree
- **Environment:** Local + SSH / Remote Control to the DB host
- **Files to attach:** `/docs/audit/inventory.md`, `/docs/audit/security-owasp.md`
- **Commands/Skills:** none
- **Connectors:** Supabase / managed DB provider / storage bucket
- **Plugins:** superpowers
- **/clear:** before ✅ once 6/7/8 are merged back, during ✅ between inventory complete and restore test, after ❌ — do NOT clear until restore has been tested
- **/compact:** ✅ safe after backup inventory, before restore / ❌ don't mid-restore
- **Parallel-safe:** ❌ serial only — prod access, concurrent activity risks corruption

**Why this setup:** Touches production data. Opus because "oops, deleted the backup" is unrecoverable. Serial because backup/restore procedures must not run alongside other changes. Never clear until a restore works.

### Handoff — read before starting:
- `/docs/audit/inventory.md` (where data lives)
- `/docs/audit/security-owasp.md` (PII handling)

### ✅ Command to paste:
```
PHASE 9 — Data & Backup. Fresh session, Opus, `hardening` worktree. SERIAL — no other hardening work should be in flight.

First read: inventory.md, security-owasp.md.

Review:
1. Where does user data live? (every DB, every storage bucket)
2. Is it backed up? How often? Where? (if "no" — fix before anything else)
3. Have I ever tested restoring from backup? (if no — we test it now on a throwaway instance)
4. What data is PII? Encrypted at rest? In transit?
5. If a user requests deletion (GDPR), what's the process?

Save as /docs/runbook/data.md. Then walk me through a restore to a throwaway DB.
```

### 👀 Founder manual checks after Phase 9:
- [ ] **Restore a backup to a throwaway database** — don't just "verify backups exist"
- [ ] Count rows in restored DB → matches production?
- [ ] Log into the app pointed at the restored DB → can you read your own user record?

### 🚦 GATE 9
> **I can prove backups work because I have restored from one. I know where PII lives.**

---

# 🟢 PHASE 10 — Cost & Dependency Control

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Sonnet 4.6 — inventory work
- **Effort:** low
- **Permission mode:** Plan mode — pure research, read-only
- **Worktree checkbox:** ☑ ON — `hardening` (same, but this phase is READ-ONLY)
- **Directory:** `hardening` worktree
- **Environment:** Local
- **Files to attach:** `/docs/audit/inventory.md`
- **Commands/Skills:** none
- **Connectors:** Stripe, Render, Netlify, Supabase dashboards as applicable
- **Plugins:** superpowers
- **/clear:** before ✅, during ❌, after ✅
- **/compact:** ✅ safe / rarely needed
- **Parallel-safe:** ✅ with siblings [Phase 6, Phase 7, Phase 8] — doesn't touch code at all

**Why this setup:** Pure research — logs into external dashboards, reads billing pages. Doesn't touch the codebase so it runs in any worktree without conflict.

### Handoff — read before starting:
- `/docs/audit/inventory.md` (list of paid services)

### ✅ Command to paste:
```
PHASE 10 — Cost & Dependency Control. Fresh session, Sonnet.

PARALLEL CONTEXT (if running parallel with 6/7/8): You are Agent D.
- Worktree: hardening (same as others, but you are READ-ONLY)
- Scope: Phase 10 — cost audit only
- You MUST NOT modify any source file or config
- Output: /docs/runbook/costs.md

First read: inventory.md.

Then full cost audit:
1. Every paid service, monthly spend, contract terms
2. Any usage-based service that could spike (OpenAI, Supabase, bandwidth) — set spend caps
3. Free tiers about to flip to paid
4. API keys expiring in next 90 days
5. Anything we're paying for but no longer using

Propose a monthly /loop schedule for cost audits going forward.
```

### 👀 Founder manual checks after Phase 10:
- [ ] Log into each paid service — verify the spend numbers the AI gave you
- [ ] Set billing alerts on every usage-based service
- [ ] Put expiration dates in your calendar (not just in a doc)

### 🚦 GATE 10
> **No service can bill me more than $X without alerting me first. I have calendar reminders for expirations.**

---

# 🟢 PHASE 11 — Documentation for Future You

### ⚙️ Settings
- **Interface:** Claude Code (terminal)
- **Model:** Sonnet 4.6 — writing prose
- **Effort:** medium
- **Permission mode:** Accept edits — writes `/docs/README.md`
- **Worktree checkbox:** ☑ ON — `hardening` (SERIAL; aggregates all prior outputs)
- **Directory:** `hardening` worktree
- **Environment:** Local
- **Files to attach:** ALL `/docs/audit/*.md` and ALL `/docs/runbook/*.md`
- **Commands/Skills:** none
- **Connectors:** none
- **Plugins:** superpowers
- **/clear:** before ✅, during ✅ if generating across many files, after ⚠️ only after reading the final README yourself
- **/compact:** ✅ safe mid-draft
- **Parallel-safe:** ❌ serial only — needs every prior `/docs` file to exist

**Why this setup:** Capstone doc aggregating every `/docs` file produced so far. Runs before Phase 12; if run before other phases finish, it will reference missing files.

### Handoff — read before starting:
- ALL `/docs/audit/*.md`
- ALL `/docs/runbook/*.md`

### ✅ Command to paste:
```
PHASE 11 — Documentation. Fresh session, `hardening` worktree.

Read every file in /docs/audit/ and /docs/runbook/.

Then create /docs/README.md — the single entry point for anyone new to this codebase. Must include:
1. What this product does (1 paragraph, plain English)
2. Architecture diagram in ASCII or Mermaid
3. How to run it locally (step by step)
4. How to deploy (link to deploy.md)
5. How to roll back (link to deploy.md)
6. Where logs and errors live (link to runbook files)
7. Top 5 gotchas / "dragons here" warnings

Bus-factor test: if I get hit by a bus tomorrow, my co-founder should keep this running using only this doc.
```

### 👀 Founder manual checks after Phase 11:
- [ ] Close Claude Code
- [ ] Open `/docs/README.md` on a different machine (or phone)
- [ ] Try to set up the project from scratch using only the README
- [ ] Every step that confused you → fix before merging

### 🚦 GATE 11
> **A technical stranger could keep this running for a week using only /docs.**

---

# 🟢 PHASE 12 — Final Verification & Merge

### ⚙️ Settings
- **Interface:** Claude Code (terminal) — or Claude Cowork for GUI walk-through of the final merge
- **Model:** Opus 4.6 — last gate before main
- **Effort:** high
- **Permission mode:** Bypass permissions — briefly, merge + deploy commands are destructive; switch back once live
- **Worktree checkbox:** ☑ ON — `hardening` (all sub-worktrees must be merged back first), then merges into main
- **Directory:** `hardening` worktree, then main
- **Environment:** Local + SSH / Remote Control to deploy target
- **Files to attach:** every file in `/docs/audit/` and `/docs/runbook/`
- **Commands/Skills:** `/superpowers:verification-before-completion`
- **Connectors:** GitHub, deploy target (Render / Netlify / etc.)
- **Plugins:** superpowers, code-review
- **/clear:** before ❌ — full audit trail wanted, during ❌, after ✅ once merged and tagged
- **/compact:** ❌ never during Phase 12
- **Parallel-safe:** ❌ serial only — this IS the final gate

**Why this setup:** Final verification is the last moment to catch anything wrong. Opus gives deepest reasoning. Never clear — if something breaks post-merge, you want the full conversation as evidence of what was (and wasn't) checked.

### Handoff — read before starting:
- Every file in `/docs/audit/` and `/docs/runbook/`

### Merge sub-worktrees back first:
```
cd <hardening worktree>
git merge hardening-observability
# run baseline demo
git merge hardening-config
# run baseline demo
git merge hardening-deploy
# run baseline demo
```

### ✅ Command to paste:
```
PHASE 12 — Final Verification. Opus, `hardening` worktree, all sub-branches merged back.

/superpowers:verification-before-completion

Read all /docs/audit/ and /docs/runbook/ files.

Full verification sweep:
1. Re-run every feature demo from baseline.md — all still work
2. All tests pass
3. No critical security findings remain
4. /health endpoint returns healthy
5. Rollback simulation works
6. Docs are complete (I've read README.md myself)

Give me a single Founder Brief: what changed overall, what risks are now mitigated, what risks remain, next recommended step.
```

### 👀 Founder manual checks after Phase 12:
- [ ] Merge `hardening` into `main`, tag as `post-hardening-2026-04-15`
- [ ] Deploy to production
- [ ] Click through every baseline feature on the LIVE URL (not localhost)
- [ ] Watch error tracker for 15 minutes post-deploy — anything new?
- [ ] If clear: announce "hardening complete"

### 🚦 GATE 12 — FINAL
> **I watched every baseline feature work in production. All 11 prior gates are green. The Founder Brief makes sense to me.**

---

# 🆘 IF YOU ONLY HAVE 1 DAY

Do only these (all serial, same `hardening` worktree, clear between each):
1. Phase 0 (snapshot) — 15 min
2. Phase 3a (secrets scan) — 30 min — Opus
3. Phase 3b (dependency audit + fix criticals) — 2 hrs
4. Phase 6 (error tracking + /health) — 2 hrs
5. Phase 8 (document rollback) — 1 hr
6. Phase 9 (confirm backups exist + restore test) — 30 min — Opus

These six prevent: leaked secrets, silent breakage, irrecoverable deploys, data loss.

---

# 🔁 ONGOING — After Hardening

### Monthly (paste in fresh Sonnet session):
```
Monthly maintenance sweep:
1. npm audit / pip-audit — new vulns?
2. Any dependencies 2+ major versions behind?
3. Error tracker: top 5 errors this month?
4. Uptime: incidents I missed?
5. Cost audit — anything surprising?

Founder Brief format.
```

### Quarterly:
- Test backup restoration on throwaway environment (Opus)
- Rotate all API keys and secrets
- Re-read `/docs/README.md` and update anything stale
- Redo Phase 3c (OWASP review) — new code = new risks (Opus)

---

*Hardening Runbook v2.1 | 2026-04-15 | Settings-first per phase*
