---
name: gsd-parallelize
version: 0.1.1
description: "Analyze .planning/ roadmap and split phases into non-dependent workstreams for parallel engineer handoff. Dry-run by default; --apply materializes workstreams via gsd-sdk."
argument-hint: "[--apply] [--scope roadmap|backlog|all] [--analyze-partial]"
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
- `--analyze-partial` — Deep analysis pass. For every phase with a hard dep, estimate the fraction of work that can be done in parallel with its upstream phase (before the dep actually blocks). Adds two columns to `PARALLELIZATION.md`: **Unblocked scope** (what the X% entails) and **Blocked scope** (what the remaining 100-X% entails). Costs one extra analysis pass per hard-dep phase. See Step 2.5.

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

### Linear-chain detection

After the graph is built, check if every non-foundation phase has a hard dep on phase `N-1` (strict linear chain). If so, and `--analyze-partial` was **not** passed, emit a hint at the top of the dry-run report:

> **Linear chain detected.** Every phase hard-depends on its predecessor → only 1 workstream possible under strict dependency semantics. Many phases may still be partially parallelizable (e.g. scaffolding, independent compute, eval harness setup can begin before upstream ships). Re-run with `--analyze-partial` to estimate per-phase unblocked fraction and see what can be started early vs. what must wait.

This hint is informational — the skill still writes `PARALLELIZATION.md` with its strict-dep result. Do not emit the hint if `--analyze-partial` is already set, or if the graph is not a strict linear chain (some parallelism already detected).

## Step 2.5 — Partial analysis (only if `--analyze-partial`)

For each phase that has at least one hard dep, estimate its **unblocked fraction** — the percentage of phase work that can run in parallel with its upstream phase(s), before integration/eval forces a pause.

### Signal sources (ranked)

1. **PLAN.md task list** — if present, read every task and its referenced files. Best signal.
2. **SPEC.md scope + deliverables** — fallback when PLAN.md does not yet exist. Coarser estimate; flag confidence as `low`.
3. **ROADMAP.md phase summary** — last resort. Flag confidence as `very-low`.

### Classification

Classify each task (or deliverable, when only SPEC is available) into one of three buckets:

- **independent** — consumes no upstream artifact. Examples: schema migrations, dependency installs, test fixtures, independent compute pipelines, scaffolding, new table creation, library scaffolds.
- **partial** — needs an upstream interface or contract shape, but not upstream runtime output. Can stub against the interface and integrate later.
- **blocked** — needs upstream runtime output, tested code, or a merged DTO/explainability surface. Cannot be completed without upstream shipping.

**Unblocked fraction** = (`independent` + `partial`) / total, weighted by rough effort estimate if task size is obvious; otherwise equal-weighted and flagged as `equal-weighted`.

### Output per phase

For each hard-dep phase, produce a record:

```yaml
phase: 4
title: "Complementarity (Deck Co-occurrence)"
hard_deps: [3]
unblocked_pct: 75
confidence: medium  # high | medium | low | very-low
weighting: equal-weighted  # or effort-weighted
unblocked_scope:
  - "Schema + migration for card_pair_cooccurrence table"
  - "Deck ingest loader (reads from upstream-of-P3 fixtures)"
  - "PMI + Jaccard compute kernels"
  - "Held-out eval harness wiring"
blocked_scope:
  - "Explainability DTO alignment with P3's scoring output"
  - "Final eval comparing hybrid P3+P4 signal vs P3-only baseline"
blocks_because:
  - "Explainability DTO: P3 establishes the pattern P4's surface must conform to"
  - "Hybrid eval: needs P3's scored outputs as a comparison baseline"
```

These records feed the expanded `PARALLELIZATION.md` template (see Step 4) and the final summary table (Step 6).

### Split-candidate flagging

If `unblocked_pct >= 60` AND `confidence >= medium`, flag the phase as a **split candidate**. The skill does NOT auto-split — it only suggests. Sub-phase splitting remains v2 (`--split`).

### Cost note

Each hard-dep phase adds one analysis pass. For an 8-phase linear chain that is 7 extra passes. The `--analyze-partial` flag is opt-in precisely for this reason. The linear-chain hint in Step 2 points users here when the investment is likely to pay off.

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

## Partial-Completion Analysis

_Only emitted when `--analyze-partial` was passed._

| Phase | Hard deps | Unblocked % | Confidence | Unblocked scope (can start early) | Blocked scope (must wait on upstream) |
|---|---|---|---|---|---|
| 4 — Complementarity | 3 | 75% | medium | Schema + migration; deck ingest loader; PMI/Jaccard compute; held-out eval harness | Explainability DTO alignment w/ P3 output; final hybrid eval vs P3 baseline |
| 5 — Directional Semantics | 4 | 40% | medium | Telemetry table + emitter; query-log schema | Directional scoring (needs P4 co-occurrence); feedback integration |
| 7 — REST API | 6 | 70% | high | OpenAPI spec; handler scaffolds; auth middleware; contract tests against stubs | Wiring to real gateway (P6); E2E tests |

**Split candidates** (unblocked ≥60% AND confidence ≥medium): Phase 4, Phase 7. Sub-phase split not auto-applied — see v2 `--split` flag.

**Why "blocked" entries block:** brief rationale per phase. Example — Phase 4 blocks on explainability DTO because P3 establishes the pattern P4's DTO must conform to.

## Unplanned Phases (no PLAN.md)

- Phase 9 — skipped from file-overlap analysis, grouped by explicit deps only. Partial-analysis confidence for unplanned phases is `low` or `very-low` (SPEC-only / ROADMAP-only signal).
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

If `--analyze-partial` was passed, print a second table:

| Phase | Hard deps | Unblocked % | Confidence | Split candidate? |
|---|---|---|---|---|
| 4 | 3 | 75% | medium | yes |
| 5 | 4 | 40% | medium | no |
| 7 | 6 | 70% | high | yes |

Full unblocked/blocked scope per phase lives in `PARALLELIZATION.md` — the terminal summary is just the headline numbers.

If the input was a strict linear chain and `--analyze-partial` was NOT passed, print the same hint written to the file:

> Linear chain detected. Re-run with `--analyze-partial` to estimate per-phase unblocked fraction.

Print handoff instructions:
> Push to remote: `git push origin main`
> Engineers clone repo and run: `/gsd-resume-work --ws <name>`

## Constraints

- **Never** run `workstream.create` without `--apply`.
- **Never** delete phases from the canonical milestone dir.
- **Never** push commits. Local commit only; user decides when to push.
- If `gsd-sdk workstream.create` fails, stop the loop — do not continue creating siblings. Report partial state so user can clean up manually.
- Dry-run output (`PARALLELIZATION.md`) must be **idempotent** — running dry-run twice produces the same file. Partial-analysis classifications may vary slightly run-to-run (LLM non-determinism); record confidence + classification bucket but do not promise stable percentages across reruns.
- `--analyze-partial` is **read-only advisory**. It never mutates ROADMAP.md, phase files, or dependency declarations. It only enriches `PARALLELIZATION.md` with advisory columns.

## Non-goals (v1)

- Sub-phase splitting (see `--split` in v2).
- Engineer assignment (pure grouping; humans pick who works on what).
- Cross-repo / multi-workspace parallelization (this operates on one repo's `.planning/`).
- Auto-push to remote.
- Conflict resolution at merge time — skill only prevents parallel conflicts upfront.
