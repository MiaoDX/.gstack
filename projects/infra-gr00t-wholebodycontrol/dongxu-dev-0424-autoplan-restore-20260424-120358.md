# /autoplan Restore Point
Captured: 2026-04-24T12:03:58+08:00 | Branch: dongxu/dev-0424 | Commit: be8af8bb

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
---
phase: 67-converge-g1-grasp-logic-with-realman-semantics
plan: 02
type: execute
wave: 2
depends_on:
  - 67-01
  - 67.1-01
  - 68-01
autonomous: true
requirements:
  - P67REQ-1
  - P67REQ-2
  - P67REQ-3
  - P67REQ-4
---

# Phase 67 - Plan 02

## Goal

Improve the remaining G1 weird-wrist posture drift by moving the convergence
intervention earlier, at the locked-waist goalset / probe winner-selection
seam, so the selected bottle-yaw family stays closer to the Realman hug-like
posture without reopening the already-regressive fallback-removal or broad
hint-widening experiments.

## Work Items

1. Add winner-selection diagnostics for the locked-waist probe / goalset path
   so the suspicious cases record why a bottle-yaw family won before the late
   grasp seed tie-break runs.
2. Add one bounded planner-side posture-family convergence mechanism at the
   earlier winner-selection seam, preferring more centered inward hug-family
   candidates when they are otherwise comparable and still execution-safe.
3. Re-run the focused suspicious G1 cases first, then the representative G1
   and Realman 8-case harnesses, and only keep the change if verdicts and
   tracked timing metrics stay within the current envelopes.
4. Refresh the short current-state grasp docs and planning summaries only if
   the runtime truth changes.

## Acceptance Criteria

- The phase records early-stage selection evidence for `X35_Y0_Z15` and
  `X45_YN15_Z10`, including selected `bottle_yaw`, hug-center deviation, and
  any new posture-quality / inward-bias metrics used by the selector.
- Any new ranking or arbitration remains local to the G1 locked-waist planning
  seam; it does not change the canonical `ControlGoalPacket`, the supported
  `decoupled_wbc` backend boundary, or the default hint-scope policy.
- Existing shared `approach_endpoint_*` diagnostics and stable `g1_*` /
  `locked_waist_*` planning-metric keys remain usable for downstream tools and
  tests.
- The suspicious cases stay `pass` and either move off the `180.5 deg`
  edge-family winner or produce explicit evidence that the edge-family choice
  is still required under the current safe boundary.
- Targeted planner tests cover the new early-stage winner-selection behavior
  and preserve the existing late tie-break coverage.
- The G1 representative suite stays at or above
  `6 pass / 2 fail / 0 execution_error`.
- The Realman representative suite stays at or above
  `7 pass / 1 fail / 0 execution_error`.
- G1 `planning_time_s`, `reik_wall_time_s`, and
  `combined_effective_latency_s` do not regress materially against the
  refreshed baselines.
- Realman `planning_time_s` does not regress materially against the refreshed
  baseline or worsen the accepted latency debt.

## Verification

- Targeted pytest on:
  - `tests/grasp/planning/test_motion_gen_stage_pipeline.py`
  - `tests/grasp/planning/test_motion_gen_planner_unit.py`
  - `tests/grasp/planning/test_waist_strategy.py`
- Focused G1 visual-harness probes on `X35_Y0_Z15` and `X45_YN15_Z10` under
  `tmp/phase67_2_experiments/g1`.
- One representative 8-case G1 harness sweep under `tmp/phase67_2_harness/g1`.
- One representative 8-case Realman harness sweep under
  `tmp/phase67_2_harness/realman`.
- A written verification summary that records verdict counts, selected
  yaw-family deltas for the suspicious cases, and the key timing metrics versus
  the current baselines.

## Task Breakdown

### Task 1: Expose earlier locked-waist winner evidence

- Files:
  `gr00t_manipulation/planner/curobo_grasp_planner.py`,
  `gr00t_manipulation/planner/waist_strategy.py`,
  `tests/grasp/planning/test_motion_gen_planner_unit.py`,
  `tests/grasp/planning/test_waist_strategy.py`
- Action:
  record the pregrasp goalset / probe winner details that currently disappear
  before the final locked-waist grasp candidate tie-break, using the existing
  planning-metrics / harness-visible surfaces rather than inventing a parallel
  telemetry path.
- Done:
  a suspicious-case report can show whether the planner already committed to an
  edge-family bottle yaw before runtime grasp finalization.

### Task 2: Add a bounded earlier-stage posture-family convergence selector

- Files:
  `gr00t_manipulation/planner/curobo_grasp_planner.py`,
  `gr00t_manipulation/planner/waist_strategy.py`,
  `tests/grasp/planning/test_motion_gen_stage_pipeline.py`,
  `tests/grasp/planning/test_motion_gen_planner_unit.py`
- Action:
  implement one localized G1-only selection change at the early winner seam,
  such as a near-tie arbitration or shortlist verify pass that can prefer a
  more centered inward hug-family yaw before the late `one_shot` versus
  `chain_seeded` endpoint choice; keep Realman and the backend contract
  untouched.
- Done:
  the selected bottle-yaw family can change earlier in the pipeline, not just
  the final endpoint seed branch.

### Task 3: Re-run suspicious cases, representative gate, and current-state docs

- Files:
  `docs/grasp/g1.md`,
  `docs/grasp/comparison.md`,
  `.planning/phases/67-converge-g1-grasp-logic-with-realman-semantics/67-02-SUMMARY.md`,
  `.planning/phases/67-converge-g1-grasp-logic-with-realman-semantics/67-VERIFICATION.md`
- Action:
  validate the new selector first on the two suspicious cases, then on the
  dual-robot representative harness; update docs only if the selected-family
  behavior or operator-visible metrics surface changes.
- Done:
  the phase either closes with explicit no-regression evidence or leaves a
  clear bounded carry-forward record of what still blocks posture convergence.
