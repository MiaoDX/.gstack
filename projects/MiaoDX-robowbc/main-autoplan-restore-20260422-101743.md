# /autoplan Restore Point
Captured: 2026-04-22T10:17:43+08:00 | Branch: main | Commit: 3364b0d

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
# Plan: Showcase Recovery for Remaining Policy Gaps

Branch: main
Reviewed against HEAD `a7bb236`
Artifacts: `artifacts/policy-showcase-latest/`

## Current status

Based on the latest manual check through
`python scripts/serve_showcase.py --dir artifacts/policy-showcase-latest` and
the generated JSON artifacts:

- `decoupled_wbc` is now the strongest velocity-tracking baseline in the
  showcase. Current metrics are already in the "credible" range for this demo:
  `vx_rmse ~= 0.165`, `yaw_rate_rmse ~= 0.304`, `forward_distance ~= 3.89 m`,
  and `heading_change ~= -83.8 deg`.
- `wbc_agile` improved from the earlier fully broken posture, but the full
  400-tick run still diverges badly in the second half:
  `vx_rmse ~= 0.730`, `yaw_rate_rmse ~= 11.17`, `forward_distance ~= 0.03 m`,
  and `heading_change ~= 229.8 deg`.
- `gear_sonic` velocity-tracking is still poor and also shows timing pressure:
  `vx_rmse ~= 0.899`, `yaw_rate_rmse ~= 8.24`, `dropped_frames = 23`,
  `achieved_frequency_hz ~= 41.15`.
- `gear_sonic_tracking` does run successfully as a policy path:
  `ticks = 400`, `dropped_frames = 0`, `average_inference_ms ~= 4.02`, but the
  embedded viewer is currently broken.
- `bfm_zero` currently looks acceptable as a demo, but its command contract is
  `motion_tokens`, not velocity tracking, so it should not be judged by the
  same locomotion metric contract.

There is also a concrete showcase-generation bug already identified in the
generated `artifacts/policy-showcase-latest/index.html`:

- two cards reuse the same DOM id `policy-gear_sonic`
- both embedded viewers reuse `data-rerun-policy="gear_sonic"`
- only the `data-rrd-file` differs (`gear_sonic.rrd` vs
  `gear_sonic_tracking.rrd`)

That means at least one remaining issue is not policy behavior at all. It is a
showcase identity collision.

## Problem

The remaining work is now split across three distinct classes of issues:

1. **Showcase integrity issues**
   - the embedded viewer for `gear_sonic_tracking` is not trustworthy because
     the generated page reuses policy identifiers
2. **Runtime parity issues**
   - `wbc_agile` still has a deeper contract mismatch beyond the fixes already
     landed
   - `gear_sonic` velocity mode still does not behave like the official demo
3. **Behavior-contract ambiguity**
   - the showcase still mixes velocity tracking, reference motion tracking, and
     motion-token demos without making the difference explicit enough

If we do not separate those three, we will keep wasting time debugging the
wrong layer.

## Goal

Produce a showcase that is both trustworthy and interpretable:

1. Every card renders the correct `.rrd` artifact and uses a unique identity in
   the page.
2. Every card clearly states what behavior contract it is demonstrating:
   velocity tracking, reference motion tracking, or motion-token playback.
3. `wbc_agile` and `gear_sonic` recover enough official-runtime parity to
   survive the full 400-tick velocity schedule without obvious spinout or
   target/actual decoupling.
4. `decoupled_wbc` remains the known-good baseline during all follow-up work.
5. `bfm_zero` is either explicitly positioned as a non-velocity demo or given a
   better official-style behavior script later, but not silently compared
   against the velocity controllers.

## Scope

### In scope

1. Fix the showcase/embed identity bug for duplicate policy families.
2. Make behavior contract and scenario labels explicit in the showcase.
3. Perform an official-first runtime audit for `wbc_agile`.
4. Perform an official-first runtime audit for `gear_sonic` velocity mode.
5. Clarify `bfm_zero` expectations and reserve upper-body / mocap demos for
   policies that actually support them.

### Out of scope for this plan

