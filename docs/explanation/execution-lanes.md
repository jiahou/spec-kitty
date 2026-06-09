---
title: "Execution Lanes"
description: "Explanation of Execution Lanes in Spec Kitty 3.2, including the model, rationale, and operator implications."
---

# Execution Lanes

Spec Kitty uses a lane-based execution model.

- `finalize_tasks` computes `lanes.json` from dependencies, ownership, and predicted surfaces.
- Each lane gets exactly one git worktree and one lane branch.
- Sequential work packages in the same lane reuse that same worktree.
- Independent lanes can run in parallel in separate worktrees.

## Core Rules

1. Planning happens in the primary repository checkout.
2. `spec-kitty agent action implement WP## --agent <name>` requires a valid `lanes.json`.
3. The runtime chooses the lane worktree. Agents do not pick a base branch manually.
4. If a feature computes one lane, the feature uses one worktree.
5. Merge always follows `lane branches -> mission branch -> target branch`.

## Naming

As of mission `083-mission-id-canonical-identity-migration`, every mission carries a ULID
identity (`mission_id`), and branch and worktree names embed the first 8 characters of
that ULID (`mid8`) to guarantee collision-free naming even when two missions share the
same human slug.

- Mission branch: `kitty/mission-<human-slug>-<mid8>`
- Lane branch: `kitty/mission-<human-slug>-<mid8>-lane-a`
- Lane worktree: `.worktrees/<human-slug>-<mid8>-lane-a/`

Example, for a mission with `mission_slug=my-feature` and
`mission_id=01J6XW9KQT7M0YB3N4R5CQZ2EX` (so `mid8=01J6XW9K`):

- Mission branch: `kitty/mission-my-feature-01J6XW9K`
- Lane branch: `kitty/mission-my-feature-01J6XW9K-lane-a`
- Lane worktree: `.worktrees/my-feature-01J6XW9K-lane-a/`

Legacy (pre-083) forms such as `kitty/mission-001-my-feature-lane-a` and
`.worktrees/001-my-feature-lane-a/` remain readable by current tooling but
are no longer the form produced by `implement`. Upgrade via the
[mission identity migration runbook](../migration/mission-id-canonical-identity.md).

## Why This Replaced Per-WP Worktrees

Per-work-package worktrees allowed overlapping work packages to run in parallel and collide at merge time. Execution lanes eliminate that by forcing **write-scope overlapping** work packages into the same lane, branch, and worktree.

## Parallelism Preservation

`finalize-tasks` assigns WPs to lanes using a union-find algorithm. Two WPs share a lane when:

1. **File ownership overlap** — their `owned_files` globs intersect (`write_scope_overlap`).
2. **Surface heuristic** — they predict the same surface tag and ownership is not provably disjoint (`surface_heuristic`).

**Explicit WP dependencies do not collapse lanes by themselves.** If WP B depends on WP A but they touch disjoint files, they remain in separate lanes. The dependency becomes a **lane-level edge** instead:

- `depends_on_lanes` — upstream lanes that must complete before this lane starts.
- `parallel_group` — lanes at the same depth in the lane DAG may run in parallel.

This preserves parallel upstream workstreams that fan in at a later WP. Runtime claim gating still requires every dependency to reach `approved` or `done` before a downstream WP can be implemented.

When union-find forces a merge, it is recorded in `lanes.json` under `collapse_report`:

```json
{
  "collapse_report": {
    "events": [
      {
        "wp_a": "WP02",
        "wp_b": "WP03",
        "rule": "write_scope_overlap",
        "evidence": "overlapping owned_files: src/foo.py"
      }
    ],
    "total_merges": 1,
    "independent_wps_collapsed": 0,
    "by_rule": { "write_scope_overlap": 1 }
  }
}
```

Inspect `collapse_report` after `finalize-tasks` to understand why two WPs share a lane. For the full scheduling model (lanes, dependencies, dispatch, and user touchpoints), see [Work Package Parallelization and Scheduling](wp-parallelization-scheduling.md).

## Lane-Specific Test Database Isolation (FR-006)

Two parallel SaaS / Django lanes used to share a single test database when their per-lane test runners booted concurrently, which produced flaky failures. Each lane workspace now exposes a lane-suffixed identifier via `LaneWorkspaceResult.lane_test_env`, which sets `SPEC_KITTY_TEST_DB_NAME=test_<safe-mission>_<safe-lane>`. Test settings modules (Django and otherwise) should read that env var when constructing their per-lane test database name; the helpers `lane_test_db_name()` and `lane_test_env()` in `specify_cli.lanes.lane_env` are the canonical entry points and guarantee distinct DB names for distinct `(mission_slug, lane_id)` pairs.
