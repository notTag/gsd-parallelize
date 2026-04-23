---
name: gsd-parallelize
description: "Analyze .planning/ roadmap and split phases into non-dependent workstreams for parallel engineer handoff. Dry-run by default; --apply materializes workstreams via gsd-sdk."
argument-hint: "[--apply] [--scope roadmap|backlog|all]"
allowed-tools:
  - Read
  - Bash
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

# /gsd-parallelize

Split a GSD project's phases into non-dependent **workstreams** so multiple engineers (human or AI) can work in parallel without merge hell.

## Flags

- `--apply` — Materialize workstreams. Without it, skill runs dry-run and writes `PARALLELIZATION.md` only.
- `--scope` — `roadmap` (active milestone), `backlog` (999.x items), or `all` (default).

v2-reserved (not implemented in v1): `--split=skip|auto|recommend` for sub-phase splitting.

## Preconditions

1. Project has `.planning/` with an active milestone (`.planning/milestones/v*/ROADMAP.md`).
2. `gsd-sdk` is installed and on PATH (`which gsd-sdk`).
3. Working tree is clean. Abort if `git status --porcelain` is non-empty — ask user to commit/stash first.

If any precondition fails, surface the blocker and stop.

## Step 1 — Gather phases

Determine the active milestone:
- Read `.planning/STATE.md` or glob `.planning/milestones/v*/ROADMAP.md` (pick highest version).

Build the phase inventory. For each phase:
- `id` — phase number (e.g., `3`, `5.1`, `999.2`)
- `title` — from ROADMAP.md row
- `path` — phase directory (e.g., `.planning/milestones/v1/phases/3-auth-middleware/`)
- `deps` — explicit `Depends on:` entries from ROADMAP.md and PLAN.md
- `files` — union of file paths mentioned in PLAN.md under "Files to create/modify", "Files", "Artifacts", or "New files" headings. Extract with grep; normalize relative paths.

Scope filter:
- `roadmap` → only non-backlog phases (ids not starting with `999`)
- `backlog` → only `999.*` phases
- `all` → both

If a phase has no PLAN.md, flag it `unplanned` — include in output but exclude from file-overlap analysis (deps only).

## Step 2 — Build dependency graph

Two edge types:

- **Hard edge** (A → B): B has A in its `Depends on`. B cannot start until A ships.
- **Soft edge** (A ↔ B): A and B touch ≥1 of the same file. No ordering constraint, but parallel development will conflict.

Both edge types collapse phases into the **same workstream**. Rationale: soft-edge phases serialize inside the workstream; hard-edge chains serialize naturally via deps.

Algorithm:
1. Union-find across all edges → connected components = workstreams.
2. Within each component, topologically sort by hard edges → phase execution order.
3. Across components, detect cross-component hard edges — these are rare (soft edges should have merged them) but possible when e.g. ws-api exposes an interface ws-ui consumes. Record as **suggested merge order** (producer workstream merges before consumer).

## Step 3 — Name workstreams

For each component, derive a name:
- Pull keywords from phase titles (drop common words: "add", "update", "fix", "implement").
- Slug: lowercase, hyphenated, ≤24 chars. Example phases ["Auth middleware", "Auth tests", "Auth token refresh"] → `auth`.

Validate names against existing workstreams (`gsd-sdk query workstream.list`). If collision, append `-2`, `-3`, etc.

Interactive override: if stdin is interactive, show proposed names + phase groupings and ask user to confirm/rename via AskUserQuestion. Skip in `--auto` / non-interactive contexts.

## Step 4 — Emit PARALLELIZATION.md

Write to `.planning/PARALLELIZATION.md`. Structure:

```markdown
# Parallelization Plan — <milestone vX>

Generated: <ISO timestamp>
Scope: <roadmap|backlog|all>

## Workstreams

### <name> (<N> phases)

**Phases (execution order):**
1. Phase 3 — Auth middleware
2. Phase 7 — Auth token refresh

**Hard deps (internal):** 3 → 7
**Soft conflicts (shared files):**
- `src/auth/middleware.ts` — phases 3, 7

**Rationale:** <one-line why these are grouped>

---

### <next workstream>
...

## Suggested Merge Order

1. ws-auth (no cross-workstream deps)
2. ws-api (depends on ws-auth's `AuthContext` interface — phase 3)
3. ws-ui (depends on ws-api endpoints — phase 8)

## Sub-phase Split Recommendations

- **Phase 4 (12 plans)** — consider splitting via `/gsd-insert-phase`. Candidates: plans 1-5 are file-disjoint from plans 6-12.

## Unplanned Phases (no PLAN.md)

- Phase 9 — skipped from file-overlap analysis, grouped by explicit deps only.
```

Commit this file in dry-run mode — it's useful on its own.

## Step 5 — Apply (only if `--apply`)

**Before mutating anything, show a diff preview** and ask explicit confirmation via AskUserQuestion. Abort on "no".

For each workstream group:

1. **Create workstream:**
   ```bash
   gsd-sdk query workstream.create <name> --raw --cwd "$CWD"
   ```
   This creates `.planning/workstreams/<name>/` with empty STATE.md, ROADMAP.md, REQUIREMENTS.md, phases/.

2. **Copy phase dirs** (copy, not move — canonical source stays in milestone):
   ```bash
   cp -R .planning/milestones/v<X>/phases/<phase-dir> .planning/workstreams/<name>/phases/
   ```

3. **Populate workstream ROADMAP.md** — subset of milestone ROADMAP containing only this workstream's phases, in topo order.

4. **Populate workstream STATE.md** — initial state pointing at first phase.

5. **Copy shared REQUIREMENTS.md** — from milestone, filtered to requirements touched by workstream's phases (best-effort by grepping requirement IDs referenced in phase PLAN.md files).

After all workstreams created:
- Run `git add .planning/workstreams/ .planning/PARALLELIZATION.md`
- Commit: `gsd-parallelize: split v<X> into <N> workstreams (<names>)`
- Do **not** push. Let user review + push themselves.

## Step 6 — Report

Print a summary table:

| Workstream | Phases | Files | Hard deps in | Merge order |
|---|---|---|---|---|
| auth | 3, 7 | 4 | - | 1st |
| api | 4, 8 | 6 | auth#3 | 2nd |

Print handoff instructions:
> Push to remote: `git push origin main`
> Engineers clone repo and run: `/gsd-resume-work --ws <name>`

## Constraints

- **Never** run `workstream.create` without `--apply`.
- **Never** delete phases from the canonical milestone dir.
- **Never** push commits. Local commit only; user decides when to push.
- If `gsd-sdk workstream.create` fails, stop the loop — do not continue creating siblings. Report partial state so user can clean up manually.
- Dry-run output (`PARALLELIZATION.md`) must be **idempotent** — running dry-run twice produces the same file.

## Non-goals (v1)

- Sub-phase splitting (see `--split` in v2).
- Engineer assignment (pure grouping; humans pick who works on what).
- Cross-repo / multi-workspace parallelization (this operates on one repo's `.planning/`).
- Auto-push to remote.
- Conflict resolution at merge time — skill only prevents parallel conflicts upfront.
