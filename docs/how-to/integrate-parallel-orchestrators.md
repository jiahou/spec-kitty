---
title: "Integrate Parallel Orchestrators and Agent SDKs"
description: "Drive Spec Kitty's lane-based parallelism from existing orchestrator frameworks (Kestra, n8n, Airflow, Temporal, GitHub Actions) and from native Cursor and Claude agent / subagent / cloud SDK capabilities — plus a staged plan to make this turnkey."
---

# Integrate Parallel Orchestrators and Agent SDKs

Spec Kitty computes *which* work packages can run in parallel (the lane DAG in
`lanes.json`) but never launches agents itself. This guide shows how to plug the
plan into the driver of your choice, and lays out a staged plan to make each
integration turnkey.

Read [Work Package Parallelization Scheduling](../explanation/parallelization-scheduling.md)
first for the mechanism. This page is about *driving* it.

## The integration contract (read this once)

Every driver — human, framework, or SDK — obeys the same three rules:

1. **The host owns state.** All lane transitions go through the CLI:
   `spec-kitty orchestrator-api {list-ready, start-implementation, start-review,
   transition, accept-mission, merge-mission}`. Responses are a stable JSON
   envelope with `success` and `error_code`. Never edit status files directly.
2. **`lanes.json` is the schedule.** Run lanes whose `depends_on_lanes` are all
   terminal. Lanes sharing a `parallel_group` may run concurrently. WPs *within*
   a lane run in declared order.
3. **Claim before you dispatch.** `start-implementation` mutates git and
   allocates the lane worktree; it must run (serially) *before* an agent is
   spawned into that worktree. Then dispatch agents in parallel.

A minimal driver loop:

```bash
# 1. discover
spec-kitty orchestrator-api list-ready --mission <slug> --json
# 2. claim each ready WP (serial — git mutations)
spec-kitty orchestrator-api start-implementation --mission <slug> --wp WP01 \
  --actor <agent> --policy '{"mode":"worktree"}'
# 3. dispatch the agent into the returned workspace_path (parallel)
# 4. on completion, transition through review
spec-kitty orchestrator-api start-review --mission <slug> --wp WP01 --actor <reviewer>
spec-kitty orchestrator-api transition --mission <slug> --wp WP01 --to done --policy ...
# 5. when all approved
spec-kitty orchestrator-api accept-mission --mission <slug> --actor <driver>
```

Get the full surface with `spec-kitty orchestrator-api contract-version` and the
[Orchestrator API Reference](../reference/orchestrator-api.md).

## Native Claude Code (subagents via the Task tool)

When Claude Code is itself the orchestrator, fan out with the built-in `Task`
tool. Claim serially, dispatch in the background, one subagent per ready lane:

```python
# Pseudocode for a Claude Code orchestrating session.
ready = json(run("spec-kitty orchestrator-api list-ready --mission SLUG --json"))
for wp in ready["data"]["wp_ids"]:                      # serial: git mutations
    run(f"spec-kitty orchestrator-api start-implementation "
        f"--mission SLUG --wp {wp} --actor claude --policy '{{\"mode\":\"worktree\"}}'")

for wp in ready["data"]["wp_ids"]:                      # parallel: subagents
    Task(
        subagent_type="general-purpose",
        description=f"Implement {wp}",
        run_in_background=True,
        prompt=f"""cd {workspace_path[wp]}
                   cat {prompt_file[wp]}   # the generated implementation prompt
                   # implement, commit, then:
                   spec-kitty agent tasks move-task {wp} --to for_review""",
    )
```

Key points: subagents get isolated context, `run_in_background=True` gives real
concurrency, and each subagent only ever touches its own lane worktree. The
parent session polls `mission-state` and dispatches reviews as WPs reach
`for_review`. This is the lowest-friction path today and needs no new code.

## Claude Agent SDK / cloud (headless, programmatic)

For programmatic or cloud execution (CI, a service, Claude Code on the web), use
the Claude Agent SDK to run one headless agent per lane. The SDK call replaces
the `Task` dispatch; the host contract is unchanged:

```python
import anthropic  # Claude Agent SDK / Messages API
# ... for each claimed lane, in parallel (asyncio / workers):
#   1. cd into workspace_path
#   2. feed prompt_file as the agent's task
#   3. let the agent run spec-kitty agent tasks move-task <wp> --to for_review
```

Because every agent coordinates only through `spec-kitty orchestrator-api`, the
SDK process can run anywhere with the repo checked out and the CLI on `PATH` —
including Claude Code on the web sessions. Bound concurrency to
`max_useful_parallelism` (see roadmap) to avoid spawning agents that only queue.

> When wiring SDK calls, confirm current model IDs, streaming, and tool-use
> parameters against the Claude API reference rather than memory — see the
> `claude-api` skill.

## Native Cursor

Cursor is a Tier 2 agent: its CLI works but needs a timeout wrapper. Drive it
exactly like any other agent — claim via the host, then dispatch:

```bash
spec-kitty orchestrator-api start-implementation --mission SLUG --wp WP01 \
  --actor cursor --policy '{"mode":"worktree"}'
# dispatch (note the timeout guard for the known hang):
timeout 600 cursor-agent -p --force "$(cat <prompt_file>)" -C <workspace_path>
```

