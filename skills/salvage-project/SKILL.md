---
name: salvage-project
description: Use when a project has gone down a bad path (wrong stack, over-engineered, incomplete requirements) and the developer wants to start over simpler without losing what they learned. Recovers the real requirements and emits a portable spec bundle to rebuild. Also runs on session start to detect and accept an existing salvage/ bundle. Triggers on "start over", "salvage this project", "rebuild simpler".
---

# Salvage Project

Start over, simpler — without losing what you learned.

This skill has two modes. Which one runs is decided by **mode detection**, below.
Run mode detection first, every session, before doing anything else.

## Mode detection (run first, every session)

Check the working directory for the file `salvage/manifest.yaml`.

- **Present** → a salvage bundle is already here. Run **Accept mode (Act 3)** —
  set up a fresh project from the bundle. See that section below.
- **Absent** → no bundle. Run **Salvage mode (Act 1)** — recover the real
  requirements from a broken project and emit a bundle. Salvage mode is
  triggered by the `/salvage` command or by natural language such as "start
  over", "salvage this project", or "rebuild simpler".

Do not mix modes. Decide once, then follow that act to the end.

---

## Salvage mode (Act 1)

You are in a project that went down a bad path. Your job is to recover the
*real* requirements (now that the developer knows more) and write a portable
spec bundle that a fresh agent can rebuild from — on the simplest stack the
developer can actually maintain.

**Two rules of discipline for the whole act:**

1. **One question at a time.** Never batch questions. If a topic needs more
   exploration, break it into separate messages. Prefer multiple-choice when it
   makes a question easier to answer.
2. **Write NOTHING into the project until Stage 4.** Stages 1–3 only *read and
   ask*. No files, no edits. The bundle is written once, at the end.

The four stages run in order, pausing for the developer between each.

### Stage 1 — Intake (what went wrong)

Ask these one at a time. Listen for the real requirement hiding behind each
complaint.

1. What is this app *supposed* to do, in one sentence?
2. Which features fought you hardest? What broke, or wouldn't scale?
3. Where did the previous agent resist a change you wanted?
4. **Maintenance ceiling (HARD constraint).** What languages and tools are you
   comfortable owning long-term? Are you OK with a build step? With Docker? With
   async code? Be concrete.

The maintenance ceiling is not a preference — it is a **hard cap on Stage 3**.
No rung you choose later may exceed what the developer answered here, no matter
how tempting. Record the ceiling verbatim; you will weigh every architecture
decision against it.

### Stage 2 — Code archaeology (what it actually does)

Read the existing code to extract the *real* requirements. Your goal is to
separate two things that are tangled together in the old project:

- **Genuine behavior (KEEP)** — what the app actually needs to do for its users.
- **Incidental tech decisions (DROP)** — choices the last build made that aren't
  requirements at all (a framework, a database, a pattern).

Produce two drafts and show them to the developer:

1. A **feature inventory** — the "keep" list, in plain behavioral terms.
2. A list of **complexity smells** — places where the old build climbed higher
   than any requirement justified. For example: "websockets for a dashboard that
   refreshes every 30s", "Postgres for fewer than 1000 rows", "a SPA for three
   static pages".

The developer confirms or corrects the inventory before you continue. Do not
move to Stage 3 until the "keep" list is agreed.

### Stage 3 — Architecture interview

Now choose the stack, one fork at a time. Invoke the **simplicity-guide** skill;
it holds one reference file per fork (data store, frontend, realtime, auth, jobs,
deploy), each structured as a ladder of rungs.

For **each fork that is relevant** to this project, run this loop:

1. Open the matching `simplicity-guide` reference.
2. Find the **lowest rung whose "Use when" the project satisfies.**
3. Climb a rung **only when a STATED requirement (from Stage 1 or the Stage 2
   keep list) matches that rung's "Climb when."** Never climb "to be safe" or
   "for scale we might need."
4. Weigh the rung's **Cost** against the Stage-1 **maintenance ceiling.** If a
   rung's cost exceeds the ceiling, you may not choose it — stay lower and tell
   the developer why.
5. Ask the developer the trade-off question, framed by their ceiling, and
   confirm the choice.
6. Record the decision by filling the rung's **Rationale template**, and cite
   the exact rung you used (for example `data-store.md#sqlite`).

Each fork ends with one recorded, rung-cited decision. These become
`tech-stack.md` in Stage 4.

### Stage 4 — Emit the bundle

Now — and only now — write the portable bundle. Create a `salvage/` directory at
the **root of the broken project** containing exactly these six files:

**`manifest.yaml`** — the detection file and greeting source. Schema:

```yaml
salvage_version: 1
created: <today's date, YYYY-MM-DD>
app_name: <one-line app name>
stack: <chosen stack, one line>
docs: [CLAUDE.md, requirements.md, tech-stack.md, design-notes.md, anti-patterns.md]
```

**`requirements.md`** — the recovered "keep" feature inventory from Stage 2.
Behavioral requirements only; no tech decisions.

**`tech-stack.md`** — the chosen stack, with the filled rationale (from Stage 3)
for each decision, each citing its simplicity-guide rung.

**`design-notes.md`** — how the pieces fit together. Keep this deliberately
small; it is orientation, not a spec.

**`anti-patterns.md`** — the failed decisions from last time. For each one, state
**WHY** it failed and mark it clearly **"do NOT reintroduce."** This is the
single most valuable artifact in the bundle — it stops the fresh agent from
walking back into the same wreck.

**`CLAUDE.md`** — start from `templates/CLAUDE.md` and fill its slots from the
interview: `{{skill_level}}` (from the Stage-1 maintenance ceiling), `{{stack}}`
(from Stage 3), and `{{requirements}}` (a one-line summary of the keep list). The
fixed principles half is copied verbatim — do not soften it.

Then tell the developer exactly what to do next:

1. Create a new, empty git repository for the rebuild.
2. Copy the `salvage/` directory into it.
3. Open a fresh Claude Code session in that repo — this skill will detect the
   bundle and set the project up automatically.

## Accept mode (Act 3)

See below — added next.
