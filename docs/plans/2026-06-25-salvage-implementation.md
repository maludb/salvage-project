# salvage Plugin Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build the `salvage` Claude Code plugin — a guided flow that recovers the real requirements from a project that went down a bad path and emits a portable, agent-agnostic spec bundle to rebuild on the simplest maintainable stack.

**Architecture:** The plugin is a set of **prompt/instruction artifacts**, not executable code. A thin `/salvage` command triggers the `salvage-project` process skill, which runs a 4-stage interview (Act 1) consulting the `simplicity-guide` reference skill, then emits a `salvage/` bundle. The same `salvage-project` skill, in "accept" mode (Act 3), detects a bundle in a fresh repo and writes canonical project docs, optionally handing off to `superpowers`.

**Tech Stack:** Markdown skills/commands, JSON plugin manifest, YAML bundle manifest. No runtime code. Validation is done with `python3` one-liners (JSON/YAML parse, frontmatter checks) and structured manual dry-runs — these are the "tests."

**Source of truth:** [docs/plans/2026-06-25-salvage-design.md](2026-06-25-salvage-design.md) — the validated design with all four open items resolved. Read it before starting.

---

## How "testing" works in this plan

This plugin has no unit-testable functions. Each task's verification is one of:

- **Parse check** — the file is valid JSON/YAML/frontmatter (runnable, real pass/fail).
- **Structure check** — required sections/keys/anchors exist (`grep`, runnable).
- **Dry-run check** — a written walkthrough where you (the implementing agent) read the artifact as if you were the agent executing it, against a concrete scenario, and confirm the instructions are unambiguous and produce the intended output. This is a manual gate; record the result in the commit body.

Keep commits frequent (one per task). DRY across the six reference files (one template, six fills). YAGNI — build only what the design specifies.

---

## Task 0: Verify starting state

**Files:** none (inspection only).

**Step 1:** Confirm the design doc exists and the tree is clean.

Run: `git status --porcelain && ls docs/plans/`
Expected: no uncommitted changes; `2026-06-25-salvage-design.md` and `2026-06-25-salvage-implementation.md` present.

**Step 2:** Confirm `python3` is available for validation steps.

Run: `python3 --version`
Expected: prints a 3.x version.

No commit (inspection only).

---

## Task 1: Plugin manifest & directory skeleton

**Files:**
- Create: `.claude-plugin/plugin.json`

**Step 1: Write the manifest**

```json
{
  "name": "salvage",
  "version": "0.1.0",
  "description": "Start over, simpler — recover the real requirements from a project that went down a bad path and emit a portable spec to rebuild on the simplest maintainable stack.",
  "author": {
    "name": "Edward Honour"
  },
  "keywords": ["rewrite", "rebuild", "simplicity", "requirements", "spec"]
}
```

**Step 2: Verify it parses**

Run: `python3 -c "import json; d=json.load(open('.claude-plugin/plugin.json')); assert d['name']=='salvage'; print('ok', d['version'])"`
Expected: `ok 0.1.0`

**Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add salvage plugin manifest"
```

---

## Task 2: simplicity-guide reference template (one file, to be copied six times)

This establishes the uniform rung schema (design decision #2) before filling all six. Build `data-store.md` first as the canonical exemplar, validate the schema, then Task 3 replicates it.

**Files:**
- Create: `skills/simplicity-guide/references/data-store.md`

**Step 1: Write the reference using the fixed template**

Every reference file has: a one-line intro, then one `## ` section per rung. Each rung section contains exactly four bolded fields: **Use when**, **Climb when**, **Cost**, **Rationale template**. Rung headings are anchorable (lowercase, hyphenated) so the interview can cite `data-store.md#sqlite`.

```markdown
# Data store

The ladder for where your data lives. Start at files; climb only when a stated
requirement forces it.

## flat-files

Plain files on disk (JSON, CSV, Markdown, SQLite-less).

- **Use when:** data is small, read-mostly, and single-process; you can hold the
  whole dataset in memory; no concurrent writers.
- **Climb when:** you need concurrent writes, queries across records, or
  transactional integrity.
- **Cost:** none beyond the filesystem. No schema, no migrations, no server.
- **Rationale template:** `"chose flat files — {small read-mostly data},
  {single process}, {no query needs}"`

## sqlite

A single-file embedded relational database.

