# /autoplan Restore Point
Captured: 2026-05-19T02:28:15Z | Branch: dongxu/dev-0515 | Commit: 4d118168

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
---
refactor_scope: runtime-task-execution-seam
status: DRAFT
accepted_severities:
  - P0
  - P1
  - P2
last_verified: null
---

# Refactor Scope: Runtime Task Execution Seam

## Status

DRAFT

## Target

Deepen the task-first runtime seam so `gr00t_manipulation.runtime.task` is more
than a compatibility alias over `gr00t_manipulation.runtime.grasp`.

This slice should keep the protected grasp launch surface intact while making
runtime task planning and execution payload handling easier to test through
task-level interfaces. The near-term goal is not to move the live grasp state
machine out of `runtime.grasp`; it is to concentrate task-level semantics that
are already present but still leak through `GraspPlan` / `TaskPlan.runtime_plan`
and controller-private helper paths.

## Current Evidence

- `runtime.task` owns task construction, robot capability profile construction,
  and deterministic task-planner invocation through
  `gr00t_manipulation.runtime.task.planning`.
- `runtime.task.__init__` still exports `TaskRuntimeController`,
  `TaskRuntimeConfig`, `TaskRuntimeState`, and `main` as lazy aliases to the
  protected grasp runtime.
- `GraspLoopController` builds `ManipulationTask` values, calls
  `plan_runtime_task(...)`, then unwraps `TaskPlan.runtime_plan` back into the
  current grasp runtime payload.
- `TaskPlan.runtime_plan` is still an untyped object escape hatch; current
  deterministic plans are adapted from `GraspPlan`.
- Focused checks currently pass when ROS is sourced:
  `tests/manipulation/test_task_contracts.py`,
  `tests/manipulation/test_task_runtime.py`,
  `tests/manipulation/test_g1_control_backend_contract.py`, and
  `tests/visual_harness/test_task_report_labels.py`.

## Accepted Severities

- P0: Current deterministic G1 or Realman grasp/place runtime behavior
  regresses, or representative visual harnesses gain new execution errors.
- P0: `python -m gr00t_manipulation.runtime.grasp` or
  `python -m gr00t_manipulation.runtime.task` loses the protected launch
  contract.
- P1: Runtime task planning continues to require callers or tests to know
  `GraspPlan` internals when the behavior can be exposed through `TaskPlan`,
  `TaskStage`, or a task-runtime result Module.
- P1: `GraspLoopController` remains the only place where task-level planning
  results are interpreted, making future `ManipulationTask` kinds require
  controller-private edits instead of crossing a clearer runtime task seam.
- P1: The default or documented verification surface omits the task-first
  contract/runtime/report tests that protect this seam.
- P2: Public-surface tests only assert alias identity instead of behavior that
  must survive if `runtime.task` gains a deeper implementation.
- P2: Stale test-local docs still describe MotionGen or grasp as the only
  public planning path where current task-first docs say otherwise.

## Accepted Cleanup Checklist

- [ ] Add a task-runtime result or interpreter Module under
  `gr00t_manipulation.runtime.task` that owns deterministic task-plan result
  handling for success, planner failure with no runtime payload, and place
  planning with no runtime payload.
- [ ] Move `GraspLoopController` grasp/place call sites to delegate
  `TaskPlan` interpretation to that Module while preserving the current
  `GraspPlan` execution payload and state-machine behavior.
- [ ] Keep `TaskPlan.runtime_plan` compatibility for this slice, but reduce the
  number of places that directly inspect or unwrap it outside the task-runtime
  seam.
- [ ] Add focused behavior tests through the task-runtime seam for successful
  plans, no-runtime-plan failure handling, and place no-runtime-plan handling.
- [ ] Keep the protected `runtime.grasp` and `runtime.task` import/CLI alias
  contracts green.
- [ ] Update `tests/README.md` so the task-first contract/runtime/report tests
  are visible as important regression surfaces.
- [ ] Refresh stale test-local documentation that still describes MotionGen or
  grasp as the sole public planning path.
- [ ] Run focused static/unit/docs verification.
- [ ] Run the 8-case visual harness gates for:
  G1 `decoupled_wbc`, G1 `sonic_compat`, and Realman.
- [ ] Compare visual outputs against the tracked baseline or prior slice and
  record any accepted known failures, blockers, or material regressions.
- [ ] Commit this slice as one coherent architecture cleanup.

## Parked Cross-Seam / Future Ideas

- Do not remove `TaskPlan.runtime_plan` in this slice; that is a later payload
  migration once the task-runtime seam is load-bearing.
- Do not move the live grasp state machine out of `runtime.grasp`.
- Do not change `MotionProvider` shape here; keep that as the next major
  refactor slice.
- Do not introduce real `policy_control` or upstream VLA execution in this
  deterministic runtime cleanup.
- Do not promote `sonic_native` or `hardware_live`; both remain blocked.

## Required Evidence

- L0 static:
  `uv run --extra dev ruff check` on touched Python/test files and
  `git diff --check`.
- L1 unit/contract:
  focused tests for `runtime.task` result handling plus existing task contract,
  task runtime, public surface, and report-label checks.
- L1 docs:
  `make docs-check` plus stale-term checks for the touched docs.
- L4 local visual:
  8-case representative visual harness gates for G1 `decoupled_wbc`, G1
  `sonic_compat`, and Realman. New execution errors, unexplained verdict
  regressions, or material metric regressions block completion unless recorded
  with analysis.

## Stop Condition

Stop this slice when `runtime.task` owns deterministic `TaskPlan` interpretation
for current grasp/place runtime planning, the grasp controller delegates that
interpretation without behavior changes, focused tests exercise the task seam
instead of only private grasp internals, current docs/test docs reflect the
task-first verification surface, all required static/unit/docs checks pass, the
three 8-case visual harness gates have reviewable artifacts with no
unreasonable regressions, and the slice is committed.

## Autoplan Review

Pending. Before implementation, run the `autoplan` review gate and reconcile
accepted review decisions into this file.

## Execution Log

- 2026-05-19: Created by `intuitive-flow` after the architecture status scan.
  This is the first requested refactor slice after the completed task-first
  architecture cleanup, combining runtime-task depth with directly related
  verification/doc drift only.