1. Real-robot transport or hardware benchmarking.
2. Broad MuJoCo transport refactors unless a new root cause is proven.
3. Blind gain tuning without an official-contract diff.
4. Treating `bfm_zero` motion-token playback as a velocity-tracking regression.
5. Adding new policy families before the current showcase is trustworthy.

## What already exists

- `artifacts/policy-showcase-latest/*.json` already captures the current
  before-state for all five relevant demos.
- `artifacts/posture-ablation/` already contains several focused before/after
  experiments for `decoupled_wbc`, `wbc_agile`, and `bfm_zero`.
- `configs/robots/unitree_g1_decoupled_wbc.toml` and
  `configs/robots/unitree_g1_35dof_wbc_agile.toml` now give `decoupled_wbc`
  and `wbc_agile` dedicated robot defaults instead of forcing them through the
  shared GEAR-Sonic posture.
- `crates/robowbc-ort/src/wbc_agile.rs` now uses real base angular velocity,
  prefers `base_pose` for root orientation, and feeds relative joint positions
  instead of absolute angles.

That means the next steps should not repeat those already-proven fixes.

## Priority order

1. **P0: Fix the showcase first**
   - if the page cannot display both GEAR-Sonic cards independently, we lose a
     key debugging tool
2. **P1: Recover `wbc_agile` official parity**
   - it is the most obviously still-broken velocity policy after the current
     fixes
3. **P2: Recover `gear_sonic` velocity parity**
   - it likely needs both runtime-contract and timing-path work, but only after
     the showcase and AGILE evidence chain are trustworthy
4. **P3: Clean up `bfm_zero` semantics and future upper-body demos**
   - important for user understanding, but it should not block the locomotion
     parity work

## Phase 1: Fix showcase/embed correctness

Files:

- `scripts/generate_policy_showcase.py`
- `scripts/serve_showcase.py` only if it also assumes policy-name uniqueness
- a new regression test under `tests/` for showcase HTML identity generation

Work:

1. Stop using `policy_name` alone as the DOM identity when multiple cards share
   a policy family.
2. Generate a unique card key per showcase case, for example using the output
   stem or a dedicated `card_id`.
3. Ensure both the visible card id and the embedded Rerun `data-rerun-policy`
   key are unique for `gear_sonic` and `gear_sonic_tracking`.
4. Add a small regression test that renders the HTML and proves duplicate
   policy-family cards no longer collide.
5. Regenerate the bundle and manually verify both GEAR-Sonic cards render.

Exit criteria:

1. `gear_sonic.rrd` and `gear_sonic_tracking.rrd` both display in the embedded
   viewer.
2. The generated HTML has no duplicate card ids for policy families that appear
   more than once.
3. This becomes mechanically tested, not just manually remembered.

## Phase 2: Lock the showcase behavior contract per card

Files:

- `scripts/generate_policy_showcase.py`
- showcase config metadata or an adjacent manifest consumed by the generator
- optional supporting docs under `docs/` if the labels need explanation

Work:

1. Add explicit per-card labels for:
   - command kind
   - scenario name
   - expected behavior
2. Make the current cards explicit:
   - `decoupled_wbc`: velocity tracking
   - `wbc_agile`: velocity tracking
   - `gear_sonic`: velocity planner locomotion
   - `gear_sonic_tracking`: reference motion tracking
   - `bfm_zero`: motion-token playback / latent motion prior
3. Stop presenting "stands only" as an unexplained failure when the policy is
   actually running a non-locomotion or non-velocity contract.
4. Reserve future hand-wave / upper-body demos for policies with confirmed
   mocap or official upper-body references.

Exit criteria:

1. A reader can tell from the card itself what the intended behavior is.
2. Velocity RMSE is only shown where velocity tracking is actually the contract.
3. `bfm_zero` and tracking cards are no longer visually framed as failed
   versions of the same locomotion test.

## Phase 3: Official-first WBC-AGILE recovery

Files:

- `crates/robowbc-ort/src/wbc_agile.rs`
- `configs/wbc_agile_g1.toml`
- `configs/showcase/wbc_agile_real.toml`
- `configs/robots/unitree_g1_35dof_wbc_agile.toml`
- focused fixtures/tests for tensor-level parity