- **Use when:** you need queries/indexes/transactions but have a single writer
  (or low write concurrency) and the dataset fits on one box.
- **Climb when:** you need many concurrent writers, network access from multiple
  hosts, or horizontal scale.
- **Cost:** a SQL schema and migrations to own; still zero ops (no server).
- **Rationale template:** `"chose SQLite — {single writer}, {<Nk rows},
  {you wanted zero ops}"`

## postgres

A networked relational database server.

- **Use when:** multiple hosts/processes write concurrently, you need it as a
  shared service, or scale exceeds a single file.
- **Climb when:** (top of this ladder) — sharding/replication is a separate,
  later decision driven by measured load.
- **Cost:** a server to run, back up, secure, and upgrade. Real ops burden —
  check it against the maintainer ceiling.
- **Rationale template:** `"chose Postgres — {N concurrent writers},
  {shared service}, {scale beyond one box}"`
```

**Step 2: Verify structure (intro + three rungs, four fields each)**

Run:
```bash
python3 - <<'PY'
import re
t=open('skills/simplicity-guide/references/data-store.md').read()
rungs=re.findall(r'^## (.+)$', t, re.M)
assert rungs==['flat-files','sqlite','postgres'], rungs
for f in ['Use when','Climb when','Cost','Rationale template']:
    assert t.count('**%s:**'%f)==3, (f, t.count('**%s:**'%f))
print('ok', rungs)
PY
```
Expected: `ok ['flat-files', 'sqlite', 'postgres']`

**Step 3: Commit**

```bash
git add skills/simplicity-guide/references/data-store.md
git commit -m "feat: add data-store simplicity reference (canonical rung template)"
```

---

## Task 3: Remaining five simplicity-guide references

Replicate the Task-2 schema for the other five ladders from the design's complexity ladder. Same four fields per rung; rungs taken verbatim from the design.

**Files:**
- Create: `skills/simplicity-guide/references/frontend.md` — rungs: `server-rendered-html`, `html-plus-htmx`, `spa-framework`
- Create: `skills/simplicity-guide/references/realtime.md` — rungs: `manual-refresh`, `polling`, `sse`, `websockets`
- Create: `skills/simplicity-guide/references/auth.md` — rungs: `framework-sessions`, `auth-library`, `third-party-identity`
- Create: `skills/simplicity-guide/references/jobs.md` — rungs: `inline`, `cron`, `queue`
- Create: `skills/simplicity-guide/references/deploy.md` — rungs: `one-box`, `containers`, `orchestration`

**Step 1:** For each file, follow the exact template from Task 2: one-line intro, one `## rung-anchor` section per rung, each with **Use when** / **Climb when** / **Cost** / **Rationale template**. Content comes from the design's ladder descriptions and parenthetical guidance (e.g. realtime: "manual refresh → polling → SSE → websockets"; the design's smell example "websockets for a dashboard that refreshes every 30s" informs the websockets **Climb when**).

**Step 2: Verify every reference conforms to the schema**

Run:
```bash
python3 - <<'PY'
import re, glob
expected={
 'frontend':['server-rendered-html','html-plus-htmx','spa-framework'],
 'realtime':['manual-refresh','polling','sse','websockets'],
 'auth':['framework-sessions','auth-library','third-party-identity'],
 'jobs':['inline','cron','queue'],
 'deploy':['one-box','containers','orchestration'],
 'data-store':['flat-files','sqlite','postgres'],
}
fields=['Use when','Climb when','Cost','Rationale template']
for p in glob.glob('skills/simplicity-guide/references/*.md'):
    name=p.split('/')[-1][:-3]
    t=open(p).read()
    rungs=re.findall(r'^## (.+)$', t, re.M)
    assert rungs==expected[name], (name, rungs)
    for f in fields:
        assert t.count('**%s:**'%f)==len(rungs), (name, f)
print('ok: all 6 references conform')
PY
```
Expected: `ok: all 6 references conform`

**Step 3: Commit**

```bash
git add skills/simplicity-guide/references/
git commit -m "feat: add remaining five simplicity references (frontend, realtime, auth, jobs, deploy)"
```

---

## Task 4: simplicity-guide SKILL.md

**Files:**
- Create: `skills/simplicity-guide/SKILL.md`

**Step 1: Write the skill**

Frontmatter `name` + `description` are required by Claude Code. This is the *reference* skill — it explains the ladder philosophy and tells the reader (the salvage interview) how to consult and cite the references.

