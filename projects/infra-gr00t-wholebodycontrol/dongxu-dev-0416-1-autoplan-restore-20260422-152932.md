# /autoplan Restore Point
Captured: 2026-04-22T07:29:32Z | Branch: dongxu-dev-0416-1 | Commit: c8db4037

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
# G1 Grasp Geometry Contract Audit And Fallback Reduction Plan

**Doc Type:** Design / Implementation Plan  
**Date:** 2026-04-22  
**Status:** Proposed  
**Scope:** Audit and harden the G1 grasp geometry contract so frequent runtime
fallbacks stop masking frame/offset bugs.

Important boundary:

- this is a forward-looking remediation plan,
- it does not redefine the current live implementation reference in
  `docs/architecture/g1_grasp_pipeline_detail.md`,
- and it does not begin by loosening thresholds or deleting fallback paths.

## Why This Plan Exists

The current code already distinguishes `wrist_*_pose` from `grip_center`.
That distinction is real in the implementation:

- the object center is the task target,
- the FK-derived `grip_center` is what should land at that target,
- and the wrist pose is solved so the `grip_center` reaches it.

However, the overall contract is still too implicit across planner, runtime,
and diagnostics. The result is that several different concepts are easy to
mentally collapse together:

- object center,
- object target after `grasp_z_offset`,
- FK-derived `grip_center`,
- wrist grasp pose,
- wrist pregrasp pose,
- hand calibration offsets,
- pregrasp backoff semantics.

That ambiguity matters because the current G1 runtime really does have multiple
fallback layers:

- candidate-level waist-yaw retries,
- constrained approach fallback to IK waypoints,
- IK-waypoint fallback to endpoint-only joint interpolation.

Those fallbacks are acceptable as safety nets. They are not a good steady-state
operating mode. If they trigger frequently, the first suspicion should be
geometry-contract drift or frame/offset misuse rather than "the thresholds are
too strict."

## Target Outcome

Make the G1 grasp pipeline speak one explicit geometry contract end-to-end.

The implementation should make these statements mechanically true and easy to
verify:

1. `object_center_xyz` is the perceived object center.
2. `object_target_xyz = object_center_xyz + [0, 0, grasp_z_offset]`.
3. `grip_center_wrist` is the closed-hand FK contact center in wrist frame.
4. `wrist_grasp_pose` is chosen so `R_grasp @ grip_center_wrist + wrist_pos`
   equals `object_target_xyz`.
5. `wrist_pregrasp_pose` is derived from `wrist_grasp_pose` plus an explicit
   backoff along the active local entry direction.
6. The planner, controller, markers, logs, and tests all refer to the same
   contract rather than recomputing variants of it.

## Required Geometry Contract

### 1. Separate object target, grip center, and wrist pose

The implementation must not treat these as interchangeable:

- `object_center_xyz`
- `object_target_xyz`
- `grip_center_wrist`
- `wrist_grasp_pose`

The core grasp placement rule should remain explicit:

```text
wrist_position = object_target_xyz - R_grasp @ grip_center_wrist
```

This means:

- the wrist pose is never the object center,
- the wrist-target error and grip-center error are related but not identical,
- and diagnostics should report them separately.

### 2. Keep offset classes separate

The implementation should treat these as different classes of offsets with
different responsibilities:

- hand calibration offsets
  - `GRIP_CENTER_OFFSET_X`
  - `GRIP_CENTER_OFFSET_Y`
  - any FK-derived closed-hand contact geometry
- task offsets
  - `grasp_z_offset`
  - `pregrasp_offset`
- entry-direction geometry
  - `entry_back_dir_local`

One offset class must not silently compensate for another. In particular:

- `pregrasp_offset` must not become a hidden fix for a wrong grip center,
- `grasp_z_offset` must not be used as a proxy for wrist placement,
- and "known offsets" must remain explicit named inputs rather than implicit
  behavior spread across multiple helpers.

### 3. Make pregrasp generation explicitly family-local

When the active hand family provides `entry_back_dir_local`, pregrasp generation
should always use:

```text
wrist_pregrasp_pos = wrist_grasp_pos + pregrasp_offset * (R_grasp @ entry_back_dir_local)
```

The straight wrist-axis backoff path should remain a fallback only for families
that truly do not expose a valid local entry vector.

## Implementation Plan

### 1. Freeze threshold tuning during the audit pass

Do not start by changing:

- `grasp_ik_threshold`
- `max_grasp_branch_delta_rad`
- G1 waist probe budgets
- rescue yaw offsets
- fallback ordering

