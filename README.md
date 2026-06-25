# salvage

**Start over, simpler — without losing what you learned.**

A Claude Code plugin for the moment a project has gone down a bad path: the
wrong tech stack, over-engineered for what it actually needs, or built on
requirements that turned out to be incomplete. Instead of fighting the existing
code, `salvage` recovers the *real* requirements — now that you know more — and
writes a portable spec to rebuild on the **simplest stack you can actually
maintain**. A fresh agent then implements that spec in a clean project.

It is the inverse of the `handoff` plugin. handoff says *"continue this work."*
salvage says *"this approach failed — keep the intent, drop the decisions."*

salvage produces **documents only**. It does not scaffold or generate code in
the broken project. The recovered spec is the product; building from it happens
later, in a fresh repo.

---

## The three-act lifecycle

salvage runs across two repositories, with you moving a bundle between them.

1. **Act 1 — in the broken project (`/salvage`).** A four-stage interview:
   - **Intake** — what the app is supposed to do, what fought you, and your
     *maintenance ceiling* (the tools you can actually own — a hard cap on the
     stack).
   - **Code archaeology** — read the existing code to separate genuine behavior
     to *keep* from incidental tech decisions to *drop*.
   - **Architecture interview** — choose the stack one fork at a time, starting
     at the simplest rung and climbing only when a stated requirement forces it.
   - **Emit the bundle** — write a self-contained `salvage/` bundle to the
     project root.

   Stages 1–3 only read and ask. Nothing is written until the final stage, and
   nothing is written into a new project yet.

2. **Act 2 — the move.** You create a clean, empty git repo and drop the
   `salvage/` bundle into it.

3. **Act 3 — in the new project (accept).** The `salvage-project` skill
   auto-detects the bundle on session start (the way `handoff` detects
   `HANDOFF.md`), confirms before touching anything, writes the canonical
   project docs to the repo root, and transitions straight into the build.

The salvage interview *is* the brainstorming phase. By the time Act 1 finishes,
the design is validated — so Act 3 does not re-brainstorm. It drops straight
into planning and building.

---

## Install

salvage installs like any Claude Code plugin (`/plugin install`). Once
installed, there are two entry points:

- **`/salvage`** — the guided command. Run it inside the broken project to start
  Act 1.
- **The `salvage-project` skill** — natural-language triggerable. Saying things
  like *"salvage this project"*, *"start over simpler"*, or *"rebuild this"*
  invokes the same flow. The skill also runs automatically on session start to
  detect an existing bundle (Act 3).

---

## What's in the bundle

Act 1 writes a `salvage/` directory containing six files:

| File | What it holds |
| --- | --- |
| `manifest.yaml` | Bundle metadata + the detection file Act 3 looks for; supplies the `[app] on [stack]` greeting. |
| `requirements.md` | The recovered "keep" inventory — behavioral requirements only, no tech decisions. |
| `tech-stack.md` | The chosen stack with rationale for each decision, each citing its simplicity-guide rung. |
| `design-notes.md` | How the pieces fit together. Deliberately small — orientation, not a spec. |
| `anti-patterns.md` | The failed decisions from last time, with *why*, marked "do NOT reintroduce." The single most valuable artifact — it stops the fresh agent from walking back into the same wreck. |
| `CLAUDE.md` | Curated build principles (the complexity ladder, maintainability ceiling, prime directive) with the project's stack, requirements, and skill level filled in. |

The bundle is portable and agent-agnostic. In Act 3, `requirements.md`,
`tech-stack.md`, `design-notes.md`, `anti-patterns.md`, and `CLAUDE.md` are
copied out to the new repo's root; `manifest.yaml` stays in `salvage/`.

---

## Relationship to superpowers

If the [`superpowers`](https://github.com/obra/superpowers) plugin is installed,
Act 3 hands the bundle to its plan/execute workflow (`writing-plans` →
`executing-plans`) for a structured build.

superpowers is an **optional accelerator, never a hard dependency.** If it isn't
installed, the bundled `CLAUDE.md` carries a built-in fallback — build
feature-by-feature from `requirements.md`, simplest-first, confirming each before
moving on. The bundle stands alone with any agent.

---

## How it's built

```
salvage/
├── .claude-plugin/plugin.json
├── commands/
│   └── salvage.md               # /salvage — runs Act 1
├── skills/
│   ├── salvage-project/         # the spine: salvage mode (Act 1) + accept mode (Act 3)
│   │   └── SKILL.md
│   └── simplicity-guide/        # bundled reference consulted during the architecture interview
│       ├── SKILL.md
│       └── references/          # data-store, frontend, realtime, auth, jobs, deploy
├── templates/
│   └── CLAUDE.md                # curated principles + project-specific slots
└── README.md
```

The `salvage-project` skill picks its mode by checking for `salvage/manifest.yaml`
in the working directory: absent → Act 1 (recover and emit); present → Act 3
(accept and set up). The `simplicity-guide` skill supplies one reference file per
architecture fork, each laid out as a ladder of rungs with **Use when**,
**Climb when**, **Cost**, and a fill-in **Rationale template**.