```markdown
---
name: simplicity-guide
description: Reference for choosing the simplest stack that meets a stated requirement. Use during the salvage architecture interview to frame each technical fork and record a rationale. Covers data store, frontend, realtime, auth, jobs, and deploy.
---

# Simplicity Guide

The spine of opinions behind salvage. For every technical choice, start at the
lowest rung of its ladder and climb ONLY when a written requirement forces it.

## The ladders

Each fork has a reference file under `references/`, structured identically:
one `## rung` section per rung, each with **Use when**, **Climb when**, **Cost**,
and a **Rationale template**.

- Data store → `references/data-store.md` (flat files → SQLite → Postgres)
- Frontend → `references/frontend.md` (server-rendered → HTMX → SPA)
- Realtime → `references/realtime.md` (manual refresh → polling → SSE → websockets)
- Auth → `references/auth.md` (framework sessions → library → third-party)
- Jobs → `references/jobs.md` (inline → cron → queue)
- Deploy → `references/deploy.md` (one box → containers → orchestration)

## How to consult during the interview

For each relevant fork:
1. Open the matching reference.
2. Find the lowest rung whose **Use when** the project satisfies.
3. Check the **Climb when** — climb one rung only if a *stated* requirement
   matches it. Never climb "to be safe."
4. Weigh **Cost** against the developer's maintenance ceiling (from Stage 1).
5. Record the decision in `tech-stack.md` by filling the rung's **Rationale
   template**, citing the rung (e.g. `data-store.md#sqlite`).
```

**Step 2: Verify frontmatter parses and required keys exist**

Run:
```bash
python3 - <<'PY'
t=open('skills/simplicity-guide/SKILL.md').read()
assert t.startswith('---'), 'missing frontmatter'
fm=t.split('---',2)[1]
assert 'name: simplicity-guide' in fm
assert 'description:' in fm
print('ok')
PY
```
Expected: `ok`

**Step 3: Commit**

```bash
git add skills/simplicity-guide/SKILL.md
git commit -m "feat: add simplicity-guide reference skill"
```

---

## Task 5: templates/CLAUDE.md (principles + slots)

**Files:**
- Create: `templates/CLAUDE.md`

**Step 1:** Copy the principles half **verbatim** from the design doc's
"Bundled `templates/CLAUDE.md`" section (lines under `# Project Principles`),
keeping the `{{skill_level}}`, `{{stack}}`, `{{requirements}}` slots. Append the
"## How to build" fallback block exactly as in the design (this is the
superpowers-absent path).

**Step 2: Verify the fixed principles and all three slots are present**

Run:
```bash
python3 - <<'PY'
t=open('templates/CLAUDE.md').read()
for s in ['{{skill_level}}','{{stack}}','{{requirements}}']:
    assert s in t, s
for h in ['Prime directive','The complexity ladder','Maintainability ceiling',
          'When you disagree','How to build']:
    assert h in t, h
print('ok')
PY
```
Expected: `ok`

**Step 3: Commit**

```bash
git add templates/CLAUDE.md
git commit -m "feat: add bundled CLAUDE.md template with principles and slots"
```

---

## Task 6: salvage-project SKILL.md — the spine (Act 1 emit + Act 3 accept)

This is the core artifact. It mode-switches on whether a `salvage/` bundle is
present in the working directory. Build it in two passes (6a emit, 6b accept) but
it is one file; commit after each pass.

**Files:**
- Create: `skills/salvage-project/SKILL.md`

### Step 1 (pass 6a): Frontmatter + mode-switch + Act 1 (the `/salvage` emit flow)