For interactive Cursor use, open the lane worktree as the workspace folder and
run the generated `/spec-kitty.implement` command; Cursor's own multi-tab model
gives you per-lane parallelism with a human deciding fan-out.

## Existing orchestrator frameworks

All frameworks follow the same shape: a **discover → fan-out → join** graph where
each node shells out to `spec-kitty orchestrator-api`. `parallel_group` maps
directly onto each engine's parallelism primitive.

| Framework | Parallelism primitive | How to map lanes |
|-----------|----------------------|------------------|
| **GitHub Actions** | matrix jobs + `needs:` | one job per lane; `needs:` mirrors `depends_on_lanes` |
| **Kestra / n8n** | parallel branches | branch per lane; join gates on lane completion |
| **Airflow** | dynamic task mapping | `expand()` over `list-ready`; set deps from `depends_on_lanes` |
| **Temporal** | child workflows / activities | one activity per lane; await futures per `parallel_group` |

GitHub Actions sketch:

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      lanes: ${{ steps.lanes.outputs.json }}
    steps:
      - run: spec-kitty orchestrator-api mission-state --mission ${{ inputs.slug }} --json
        id: lanes
  implement:
    needs: plan
    strategy:
      matrix:
        lane: ${{ fromJSON(needs.plan.outputs.lanes).lanes }}
    runs-on: ubuntu-latest
    steps:
      - run: spec-kitty orchestrator-api start-implementation --mission ${{ inputs.slug }} --wp ${{ matrix.lane.wp_ids[0] }} --actor ci --policy '{"mode":"worktree"}'
      # dispatch your agent into the returned workspace, then transition.
```

For the reference provider rather than a hand-rolled graph, see
[Run the External Orchestrator](run-external-orchestrator.md) and
[Build a Custom Orchestrator](build-custom-orchestrator.md).

## Choosing a driver

- **One developer, a few lanes** → multiple terminals
  ([Parallel Development](parallel-development.md)).
- **Claude Code session already open** → `Task` subagents (no new infra).
- **Headless / cloud / scheduled** → Claude Agent SDK or an existing framework.
- **Repeatable production pipeline** → `spec-kitty-orchestrator` or a custom
  provider on the host API.

All consume the same `lanes.json`; switching drivers never changes the plan.

---

## Improvement roadmap

A staged plan to make the integrations turnkey and to extract more parallelism
from the dependency graph. Each phase is independently shippable and ordered by
value-to-effort. Rationale for each item is in
[the adversarial review and graph-improvements sections](../explanation/parallelization-scheduling.md#adversarial-review-shortcomings-of-the-current-mechanism).

### Phase 1 — make the contract safe and observable (low effort, high value)

- **Host-enforced lane gating.** `start-implementation` refuses to start a lane
  whose `depends_on_lanes` are not all terminal. Turns `parallel_group` from a
  hint into an enforced invariant; protects every driver, including hand-rolled
  ones.
- **`spec-kitty doctor lanes`.** Report DAG width, depth, critical path, the
  most-collapsing `owned_files` glob, and parallelism lost per collapse rule.
  Makes the schedule legible before agent time is spent.
- **Fix doc drift.** Correct "one worktree per WP" in
  `multi-agent-orchestration.md` to the lane model.

### Phase 2 — driver ergonomics (medium effort)

- **First-class fan-out hints in `lanes.json`:** `critical_path`,
  `max_useful_parallelism` (max antichain). Drivers size their worker pool from
  data instead of guessing; CI/SDK callers stop over-provisioning.
- **Reference adapters** for GitHub Actions, Kestra/n8n, Airflow, Temporal, and a
  Claude Agent SDK runner — thin wrappers over `orchestrator-api`, shipped as
  copyable templates in `docs/` + `examples/`.
- **Capability negotiation:** host reports the realistic max parallelism for the
  configured agent mix (Tier 1/2/3), so drivers don't promise concurrency the
  agents can't deliver.

### Phase 3 — smarter scheduling from graph data (higher effort)

- **Finer conflict granularity:** file/symbol-level ownership intersection
  (reuse the AST tooling in `post_merge/`) instead of directory-glob overlap, plus
  a runtime guard rejecting out-of-scope writes — fewer false collapses, more real
  lanes, and `owned_files` becomes a contract.
- **Cost-aware ordering:** schedule ready lanes by longest-path-to-completion
  (critical-path first) within a `parallel_group`; needs only a coarse per-WP
  size estimate.
- **Incremental re-planning:** recompute `compute_lanes` over remaining WPs on
  each completion using observed actuals, converting the static one-shot plan into
  a rolling schedule.
- **Speculative dispatch on soft edges:** start soft-dependent lanes early in
  throwaway worktrees and validate at merge, with sequential fallback on conflict.

### Sequencing rationale

Phase 1 hardens correctness and visibility with minimal code and unblocks safe
fan-out for *every* driver. Phase 2 makes the common integrations copy-paste.
Phase 3 is where the dependency graph pays off in shorter makespan, but it
depends on the safety and observability landed earlier.

## See also

- [Work Package Parallelization Scheduling](../explanation/parallelization-scheduling.md)
- [Parallel Development](parallel-development.md)
- [Run the External Orchestrator](run-external-orchestrator.md)
- [Build a Custom Orchestrator](build-custom-orchestrator.md)
- [Orchestrator API Reference](../reference/orchestrator-api.md)
</content>
