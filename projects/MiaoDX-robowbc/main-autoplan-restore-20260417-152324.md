# /autoplan Restore Point
Captured: 2026-04-17 15:23:24 UTC | Branch: main | Commit: d749dad

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
# Plan: Real GEAR-SONIC End-to-End Integration

Branch: main
Commit: d749dad
Design doc: ~/.gstack/projects/MiaoDX-robowbc/mi-main-design-20260416-100806.md

## Scope

RoboWBC v0.1.0 has a solid architecture and the GEAR-SONIC planner velocity path already works with real ONNX models. This plan addresses what is still missing:
1. Fix the broken benchmark to exercise real model inference
2. Wire the real encoder+decoder tracking contract for the full 3-model pipeline
3. Run end-to-end in MuJoCo sim and fill benchmark docs with real numbers

## What already exists

- `models/gear-sonic/`: real ONNX checkpoints downloaded (48M encoder, 40M decoder, 739M planner)
- `crates/robowbc-ort/src/lib.rs`: `GearSonicPolicy` with velocity planner path and fixture motion-token path
- `crates/robowbc-ort/benches/inference.rs`: benchmark suite (currently broken for real GEAR-SONIC — uses `MotionTokens` instead of `Velocity`)
- `configs/sonic_g1.toml`: CLI config pointing at real model paths
- `docs/benchmarks/README.md`: comparison table template, no real data
- Integration test `gear_sonic_real_model_inference`: passes with real planner on CPU

## NOT in scope

- Python bindings / LeRobot integration (deferred until core API is stable)
- Unitree G1 hardware transport (deferred until sim path is solid)
- New policy wrappers (BFM-Zero, WholeBodyVLA, WBC-AGILE)
- VLA layer integration

## Implementation steps

### Step 1: Fix the GEAR-SONIC benchmark
**File:** `crates/robowbc-ort/benches/inference.rs`

The benchmark `bench_gear_sonic_predict` constructs an `Observation` with `WbcCommand::MotionTokens`. Real encoder/decoder checkpoints expose the `obs_dict` tracking contract, so `run_fixture_motion_tokens` rejects this command.

**Change:** Switch the benchmark observation to `WbcCommand::Velocity` with a non-zero twist so it exercises `run_velocity_planner` and the real `planner_sonic.onnx`.

```rust
let obs = Observation {
    joint_positions: vec![0.0; n],
    joint_velocities: vec![0.0; n],
    gravity_vector: [0.0, 0.0, -1.0],
    angular_velocity: [0.0; 3],
    command: WbcCommand::Velocity(robowbc_core::Twist {
        linear: [0.3, 0.0, 0.0],
        angular: [0.0, 0.0, 0.0],
    }),
    timestamp: Instant::now(),
};
```

**Verification:**
```bash
GEAR_SONIC_MODEL_DIR=/home/mi/ws/gogo/robowbc/models/gear-sonic cargo bench -p robowbc-ort gear_sonic
```

### Step 2: Wire the real encoder/decoder tracking contract
**File:** `crates/robowbc-ort/src/lib.rs` (inside `GearSonicPolicy`)

The real encoder (`model_encoder.onnx`) and decoder (`model_decoder.onnx`) support the `obs_dict` tracking contract (1762D input / 994D latent, per founding doc). The current code has `supports_real_tracking_contract` but returns an error when called from `run_fixture_motion_tokens`.

**Approach:** Add a new execution path `run_tracking_contract` that:
1. Builds the `obs_dict` tensor from `Observation` (proprioception + command context)
2. Runs encoder → gets latent tokens
3. Runs decoder with latent → gets joint targets
4. Returns `JointPositionTargets`

This keeps the existing `Velocity` planner path untouched and adds the missing full 3-model pipeline.

**Open question:** Exact `obs_dict` layout (1762D). Need to inspect the real encoder input shape or read NVIDIA's C++ deploy code to construct it correctly.

**Verification:**
```bash
cargo test -p robowbc-ort -- --ignored gear_sonic_real_model_inference
```
(extend the test to cover the tracking path)

### Step 3: Run end-to-end in MuJoCo sim
**Command:**
```bash
cargo run -- run --config configs/sonic_g1.toml
```

This uses `robowbc-sim` (MuJoCo transport) and should produce valid joint targets at 50 Hz.

**Verification metrics:**
- No panics
- `achieved_hz` near 50 Hz
- `sent_commands` equals `max_ticks`

### Step 4: Benchmark and document
**Commands:**
```bash
cargo bench -p robowbc-ort
cargo bench -p robowbc-comm
```

**Update:** `docs/benchmarks/README.md` comparison table with:
- Single inference P50 / P99 for real GEAR-SONIC (encoder+planner+decoder or planner-only)
- Control loop frequency achieved in sim
- Memory RSS during benchmark

## Test coverage

### Unit tests (existing, no changes needed)
- `gear_sonic_policy_builds_with_identity_models`
- `gear_sonic_policy_rejects_mismatched_joint_velocities`
- `gear_sonic_policy_rejects_non_motion_command`
- `gear_sonic_velocity_requires_real_planner_contract`

### Integration tests
- `gear_sonic_real_model_inference` (existing, `#[ignore]`)
  - **Extend** to cover both `Velocity` and `MotionTokens` (tracking) paths
  - Keep `#[ignore]` for CI; run locally before release

### Benchmarks
- `policy/gear_sonic_predict` (fix to use real models)
- `ort/identity_*` and `ort/relu_*` (existing mock baselines)

## Failure modes

1. **Benchmark still panics after fix** → verify `GEAR_SONIC_MODEL_DIR` env var and model file existence
2. **Encoder/decoder tensor shape mismatch** → inspect ONNX metadata with `ort` API or Netron
3. **50 Hz not achieved on CPU** → profile with `cargo flamegraph` or reduce optimization level; document actual achieved frequency
4. **MuJoCo sim transport fails** → check `robowbc-sim` config and MuJoCo model path

## Parallelization

Sequential implementation — all steps touch `crates/robowbc-ort/src/lib.rs` and the benchmark file. No parallel worktree opportunity.
