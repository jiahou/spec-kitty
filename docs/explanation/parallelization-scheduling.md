---
title: "Work Package Parallelization Scheduling"
description: "How Spec Kitty schedules work packages for parallel execution: the lane DAG, parallel groups, worktrees, dispatch tiers, where the user fits, known limitations, and graph-driven improvements."
---

# Work Package Parallelization Scheduling

This page explains end-to-end how Spec Kitty decides which work packages (WPs)
can run at the same time, how that schedule is produced, how agents and
orchestrators consume it, and where the user is involved. It also includes an
adversarial review of the current mechanism and a set of improvements that the
WP dependency graph itself makes possible.

If you only want the operator workflow, read
[Parallel Development](../how-to/parallel-development.md) and
[Integrate Parallel Orchestrators](../how-to/integrate-parallel-orchestrators.md).
This page is the *why* and *how it works under the hood*.

## TL;DR

- Parallelism is **planned declaratively**, not scheduled at runtime by a central
  process. `finalize-tasks` computes a **lane DAG** and writes it to
  `lanes.json`.
- A **lane** is a group of WPs that share one git worktree and one branch.
  WPs inside a lane run **sequentially**; independent lanes run **in parallel**.
- Each lane carries a `parallel_group` integer (its depth in the lane DAG).
  **All lanes with the same `parallel_group` can run concurrently.**
- Spec Kitty does **not** launch agents. It produces the plan and owns the state
  machine. *Launching* parallel agents is the job of an external driver: a human
  in multiple terminals, the Claude Code `Task` tool, a CI matrix, or an external
  orchestrator that calls `spec-kitty orchestrator-api`.

## The pipeline, start to finish

```
tasks.md  (WP sections with dependencies)
   │
   ▼  parse + validate dependencies
WP##.md frontmatter:  dependencies: [WP01, ...] , owned_files: [...]
   │
   ▼  build_dependency_graph()  →  detect_cycles()  →  validate_dependencies()
WP dependency DAG  { "WP02": ["WP01"], ... }
   │
   ▼  compute_lanes()        (union-find collapse + topo order + depth)
Lane DAG  →  write_lanes_json()
   │
   ▼  lanes.json   (lanes, depends_on_lanes, parallel_group, collapse_report)
   │
   ▼  spec-kitty implement WP## / orchestrator-api start-implementation
allocate_lane_worktree()  →  .worktrees/<slug>-<mid8>-lane-<id>/
   │
   ▼  external driver dispatches one agent per ready lane (same parallel_group)
agents implement → for_review → in_review → approved/done
   │
   ▼  spec-kitty merge   (lane branch → mission branch → target branch)
```

### 1. Dependencies are declared per WP

Each `kitty-specs/<mission>/tasks/WP##-*.md` declares, in YAML frontmatter, what
it depends on and what files it owns:

```yaml
---
work_package_id: WP02
title: Charter Shim Deletion
dependencies: [WP01]
owned_files:
  - src/specify_cli/charter/
---
```

Two independent signals drive scheduling:

- **`dependencies`** — ordering constraints (WP02 cannot start until WP01 is
  `approved` or `done`). Enforced at runtime by
  `dependency_readiness_for_wp()` in
  [`src/specify_cli/core/dependency_graph.py`](../../src/specify_cli/core/dependency_graph.py).
- **`owned_files`** — write-scope claims used to detect physical write conflicts.

### 2. The dependency graph is built and validated