Work:

1. Audit the public deploy contract against the official implementation instead
   of guessing:
   - joint ordering
   - action scaling and clipping
   - command normalization
   - history layout and stride
   - previous-action semantics
   - root-state frame convention
   - reset / warm-start behavior
2. Build a deterministic replay fixture from one short recorded observation
   trace and compare the assembled input tensors against the official runtime.
3. Fix the first remaining mismatch that causes long-horizon divergence rather
   than tuning gains.
4. Re-run the full 400-tick showcase after each contract fix, not just a short
   smoke test.

Success gate:

1. No catastrophic late spinout in the full 400-tick run.
2. `forward_distance_m > 2.5`
3. `abs(heading_change_deg + 90.0) < 30.0`
4. `yaw_rate_rmse_rad_s < 1.5`

Those are still looser than `decoupled_wbc`, but they are strict enough to
prove the runtime is no longer fundamentally wrong.

## Phase 4: Official-first GEAR-Sonic velocity recovery

Files:

- the GEAR-Sonic wrapper in `crates/robowbc-ort`
- `configs/sonic_g1.toml`
- showcase generation and report code only where metric labeling needs updates

Work:

1. Treat the official velocity demo as the oracle for the schedule semantics:
   - planner refresh cadence
   - command frame and yaw sign convention
   - warmup / stand-to-walk gating
   - latent cache reset semantics
   - any planner-vs-decoder timing split that affects dropped frames
2. Compare RoboWBC tensor inputs and output cadence against the official path
   for one identical velocity schedule.
3. Separate "wrong behavior" from "too slow" by measuring:
   - dropped frames
   - achieved control frequency
   - planner latency
   - closed-loop velocity error
4. Only optimize runtime after the command and planner contract is proven.

Success gate:

1. `dropped_frames <= 1`
2. `achieved_frequency_hz >= 47.0`
3. `vx_rmse_mps < 0.4`
4. `yaw_rate_rmse_rad_s < 1.5`
5. The resulting trajectory visibly follows the staged schedule instead of
   mostly standing, drifting, or turning the wrong way

## Phase 5: BFM-Zero and future upper-body cleanup

Files:

- showcase metadata / generator
- `configs/bfm_zero_g1.toml` only if a future official contract fix is proven
- optional new showcase assets for upper-body references

Work:

1. Keep `bfm_zero` on its current acceptable path for now and do not destabilize
   it with more posture experiments unless a stronger official contract is
   available.
2. Explicitly label `bfm_zero` as a motion-token demo in the showcase.
3. For upper-body policies, only add a hand-wave or mocap demo after confirming
   the official code or released assets provide a trustworthy reference.

Exit criteria:

1. Users no longer read `bfm_zero` as a failed velocity tracker.
2. Future upper-body demos are tied to real reference assets, not improvised
   motions.

## Validation plan

### Required Rust verification for code-bearing phases

```bash
rustc --version
cargo --version
cargo build
cargo check
cargo test
cargo fmt --check
cargo doc --no-deps
cargo clippy -- -D warnings
```

Current note: `cargo clippy -- -D warnings` already has pre-existing backlog in
`crates/robowbc-ort/src/lib.rs`. Do not treat unrelated existing lint debt as a
reason to skip the rest of the parity work, but do report it honestly on every
implementation slice.

### Showcase verification after each phase

```bash
python scripts/serve_showcase.py --dir artifacts/policy-showcase-latest
```

Manual checks:

1. Both GEAR-Sonic cards render independently.
2. Card labels match the actual command contract.
3. `decoupled_wbc` remains the baseline sanity check.
4. `wbc_agile` and `gear_sonic` are judged on full-run artifacts, not short
   clips.

## Execution note

The central rule for the remaining work is simple:

1. fix the showcase identity bug first
2. use the official runtime as the oracle for `wbc_agile` and `gear_sonic`
3. do not "tune until it looks okay"

That is the shortest path to a showcase that people can trust.
