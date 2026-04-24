# gsd-parallelize

Claude Code skill that splits a GSD project's phases into non-dependent **workstreams** for parallel engineer handoff.

## What it does

Reads `.planning/` roadmap → builds a dependency graph (explicit deps + file-overlap conflicts) → groups phases into independent workstreams → emits `PARALLELIZATION.md` → optionally materializes workstreams via `gsd-sdk`.

Each workstream becomes a `.planning/workstreams/<name>/` directory with its own `STATE.md`, `ROADMAP.md`, `phases/`. Engineers pick one up via `/gsd-resume-work --ws <name>` and work in isolation.

## Why

Hand off independent chunks of a milestone to different engineers (human or AI) without merge hell. The skill guarantees:

- **Hard deps** (`Depends on:` chains) stay inside one workstream, sequenced correctly.
- **Soft conflicts** (phases touching the same files) stay inside one workstream, sequenced to avoid merge conflicts.
- Phases with disjoint file touches + no deps land in **different** workstreams → safe parallel work.

## Installation

Skill directory is symlinked from this project into Claude Code's skill dirs:

- `~/.claude/skills/gsd-parallelize` → `~/Code/Projects/gsd-parallelize`
- `~/.claude/.jean-claude/skills/gsd-parallelize` → `~/Code/Projects/gsd-parallelize`

To edit the skill, edit `SKILL.md` in this repo. Changes take effect immediately.

## Setup tip: prefer granular phases

When initializing a GSD project, **more phases is better**. Granular phase structure > coarse phase structure.

Why: parallelization quality is bounded by phase granularity. Coarse phases bundle unrelated work → forces everything into one workstream. Granular phases expose disjoint file touches → more phases land in parallel workstreams → faster wall-clock completion.

## Usage

```bash
# Dry-run (default) — writes .planning/PARALLELIZATION.md only
/gsd-parallelize

# Materialize workstreams via gsd-sdk
/gsd-parallelize --apply

# Scope
/gsd-parallelize --scope roadmap   # active milestone only
/gsd-parallelize --scope backlog   # 999.x items only
/gsd-parallelize --scope all       # default

# Deep analysis — estimate per-phase unblocked fraction for linear/heavily-dependent roadmaps
/gsd-parallelize --analyze-partial
```

## Flags

| Flag | Default | Description |
|---|---|---|
| `--apply` | off | Materialize workstreams under `.planning/workstreams/<name>/` via `gsd-sdk`. Without it, the skill runs dry-run and writes only `PARALLELIZATION.md`. Requires a clean working tree. |
| `--scope <roadmap\|backlog\|all>` | `all` | Which phases to consider. `roadmap` = active milestone phases only. `backlog` = `999.x` items only. `all` = both. |
| `--analyze-partial` | off | Deep analysis pass. For every hard-dep phase, estimate the fraction of work that can start in parallel with its upstream phase. Adds **Unblocked %**, **Unblocked scope**, and **Blocked scope** columns to `PARALLELIZATION.md`. Read-only advisory — never mutates state. Costs one extra analysis pass per hard-dep phase. See section below. |

Reserved for v2 (not yet implemented): `--split=skip|auto|recommend` for automatic sub-phase decomposition.

### `--analyze-partial` (deep analysis)

Strict `Depends on:` semantics are binary: a phase is either unblocked or fully blocked. Reality is softer — many phases can be brought to ~60-80% completion (schema, scaffolding, independent compute, eval harness wiring) in parallel with their upstream phase, pausing only for integration/eval at the end.

`--analyze-partial` adds a second pass that, for each hard-dep phase, classifies its tasks into three buckets — `independent` / `partial` / `blocked` — and reports:

- **Unblocked %** — fraction of phase work that can start in parallel with the upstream phase
- **Unblocked scope** — what that X% actually entails (concrete task list)
- **Blocked scope** — what the remaining 100-X% entails and why it blocks

Phases with ≥60% unblocked and ≥medium confidence are flagged as **split candidates** (actual sub-phase splitting is v2).

**Auto-hint:** when dry-run detects a strict linear chain (every phase hard-deps on N-1, yielding only 1 workstream), the skill suggests re-running with `--analyze-partial` — exactly the scenario where the partial-completion analysis pays off.

**Cost:** one extra analysis pass per hard-dep phase. Opt-in for a reason.

**Read-only:** never mutates ROADMAP.md, phase dirs, or deps. Advisory columns in `PARALLELIZATION.md` only.

## Design decisions (v1)

| Decision | Choice | Rationale |
|---|---|---|
| Phase migration | **Copy** from milestone to workstream | Canonical source preserved; reversible |
| Dep inference | Hard (`Depends on:`) + Soft (file overlap) | Catches both logical and merge-time conflicts |
| Sub-phase splitting | Skip in v1; recommend-only in output | Avoid overlap with `/gsd-plan-phase` |
| Engineer assignment | Out of scope | Pure grouping; humans pick owners |
| Merge order | Emitted as recommendation | Cross-workstream hard deps determine order |
| Default mode | Dry-run | `--apply` required to mutate state |

## Roadmap

- v1 — what's described above.
- v1.1 — `--analyze-partial` advisory analysis of per-phase unblocked fraction (shipped alongside v1; read-only).
- v2 — `--split=skip|auto|recommend` flag for automatic sub-phase decomposition using `--analyze-partial` output (carve split candidates into `-a` / `-b` phases automatically).
- v2 — optional engineer assignment hints via `git blame` on touched files.
- v2 — effort-weighted unblocked fraction (currently equal-weighted fallback when task sizes are not obvious).

## Dependencies

- `gsd-sdk` on PATH (Homebrew: `brew install gsd-build/tap/gsd-sdk` or equivalent).
- A GSD-initialized project (`.planning/` with an active milestone).