`build_dependency_graph()` scans the `tasks/WP*.md` files into an adjacency list.
`detect_cycles()` (three-color DFS) and `validate_dependencies()` reject cycles,
self-dependencies, and references to missing WPs **before** any lanes are
computed. `topological_sort()` (Kahn's algorithm) provides the within-lane
execution order.

### 3. Lanes are computed (`compute_lanes`)

The scheduler lives in
[`src/specify_cli/lanes/compute.py`](../../src/specify_cli/lanes/compute.py).
It does **not** treat dependencies as a reason to serialize into one lane — it
groups WPs by *write conflict*, then layers dependency ordering on top:

1. **Separation.** Planning-artifact WPs are pulled into a single canonical
   `lane-planning` lane that resolves to the main checkout (not a worktree).
   Code WPs enter a union-find structure.
2. **Collapse (union-find).** Two code WPs are merged into the same lane when:
   - **`write_scope_overlap`** — their `owned_files` globs intersect, or
   - **`surface_heuristic`** — they share a predicted surface tag and ownership
     is not provably disjoint.
   Every merge is recorded as a `CollapseEvent` with human-readable evidence.
3. **Lane assignment.** Each union-find component becomes an `ExecutionLane`
   with a deterministic id (`lane-a`, `lane-b`, …). WPs inside a lane are
   topologically sorted by their dependencies.
4. **Lane-level edges.** If a WP in lane B depends on a WP in lane A, then lane B
   `depends_on_lanes: ["lane-a"]`. Dependency edges are preserved *between*
   lanes, not collapsed *into* lanes.
5. **Parallel groups.** `_compute_lane_depths()` assigns each lane a depth in the
   lane DAG: `depth(L) = 0` if it has no lane dependencies, else
   `1 + max(depth(deps))`. That depth **is** the `parallel_group`.

The result is persisted atomically (temp file + rename) by
[`src/specify_cli/lanes/persistence.py`](../../src/specify_cli/lanes/persistence.py).

### 4. What `lanes.json` looks like

```json
{
  "version": 1,
  "mission_slug": "test-stabilization-01KSF9HJ",
  "lanes": [
    {
      "lane_id": "lane-a",
      "wp_ids": ["WP02", "WP03"],
      "write_scope": ["src/foo.py"],
      "predicted_surfaces": ["api"],
      "depends_on_lanes": [],
      "parallel_group": 0
    },
    {
      "lane_id": "lane-b",
      "wp_ids": ["WP05"],
      "depends_on_lanes": ["lane-a"],
      "parallel_group": 1
    }
  ],
  "collapse_report": {
    "events": [
      {"wp_a": "WP02", "wp_b": "WP03", "rule": "write_scope_overlap",
       "evidence": "overlapping globs: 'src/foo.py' vs 'src/foo.py'"}
    ],
    "independent_wps_collapsed": 0
  }
}
```

Read `collapse_report` after `finalize-tasks` to understand *why* two WPs share a
lane — it is the audit trail for lost parallelism.

### 5. Worktrees are allocated per lane (not per WP)

When `spec-kitty implement WP##` (or `orchestrator-api start-implementation`)
runs, `allocate_lane_worktree()`
([`src/specify_cli/lanes/worktree_allocator.py`](../../src/specify_cli/lanes/worktree_allocator.py))
resolves the WP's lane and creates **one** worktree and branch for that lane:

- Worktree: `.worktrees/<slug>-<mid8>-lane-<id>/`
- Branch: `kitty/mission-<slug>-<mid8>-lane-<id>`

Sequential WPs in the same lane reuse that worktree. Independent lanes get
independent worktrees, which is what makes physical parallelism safe. (This
replaced the older per-WP-worktree model, which let overlapping WPs collide at
merge time — see [Execution Lanes](execution-lanes.md).)

### 6. State is serialized even when work is parallel

The lane state machine — `planned → claimed → in_progress → for_review →
in_review → approved → done` (plus `blocked`/`canceled`) — is the **sole
authority** for WP state, backed by an append-only `status.events.jsonl`.
Concurrent agents in different worktrees can run freely, but **status writes are
serialized** by a per-mission file lock
([`src/specify_cli/status/locking.py`](../../src/specify_cli/status/locking.py))
held in the git common dir so the main checkout and every worktree coordinate.
Lane-suffixed test databases (`SPEC_KITTY_TEST_DB_NAME`) prevent parallel test
runners from colliding.

## How the user is involved

Spec Kitty deliberately keeps a human (or a human-authorized driver) in the loop.
There is no fully hidden autopilot in the core CLI. The user touches the
parallelization at five points:

| Stage | What the user does | Why it matters |
|-------|--------------------|----------------|
| **Planning** | Writes / reviews WP `dependencies` and `owned_files` | These two fields *are* the schedule. Garbage in, serial out. |
| **Finalize** | Runs `spec-kitty agent mission finalize-tasks` | Triggers cycle validation + lane computation. Fails loudly on conflicts. |
| **Inspect** | Reads `lanes.json` / `collapse_report` / dashboard | Confirms expected parallelism before committing agent time/cost. |
| **Dispatch** | Chooses the driver: terminals, `Task` tool, CI, external orchestrator; sets concurrency (e.g. `--max-concurrent`) | The core CLI never spawns agents. The user authorizes and bounds fan-out. |
| **Accept / merge** | Reviews and runs `spec-kitty merge` then opens a PR | Lane branches land via `lane → mission → target`; origin/main only via PR. |

The manual path (multiple terminals, each running `spec-kitty agent action
implement WPxx`) and the automated path (an external provider calling
`orchestrator-api`) consume the **same** `lanes.json`. The plan is identical; only
the driver differs.

## Dispatch tiers (who can be a parallel worker)

Agents are classified by how they can be launched
(`src/doctrine/skills/spec-kitty-implement-review/`):

- **Tier 1 — headless CLI** (Claude Code, Codex, Gemini, Copilot, OpenCode, Qwen,
  Kilocode, Augment, Antigravity): dispatch with `claude -p "<prompt>"
  --output-format json -C <workspace>` and equivalents.
- **Tier 2 — CLI with workaround** (Cursor): `timeout 600 cursor-agent -p
  --force`.
- **Tier 3 — GUI only** (Windsurf, Roo Cline, Amazon Q): no headless dispatch;
  the driver prints the workspace + prompt paths and a human runs the agent, then
  the driver records the transition.

A Claude Code orchestrator can fan out with the `Task` tool and
`run_in_background=True`, claiming workspaces sequentially (git mutations) and
then dispatching one subagent per ready lane.

---

## Adversarial review: shortcomings of the current mechanism

The mechanism is sound for its design center (a handful of lanes, a human or
single orchestrator driving). Pushed harder, these weaknesses surface:

1. **Static schedule, computed once.** `parallel_group` is frozen at
   `finalize-tasks`. If a WP turns out larger than planned, or a dependency is
   discovered mid-flight, the schedule is not recomputed. There is no dynamic
   re-planning from observed progress.

2. **Coarse conflict model collapses parallelism unnecessarily.** Collapse is
   driven by `owned_files` *glob overlap* and a *surface heuristic*. A directory
   glob like `src/specify_cli/charter/` forces serialization even when two WPs
   touch entirely different files inside it. The `surface_heuristic` collapses on
   shared tags "unless ownership is provably disjoint" — i.e. it defaults to
   *less* parallelism whenever ownership is fuzzy. Real concurrency is often left
   on the table.

3. **`owned_files` is a planning-time guess, unverified at runtime.** Nothing
   enforces that an agent only writes inside its declared scope. A WP that edits a
   file it never claimed can corrupt a "parallel-safe" lane at merge time. The
   conflict model trusts a field humans hand-write.

4. **`parallel_group` is descriptive, not prescriptive.** The integer says "these
   *can* run together," but no component in the core actually *gates* on it. A
   naive driver can launch a `parallel_group: 1` lane before its
   `parallel_group: 0` dependency finishes; correctness rests entirely on the
   driver respecting `depends_on_lanes`. The safety property is documented, not
   enforced by the host.

5. **Global status lock serializes a hot path.** Every transition takes one
   per-mission lock. At low lane counts this is invisible; at high fan-out (or
   with a slow filesystem / network mount) it becomes a contention point and a
   single point of failure for the whole mission.

6. **Cycle handling in depth calc is best-effort.** `_compute_lane_depths()`
   breaks cycles by treating a looped lane as depth-0 rather than failing. Upstream
   validation should prevent cycles, but if a lane-level cycle slips through, the
   emitted `parallel_group` values silently misrepresent the graph.

7. **No cost/critical-path awareness.** All lanes are treated as equal weight.
   The scheduler cannot tell a 5-minute lane from a 5-hour one, so it cannot
   prioritize the critical path or warn that one giant lane dominates wall-clock
   time regardless of how many agents you throw at it.

8. **Doc/contract drift.** `multi-agent-orchestration.md` still states "one
   worktree per WP," which contradicts the lane model (one worktree per *lane*).
   Operators reading the wrong page will mis-model concurrency. (Flagged here;
   worth correcting in a follow-up.)

9. **Dispatch parity is uneven.** Tier 3 agents can't be driven headlessly, so
   "parallel" silently degrades to "human runs N GUIs." There is no host-level
   capability negotiation that tells a driver, up front, the maximum *real*
   parallelism for the agent mix it has.

---

## Improvements unlocked by the WP dependency graph

The DAG already in `lanes.json` carries more signal than the current scheduler
extracts. Each item below is derivable from graph data the system already has.

1. **Critical-path (CPM) analysis.** With a per-WP cost estimate (even a coarse
   `size: S/M/L`), compute the longest weighted path through the lane DAG. Surface
   it as `critical_path` in `lanes.json` and in the dashboard so operators know the
   theoretical floor on wall-clock time and which lane to shorten first.

2. **Optimal worker count / fan-out hint.** The DAG's *maximum antichain* (widest
   set of mutually independent lanes) is the most agents that can ever be busy at
   once. Emit `max_useful_parallelism` so drivers and CI can size their pool and
   stop over-provisioning agents that will only queue.

