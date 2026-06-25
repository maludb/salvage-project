# salvage — Design

**Date:** 2026-06-25
**Status:** Validated design, all open items resolved — ready for implementation planning

> **salvage** — *Start over, simpler — without losing what you learned.*

A Claude Code plugin for the moment a project has gone down a bad path: wrong
tech stack, over-engineered, or built on requirements that turned out incomplete.
Instead of fighting the existing code, `salvage` recovers the *real* requirements
(now that the developer knows more) and writes a spec to rebuild on the **simplest
stack the developer can actually maintain** — which a fresh agent then implements
in a clean project.

It is the inverse of the `handoff` plugin: handoff says *"continue this work"*;
salvage says *"this approach failed — keep the intent, drop the decisions."*

---

## Design decisions (the five that shaped this)

1. **Deliverable = documents only.** salvage produces a portable, agent-agnostic
   spec bundle. It does not scaffold or implement. The doc is the product.
2. **Architecture knowledge = hybrid.** A bundled reference supplies principles
   and a "default ladder," but the per-fork questioning is conversational.
3. **One guided command + a natural-language-triggerable skill.** `/salvage` is
   the spine; saying "use the salvage-project skill / help me start over" also
   triggers it.
4. **Name = `salvage`.** The old code isn't trash — we recover the valuable
   parts (the real requirements) and rebuild clean. The wreck taught us something.
5. **The developer's maintenance ceiling is a hard input.** The intake interview
   captures what the developer can actually own, and that *caps* the stack.

---

## Lifecycle (three acts)

**Act 1 — in the broken project (`/salvage`).** Interview → code archaeology →
architecture decision → emit a portable `salvage/` bundle. Nothing is written into
a new project yet; the bundle is self-contained.

**Act 2 — the move.** Developer creates a clean empty repo and drops the
`salvage/` bundle in.

**Act 3 — in the new project (`accept`).** The same `salvage-project` skill
auto-detects the bundle on session start (the way `handoff` detects `HANDOFF.md`),
writes the canonical project docs, and transitions into the build — delegating to
`superpowers` if present, or standing alone if not.

The salvage interview **is** the brainstorming phase. By the time Act 1 finishes,
the design is validated — so Act 3 does *not* re-brainstorm and does *not* reinvent
plan-writing/execution. It drops straight into them.

---

## Plugin structure

```
salvage/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── salvage.md              # /salvage — runs the guided flow
├── skills/
│   ├── salvage-project/
│   │   └── SKILL.md            # the spine; natural-language triggerable;
│   │                           # mode-switches on whether a bundle is present
│   └── simplicity-guide/
│       ├── SKILL.md            # bundled reference (the "spine of opinions")
│       └── references/
│           ├── frontend.md     # server-rendered vs HTMX vs SPA
│           ├── data-store.md   # files vs SQLite vs Postgres
│           ├── realtime.md     # manual refresh vs polling vs SSE vs websockets
│           ├── auth.md         # framework sessions vs library vs third-party
│           ├── jobs.md         # inline vs cron vs queue
│           └── deploy.md       # one box vs containers vs orchestration
├── templates/
│   └── CLAUDE.md               # curated principles + project-specific slots
└── README.md
```

Two skills: one **process** skill (`salvage-project`) that runs the
interview-to-spec flow and the accept flow, and one **reference** skill
(`simplicity-guide`) it consults during the architecture stage.

---

## Act 1 — the `/salvage` flow