Write the frontmatter and the Act-1 flow. The description must be
natural-language triggerable ("help me start over", "use the salvage-project
skill") per design decision #3.

Frontmatter:
```markdown
---
name: salvage-project
description: Use when a project has gone down a bad path (wrong stack, over-engineered, incomplete requirements) and the developer wants to start over simpler without losing what they learned. Recovers the real requirements and emits a portable spec bundle to rebuild. Also runs on session start to detect and accept an existing salvage/ bundle. Triggers on "start over", "salvage this project", "rebuild simpler".
---
```

Body must specify, in order:

1. **Mode detection (run first, every session):** check for `salvage/manifest.yaml`
   in the working directory.
   - Present → **accept mode** (go to Act 3, pass 6b).
   - Absent → **salvage mode** (Act 1 below), triggered by `/salvage` or natural
     language.

2. **Act 1 — four stages, one question at a time, write nothing until Stage 4**
   (borrow brainstorming's discipline). Document each stage from the design:
   - **Stage 1 Intake** — the four intake questions, including the maintenance
     ceiling as a hard constraint that caps Stage 3.
   - **Stage 2 Archaeology** — review existing code, extract real requirements,
     separate genuine behavior (keep) from incidental tech (drop), produce a draft
     feature inventory + complexity smells, developer confirms.
   - **Stage 3 Architecture interview** — for each relevant fork, invoke
     `simplicity-guide`, find the lowest satisfying rung, climb only on a stated
     requirement, record each decision with the rung's filled rationale template.
   - **Stage 4 Emit the bundle** — write the `salvage/` directory:
     - `manifest.yaml` (schema from design decision #1: `salvage_version: 1`,
       `created`, `app_name`, `stack`, `docs` list).
     - `requirements.md` — the recovered "keep" inventory.
     - `tech-stack.md` — chosen stack + filled rationale per decision.
     - `design-notes.md` — how pieces fit, kept small.
     - `anti-patterns.md` — failed decisions + *why*, marked "do NOT reintroduce."
     - `CLAUDE.md` — `templates/CLAUDE.md` with slots filled from the interview.
     Then tell the developer exactly what to do next: create an empty repo, copy
     `salvage/` in, open a fresh session.

**Step 2 (verify pass 6a):**

Run:
```bash
python3 - <<'PY'
t=open('skills/salvage-project/SKILL.md').read()
assert t.startswith('---') and 'name: salvage-project' in t.split('---',2)[1]
for k in ['salvage/manifest.yaml','Stage 1','Stage 2','Stage 3','Stage 4',
          'maintenance ceiling','simplicity-guide','anti-patterns.md']:
    assert k in t, k
print('ok 6a')
PY
```
Expected: `ok 6a`

**Step 3 (commit pass 6a):**
```bash
git add skills/salvage-project/SKILL.md
git commit -m "feat: add salvage-project skill — Act 1 emit flow"
```

### Step 4 (pass 6b): Act 3 — the accept flow

Append the accept-mode section to the same file. Encode design decisions #1, #3,
#4:

1. **Detect & confirm.** Bundle detected via `salvage/manifest.yaml`. Read the
   manifest's `app_name`/`stack` for the greeting:
   *"Found a salvage spec to rebuild [app_name] on [stack]. Want me to set up the
   project?"*

2. **Safety check (decision #4) BEFORE writing anything:**
   - Fresh-target check: repo is "empty enough" if, ignoring `.git/`, `salvage/`,
     and the inert allowlist (`README*`, `LICENSE*`, `.gitignore`,
     `.editorconfig`), nothing else remains.
   - Not fresh → list what was found, require explicit confirmation before setup.
   - Per-file no-clobber: before writing each canonical doc, if it already exists,
     stop and ask (offer `.bak` backup); never overwrite blindly.

3. **superpowers detection (decision #3):** glob
   `~/.claude/plugins/**/superpowers/**/SKILL.md` (+ config/manifest if present).
   - Found → "superpowers is available — I'll use its plan/execute workflow."
   - Not found → do NOT assert absence; ask whether it's installed, else recommend
     `/plugin install superpowers` or offer to continue standalone.

4. **Write canonical docs** from the bundle into repo root: `CLAUDE.md`,
   `requirements.md`, `tech-stack.md`, `design-notes.md`, `anti-patterns.md`.

5. **Transition to build:**
   - superpowers present → hand to `superpowers:writing-plans` (bundle is the
     validated design), then `executing-plans`. Do NOT re-brainstorm.
   - absent → follow the `CLAUDE.md` "How to build" fallback (feature-by-feature
     from `requirements.md`, simplest-first, confirm each).

**Step 5 (verify pass 6b):**

Run:
```bash
python3 - <<'PY'
t=open('skills/salvage-project/SKILL.md').read()
for k in ['Found a salvage spec','empty enough','.bak',
          'superpowers','writing-plans','requirements.md','anti-patterns.md']:
    assert k in t, k
print('ok 6b')
PY
```
Expected: `ok 6b`

**Step 6 (dry-run check — the critical gate):**

Read the whole `salvage-project/SKILL.md` as if you are an agent that just opened
two scenarios. Confirm each path is unambiguous:
- **Scenario A (broken repo, no bundle):** `/salvage` → you can follow Stages 1–4
  and know exactly which files to write and where.
- **Scenario B (fresh repo with `salvage/`):** session start → you detect the
  manifest, run the safety check, detect superpowers, write the five docs, and
  hand off — without re-brainstorming.

Record the dry-run outcome (pass/notes) in the commit body.

**Step 7 (commit pass 6b):**
```bash
git add skills/salvage-project/SKILL.md
git commit -m "feat: add salvage-project skill — Act 3 accept flow

Dry-run: <pass/notes for scenarios A and B>"
```

---

## Task 7: /salvage command

**Files:**
- Create: `commands/salvage.md`

**Step 1:** Write a thin command that triggers the salvage-project skill in
salvage (Act 1) mode. Commands are markdown with optional frontmatter; keep it
minimal — the logic lives in the skill (DRY).

```markdown
---
description: Recover the real requirements from a project that went down a bad path and emit a portable spec to rebuild simpler.
---

Invoke the `salvage-project` skill in **salvage mode** (Act 1) for the current
project. Run the four-stage flow — Intake, Code archaeology, Architecture
interview, Emit the bundle — asking one question at a time and writing nothing
into the project until the final stage. Do not re-implement the flow here; follow
the skill.
```

**Step 2: Verify**

Run: `python3 -c "t=open('commands/salvage.md').read(); assert 'salvage-project' in t and 'Act 1' in t; print('ok')"`
Expected: `ok`

**Step 3: Commit**

```bash
git add commands/salvage.md
git commit -m "feat: add /salvage command"
```

---

## Task 8: README.md

**Files:**
- Create: `README.md`

**Step 1:** Write user-facing docs: the tagline ("Start over, simpler — without
losing what you learned"), the three-act lifecycle, install instructions, what
the bundle contains, and the relationship to `superpowers` (optional accelerator,
never a hard dependency) and `handoff` (the inverse). Pull framing from the design
doc's intro.

**Step 2: Verify the key concepts are documented**

Run:
```bash
python3 - <<'PY'
t=open('README.md').read().lower()
for k in ['salvage','/salvage','bundle','superpowers','simplest']:
    assert k in t, k
print('ok')
PY
```
Expected: `ok`

**Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add salvage README"
```

---

## Task 9: Full-plugin validation

**Files:** none (validation only).

**Step 1: Confirm the full structure matches the design's plugin tree**

Run:
```bash
python3 - <<'PY'
import os
required=[
 '.claude-plugin/plugin.json',
 'commands/salvage.md',
 'skills/salvage-project/SKILL.md',
 'skills/simplicity-guide/SKILL.md',
 'skills/simplicity-guide/references/frontend.md',
 'skills/simplicity-guide/references/data-store.md',
 'skills/simplicity-guide/references/realtime.md',
 'skills/simplicity-guide/references/auth.md',
 'skills/simplicity-guide/references/jobs.md',
 'skills/simplicity-guide/references/deploy.md',
 'templates/CLAUDE.md',
 'README.md',
]
missing=[p for p in required if not os.path.exists(p)]
assert not missing, missing
print('ok: all', len(required), 'files present')
PY
```
Expected: `ok: all 12 files present`

**Step 2: Re-run every parser/structure check from Tasks 1–7** (they should all
still pass). Optionally lint all JSON/YAML in one sweep.

**Step 3: End-to-end dry-run.** Walk the full lifecycle on paper:
broken project → `/salvage` → bundle emitted → (move) → fresh repo + bundle →
accept → docs written → handoff. Confirm no stage references a file or field that
doesn't exist. Record the outcome.

**Step 4: Final commit (if any validation fixups were needed)**

```bash
git add -A
git commit -m "test: validate full salvage plugin structure and lifecycle"
```

---

## Done criteria

- All 12 plugin files exist and pass their parse/structure checks.
- `salvage-project/SKILL.md` cleanly handles both modes (scenario A and B
  dry-runs pass).
- The bundle schema, reference rung schema, superpowers detection, and accept
  safety check all match the four resolved decisions in the design doc.
- `superpowers` is referenced only as an optional accelerator; the bundle and
  `CLAUDE.md` fallback stand alone.