3. **Cost-aware tie-breaking within a `parallel_group`.** When more lanes are
   ready than there are workers, schedule by **longest-path-to-completion first**
   (HLF / critical-path scheduling) instead of lexicographic `lane_id`. This is a
   pure ordering change over existing graph data and shortens makespan.

4. **Finer conflict granularity to reduce collapse.** Replace directory-glob
   overlap with file-level (or symbol-level, via the AST tooling already in
   `post_merge/`) ownership intersection. Fewer false-positive collapses → more
   lanes at lower depth → more real parallelism. Pair with a runtime guard that
   rejects writes outside declared scope, turning `owned_files` from a guess into a
   contract.

5. **Speculative / optimistic dispatch on weak edges.** Distinguish *hard*
   dependencies (API contract) from *soft* ones (likely-touches-same-area). Start a
   soft-dependent lane early in a throwaway worktree and validate at merge; on
   conflict, fall back to sequential. The graph already knows edge provenance from
   the collapse rules.

6. **Incremental re-planning.** On each completion event, re-run `compute_lanes`
   against *remaining* WPs with observed actuals (real files touched, real
   durations). This converts the one-shot static plan into a rolling schedule that
   adapts to reality — directly addressing shortcomings #1 and #2.

7. **Host-enforced gating.** Have `orchestrator-api start-implementation` refuse
   to start a lane whose `depends_on_lanes` are not all terminal, instead of
   trusting the driver. This makes `parallel_group` prescriptive (shortcoming #4)
   without changing the graph — just enforcing what it already asserts.

8. **Graph health diagnostics.** Add `spec-kitty doctor lanes` to report DAG
   width, depth, critical path, the single most-collapsing `owned_files` glob, and
   the parallelism lost to each collapse rule — turning `collapse_report` from a
   post-hoc log into actionable planning feedback.

A concrete, staged plan to deliver the integration and a first slice of these
improvements is in
[Integrate Parallel Orchestrators](../how-to/integrate-parallel-orchestrators.md#improvement-roadmap).

## See also

- [Execution Lanes](execution-lanes.md) — the lane/worktree model
- [Git Worktrees](git-worktrees.md) — isolation mechanics
- [Multi-Agent Orchestration](multi-agent-orchestration.md) — host/provider split
- [Integrate Parallel Orchestrators](../how-to/integrate-parallel-orchestrators.md) — drivers, SDKs, the plan
- [Handle Dependencies](../how-to/handle-dependencies.md) — declaring WP edges
- [Orchestrator API Reference](../reference/orchestrator-api.md) — the host contract
</content>
</invoke>