The first pass should prove whether the frequent fallback is caused by a bad
geometry contract or by a solver-quality issue after the geometry is already
correct.

### 2. Introduce one shared geometry builder for G1 grasp targets

Create one internal planner-side helper or result object that owns the full
target geometry bundle, including:

- object center,
- object target after `grasp_z_offset`,
- `grip_center_wrist`,
- `wrist_grasp_pose`,
- `entry_back_dir_local`,
- `wrist_pregrasp_pose`.

Use this helper as the single producer for:

- plan-time target generation,
- locked-waist runtime REIK finalize,
- runtime markers and diagnostic reconstruction,
- geometry verification tests.

The goal is to stop planner and controller from carrying near-duplicate
geometry logic that can drift.

### 3. Audit the G1 geometry chain in the current live path

The first code pass should inspect and align the following areas:

- planner grasp pose construction in
  `gr00t_manipulation/planner/curobo_grasp_planner.py`
- G1 hand calibration and FK-derived grip-center geometry in
  `gr00t_manipulation/planner/finger_fk.py`
- runtime REIK stage mutation and marker generation in
  `gr00t_manipulation/runtime/grasp/controller.py`

Specific questions to answer in code and tests:

- Does every `wrist_grasp_pose` reconstruct the intended `grip_center` target?
- Does every `wrist_pregrasp_pose` back off along the intended local entry line?
- Are runtime markers reporting both wrist and grip-center state without mixing
  them?
- Are constrained-approach rejections caused by wrist-target miss,
  grip-center miss, branch continuity miss, or solver failure?

### 4. Add contract-level observability at the rejection point

At locked-waist finalize and runtime REIK, log and surface distinct metrics for:

- wrist-target position error,
- reconstructed grip-center error,
- branch continuity delta,
- selected bottle yaw,
- chosen fallback path.

The rejection path for constrained approach should make it obvious whether the
problem is:

- a genuinely bad solver endpoint,
- a geometry-contract mismatch,
- or a branch-continuity rejection.

### 5. Re-evaluate fallback semantics only after geometry is proven

After the geometry contract is verified:

- keep candidate-level search retries that are clearly part of search,
- keep constrained-approach to IK-waypoint fallback only if both operate on the
  same verified geometry contract,
- re-evaluate endpoint-only joint interpolation if it remains a common path with
  materially different execution semantics.

If fallbacks become rare once geometry is fixed, no threshold change is needed.
If fallbacks remain frequent after geometry is correct, open a second plan aimed
at solver budgets and acceptance gates.

## Test Plan

Add or update tests so they validate the contract directly rather than only
asserting planner success.

### 1. Geometry contract tests

For canonical G1 object positions and yaws:

- reconstruct world `grip_center` from `wrist_grasp_pose`,
- verify it equals `object_center_xyz + grasp_z_offset`,
- verify wrist pose is not being treated as the object center.

### 2. Pregrasp direction tests

For G1 hand-local entry geometry:

- verify `wrist_pregrasp_pose` is built from `entry_back_dir_local`,
- verify the wrist-axis fallback is only used when the family exposes no entry
  vector,
- verify changing `pregrasp_offset` changes backoff distance without changing
  grip-center semantics.

### 3. Offset separation tests

Add tests that independently perturb:

- `GRIP_CENTER_OFFSET_X/Y`
- `grasp_z_offset`
- `pregrasp_offset`

and verify each one changes only the part of the contract it is supposed to
control.

### 4. Runtime finalize and marker tests

Update runtime-stage and controller tests so they confirm:

- `grip_center` markers are reconstructed from wrist pose plus FK grip center,
- wrist position and grip-center position are reported as different quantities,
- constrained-approach rejection paths report separate geometry and branch
  metrics.

### 5. Regression and visual checks

For canonical G1 harness cases:

- frequent constrained-approach rejection should disappear or become rare,
- any remaining fallback should be attributable to solver behavior rather than
  frame ambiguity,
- endpoint-only joint interpolation should not remain a common close-in path.

## Assumptions And Defaults

- This plan is intentionally narrower than a full planner rewrite.
- The first implementation pass is geometry-contract hardening plus
  observability, not fallback deletion.
- Existing G1 grip-center calibration constants are treated as intentional but
  currently unproven inputs.
- The current architecture reference remains the source of truth for what the
  code does today; this document is the source of truth for the intended
  remediation sequence.

## Related Documents

- `docs/architecture/g1_grasp_pipeline_detail.md`
- `docs/design/g1_realman_unified_hug_entry_plan.md`
- `docs/design/grasp_pipeline_findings_2026_03_09.md`