Four stages, pausing for the developer between each. Writes nothing into the
project until the final stage — it only reads and asks. Questions are asked one
at a time (borrowing `brainstorming`'s discipline).

### Stage 1 — Intake (what went wrong)
- What is this app *supposed* to do, in one sentence?
- Which features fought you hardest? What broke or wouldn't scale?
- Where did the previous agent resist a change you wanted?
- **Maintenance ceiling (hard constraint):** what languages/tools are you
  comfortable owning? OK with a build step? Docker? Async? This answer *caps*
  the stack in Stage 3.

### Stage 2 — Code archaeology (what it actually does)
- Review the existing code to extract the *real* requirements — separating
  genuine behavior (keep) from incidental tech decisions (drop).
- Produce a draft feature inventory + a list of complexity smells (e.g.
  "websockets for a dashboard that refreshes every 30s", "Postgres for <1000
  rows").
- Developer confirms/corrects the inventory.

### Stage 3 — Architecture interview
- For each relevant fork, consult the matching `simplicity-guide` reference,
  then ask the developer the trade-off question framed by their Stage-1 ceiling.
- The default ladder starts at the simplest rung and climbs only when a *stated*
  requirement forces it.
- Every decision recorded with rationale ("chose SQLite — single writer, <10k
  rows, you wanted zero ops").

### Stage 4 — Emit the bundle
- Write the portable `salvage/` bundle (the five docs below) to the broken
  project's root.
- Tell the developer exactly what to do next: create an empty repo, copy
  `salvage/` in, open a fresh session.

---

## Act 3 — the `accept` flow

Same `salvage-project` skill, different mode. On session start it checks for a
`salvage/` bundle in the working directory.

1. **Detect & confirm.** If a bundle exists and the repo is otherwise empty:
   *"Found a salvage spec to rebuild [app] on [stack]. Want me to set up the
   project?"* If the repo already has substantial code, warn first — don't
   clobber an in-progress project.

2. **Check for superpowers.**
   - Installed: *"superpowers is available — I'll use its plan/execute workflow
     after setup."*
   - Not installed: recommend it (`/plugin install superpowers`) for the
     structured build workflow, but make clear the docs work with any agent and
     offer to continue without.

3. **Write the canonical docs** (from the bundle):
   - `CLAUDE.md` — from `templates/CLAUDE.md`: curated principles + filled slots.
   - `requirements.md` — the recovered feature inventory (the "keep" list).
   - `tech-stack.md` — chosen stack + rationale for each decision.
   - `design-notes.md` — how the pieces fit, kept deliberately small.
   - `anti-patterns.md` — the failed decisions from last time, with *why*,
     marked "do NOT reintroduce." The single most valuable artifact.

4. **Transition to the build.**
   - Superpowers present: hand directly to `superpowers:writing-plans` (the
     bundle is the validated design), then `executing-plans`.
   - Absent: `CLAUDE.md` carries a built-in fallback — "build feature-by-feature
     from `requirements.md`, simplest-first, confirm each before moving on."

The seam is clean: salvage owns recovery and setup; superpowers (or a plain
agent) owns the build. superpowers is an *optional accelerator*, never a hard
dependency.

---

## Bundled `templates/CLAUDE.md` (principles half)

The top half is fixed and copied verbatim — the plugin's spine. The bottom half
has `{{skill_level}}` / `{{stack}}` / `{{requirements}}` slots filled from the
bundle. Curated, not regenerated, so the constraints can't be softened on the
fly.

```markdown
# Project Principles (from salvage — do not weaken these)

## Prime directive
Build the simplest thing that fully meets a stated requirement. Complexity
must be *earned* by a real, written requirement — never added "to be safe,"
"for scale we might need," or "because it's best practice."

## The complexity ladder
For every technical choice, start at the lowest rung. Climb ONLY when a
requirement in requirements.md forces it, and record why in tech-stack.md.
- Data:     files → SQLite → Postgres
- Frontend: server-rendered HTML → HTML + sprinkles (HTMX) → SPA framework
- Updates:  manual refresh → polling → SSE → websockets
- Auth:     framework sessions → library → third-party identity service
- Jobs:     do it inline → cron → a real queue
- Deploy:   one box / one process → containers → orchestration

## Maintainability ceiling
This codebase must stay maintainable by a developer at this level: {{skill_level}}.
Do not introduce tools, patterns, or abstractions above that line — no clever
metaprogramming, no premature frameworks, no build steps the owner can't debug.

## When you disagree
If you think something here blocks a genuine need, STOP and ask the developer.
Do not silently work around these rules — that is exactly what failed last time
(see anti-patterns.md).

## How to build
Work feature-by-feature from requirements.md, simplest-first. Confirm each
feature works before starting the next. Prefer boring, obvious code over
short or clever code.
```

---

## YAGNI / explicitly out of scope (for now)

- No scaffolding or code generation in Act 1 (docs only).
- No separate per-stage commands (`:intake`, `:review`, …) — stages live inside
  the one `/salvage` flow until a need proves otherwise.
- No salvage-built plan writer/executor — reuse `superpowers`.
- No hard dependency on `superpowers` — the bundle stands alone.

---

## Resolved implementation decisions

*(These four were open during design; resolved 2026-06-25, ready for planning.)*

### 1. Bundle format & detection — `salvage/` dir + `manifest.yaml`
The bundle is a `salvage/` directory holding the five `.md` docs **plus** a small
`manifest.yaml`:

```yaml
salvage_version: 1
created: 2026-06-25
app_name: <one-line app name>
stack: <chosen stack, one line>
docs: [CLAUDE.md, requirements.md, tech-stack.md, design-notes.md, anti-patterns.md]
```

`accept` detects a bundle by the presence of `salvage/manifest.yaml` (not by the
doc filenames, which can collide with a developer's own files). The manifest also
feeds the accept greeting's "[app] on [stack]" line without parsing prose.

### 2. `simplicity-guide` reference structure — templated rungs
Every reference file (`frontend.md`, `data-store.md`, …) follows one identical
template: a short intro, then the ladder as **rung sections with anchored
headings**. Each rung carries four fixed fields:
- **Use when** — conditions that make this rung sufficient.
- **Climb when** — the *stated requirement* that forces the next rung up.
- **Cost** — what this rung costs the maintainer (the ceiling check).
- **Rationale template** — a fill-in line, e.g. `"chose SQLite — {single writer},
  {<10k rows}, {you wanted zero ops}"`.

The Stage-3 interview cites a precise rung (e.g. `data-store.md#sqlite`) and lifts
the rationale template straight into `tech-stack.md`. One uniform schema → the
Stage-3 loop is one routine, not six special cases.

### 3. superpowers detection — glob, then ask if absent
`accept` globs the plugins dir (e.g. `~/.claude/plugins/**/superpowers/**/SKILL.md`,
plus config/manifest if present).
- **Found** → "superpowers is available — I'll use its plan/execute workflow."
- **Not found** → do **not** assert absence; ask: "I don't see superpowers — if
  you have it, tell me; otherwise I'd recommend `/plugin install superpowers`, or
  I can continue standalone." Only the positive claim is made with confidence; the
  negative degrades to a question, eliminating false negatives from non-standard
  install paths.

### 4. `accept` safety check — allowlist emptiness + per-file no-clobber
Two separate guards:
1. **Fresh-target check.** Repo is "empty enough" if — ignoring `.git/`, the
   `salvage/` bundle, and an inert allowlist (`README*`, `LICENSE*`, `.gitignore`,
   `.editorconfig`) — nothing else remains. Any real source files → not fresh.
2. **Not fresh** → list what was found and require explicit confirmation before
   setup ("This repo already has code (X, Y, Z) — set up anyway?").
3. **Per-file no-clobber, always** → before writing each canonical doc, if it
   already exists, stop and ask (offer `.bak` backup); never overwrite blindly,
   even in a "fresh" repo (a stray `CLAUDE.md` is not on the inert allowlist).

The clean-repo happy path runs with just the greeting; every riskier case
escalates to a confirm.
