# /autoplan Restore Point
Captured: 2026-04-17 14:31:58Z | Branch: main | Commit: ca3c772

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
<!-- /autoplan restore point: /home/mi/.gstack/projects/MiaoDX-robowbc/main-autoplan-restore-20260417-143158.md -->

# Plan: roboharness visual testing pipeline integration

## Problem
RoboWBC has working policy inference and MuJoCo simulation, but there is no automated visual regression pipeline that proves the robot actually behaves correctly in simulation. Users cannot easily see side-by-side comparisons of policy outputs, and we lack a compelling visual demo that drives adoption.

## Goal
Integrate roboharness (the sibling visual testing project) into RoboWBC so that running a policy through MuJoCo automatically produces an HTML visual report with screenshots, metrics, and regression detection.

## Premises
1. Visual proof drives adoption more than latency numbers alone.
2. roboharness already knows how to capture MuJoCo frames and generate HTML reports.
3. The GEAR-SONIC real model path is now unblocked and should be the first showcase.
4. Official runtime parity comparisons (#90-94) are valuable but secondary to having a working visual pipeline.

## Scope (IN)
- Add a `roboharness/` subdirectory or integration script in RoboWBC that orchestrates: RoboWBC CLI → MuJoCo sim → frame capture → HTML report.
- Target GEAR-SONIC G1 as the first end-to-end showcase.
- Produce one command that generates the full visual report locally.
- Document the integration in `docs/` with a stack diagram.
- Update `docs/roadmap-2026-q2.md` to reflect that GEAR-SONIC real models and roboharness integration are now in progress.

## Scope (OUT — deferred)
- Cross-repo CI job combining robowbc + roboharness (needs CI architecture decision).
- Official runtime parity harness (#90-94) — kept as follow-up after visual pipeline works.
- Hardware transport (unitree_sdk2) — remains future work.

## Proposed Approach
1. **Subdirectory integration**: `tests/roboharness/` or `scripts/roboharness_integration.py` that wraps the roboharness capture API.
2. **One-shot pipeline**: `cargo run --bin robowbc -- run --config configs/sonic_g1.toml` with a special `[report]` / `[vis]` config that also triggers frame capture.
3. **HTML report generation**: Reuse roboharness's report builder, feeding it frames + run metadata (tick count, latency, policy name).

## Existing code to leverage
- `robowbc-sim` MuJoCo transport (`crates/robowbc-sim/src/lib.rs`)
- `robowbc-vis` Rerun integration (`crates/robowbc-vis/src/lib.rs`)
- `scripts/generate_policy_showcase.py` — existing report generation pattern
- Issue #42 — already scoped this work

## Acceptance Criteria
- [ ] One command produces a visual report for GEAR-SONIC G1 in MuJoCo.
- [ ] Report includes at least: policy name, frame sequence, basic metrics (tick count, avg latency).
- [ ] Documentation updated with architecture diagram and run instructions.
- [ ] Roadmap doc updated to reflect current status.
