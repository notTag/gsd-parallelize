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
```

## Installation

Skill directory is symlinked from this project into Claude Code's skill dirs:

- `~/.claude/skills/gsd-parallelize` → `~/Code/Projects/gsd-parallelize`
- `~/.claude/.jean-claude/skills/gsd-parallelize` → `~/Code/Projects/gsd-parallelize`

To edit the skill, edit `SKILL.md` in this repo. Changes take effect immediately.

## Design decisions (v1)

| Decision | Choice | Rationale |
|---|---|---|
| Phase migration | **Copy** from milestone to workstream | Canonical source preserved; reversible |
| Dep inference | Hard (`Depends on:`) + Soft (file overlap) | Catches both logical and merge-time conflicts |
| Sub-phase splitting | Skip in v1; recommend-only in output | Avoid overlap with `/gsd-plan-phase` |
| Engineer assignment | Out of scope | Pure grouping; humans pick owners |
| Merge order | Emitted as recommendation | Cross-workstream hard deps determine order |
| Default mode | Dry-run | `--apply` required to mutate state |

## Setup tip: prefer granular phases

When initializing a GSD project, **more phases is better**. Granular phase structure > coarse phase structure.

Why: parallelization quality is bounded by phase granularity. Coarse phases bundle unrelated work → forces everything into one workstream. Granular phases expose disjoint file touches → more phases land in parallel workstreams → faster wall-clock completion.

## Roadmap

- v1 — what's described above.
- v2 — `--split=skip|auto|recommend` flag for automatic sub-phase decomposition of oversized phases.
- v2 — optional engineer assignment hints via `git blame` on touched files.

## Dependencies

- `gsd-sdk` on PATH (Homebrew: `brew install gsd-build/tap/gsd-sdk` or equivalent).
- A GSD-initialized project (`.planning/` with an active milestone).
