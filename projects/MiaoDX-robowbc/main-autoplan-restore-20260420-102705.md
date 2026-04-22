# /autoplan Restore Point
Captured: 2026-04-20 10:27:05 +0800 | Branch: main | Commit: 0a8fd26

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State

<!-- /autoplan restore point: /home/mi/.gstack/projects/MiaoDX-robowbc/main-autoplan-restore-20260417-152324.md -->

# Plan: GEAR-Sonic Truth Alignment and Performance Closure

Branch: main
Reviewed against current worktree (HEAD `d1de33d`)
Design doc: ~/.gstack/projects/MiaoDX-robowbc/mi-main-design-20260416-100806.md

## Problem

RoboWBC already ships the core GEAR-Sonic runtime pieces: the planner-path
benchmark is real, the runtime has a tracking-contract path, the ignored
integration test covers both planner and tracking, and benchmark docs publish
real CPU numbers. The remaining problem is coherence.

Public surfaces disagree about whether tracking is supported, the CLI cannot
currently trigger the empty-`MotionTokens` tracking path even though runtime
code supports it, and the benchmark docs label a planner-path benchmark as the
full 3-model pipeline. The repo looks less mature than the code.

## Goal

Make the GEAR-Sonic story honest and release-ready:
1. One truth across docs, configs, roadmap, and community materials
2. An explicit user-facing decision on whether tracking is supported through the CLI
3. Explicit planner-path vs tracking-path benchmark semantics
4. Honest CPU baseline plus deferred GPU/TensorRT milestone, instead of pretending
   the current CPU path already meets 50 Hz

## Scope

1. Align status-bearing docs, templates, examples, and report metadata with the actual runtime and CLI behavior
2. Make an explicit CLI/product decision on the standing-placeholder tracking path, with downstream report compatibility
3. Split benchmark and performance semantics into the real execution modes that exist today
4. Rewrite acceptance criteria and roadmap language around the published CPU baseline

## What already exists

- `crates/robowbc-ort/src/lib.rs`: `GearSonicPolicy::run_velocity_planner`
- `crates/robowbc-ort/src/lib.rs`: `GearSonicPolicy::run_tracking_contract`
- `crates/robowbc-ort/src/lib.rs`: `gear_sonic_real_model_inference` covering both
  `Velocity` and empty `MotionTokens`
- `crates/robowbc-ort/benches/inference.rs`: real-model planner-path benchmark using
  `WbcCommand::Velocity`
- `docs/benchmarks/README.md`: real CPU benchmark numbers and control-loop frequency
- `crates/robowbc-cli/src/main.rs`: CLI runtime command builder, report serializer,
  template generator, and visual logging
- `scripts/generate_policy_showcase.py` and `scripts/roboharness_report.py`:
  downstream consumers of `command_kind` / `command_data`
- `docs/python-sdk.md` and `examples/python/gear_sonic_inference.py`: Python-facing
  command examples that currently drift from the Rust runtime story
- `README.md`, `docs/getting-started.md`, `docs/configuration.md`,
  `configs/sonic_g1.toml`, and `docs/roadmap-2026-q2.md`: status-bearing surfaces
  that currently drift

## NOT in scope

- CUDA/TensorRT implementation itself, only the follow-up milestone definition
- New motion-reference semantics beyond today's standing-placeholder tracking mode
- New policy wrappers (BFM-Zero, WholeBodyVLA, WBC-AGILE)
- Unitree G1 hardware transport expansion
- Python bindings / LeRobot distribution work beyond truth-alignment of existing docs/examples
- New visual-report UI or roboharness redesign, only metadata/schema compatibility
  work needed to keep existing reports truthful

## Implementation steps

### Step 1: Truth audit and normative-surface alignment
**Files:** `README.md`, `docs/getting-started.md`, `docs/configuration.md`,
`configs/sonic_g1.toml`, `docs/roadmap-2026-q2.md`, `docs/python-sdk.md`,
`examples/python/gear_sonic_inference.py`, `crates/robowbc-cli/src/main.rs`
(template text), `scripts/generate_policy_showcase.py`, and selected outward-facing
`docs/community/*.md`

First declare the normative surfaces that must agree:
1. `README.md`, `docs/getting-started.md`, `docs/configuration.md`,
   `configs/sonic_g1.toml`, and the `robowbc init` template
2. JSON/report/showcase metadata that contributors actually see in generated artifacts
3. Python-facing docs/examples that present the same command semantics to another audience

Treat internal planning notes and draft-only collateral as non-blocking unless they
are linked from public docs or surfaced in published artifacts.

Audit every place that claims a GEAR-Sonic status and make them agree on:
1. `configs/decoupled_smoke.toml` is the zero-download hello-world path
2. `configs/sonic_g1.toml` is the live public-model path
3. Default CLI path = `planner_sonic.onnx` velocity path
4. The runtime also has a standing-placeholder tracking path behind empty `MotionTokens`
5. Current CPU benchmark numbers measure specific execution modes, not a generic
   "full 3-model pipeline"

**Verification:**
```bash
rg -n "tracking contract|planner_sonic|full 3-model|pending|not wired yet|32\\.4|50 Hz|23 DOF|motion_tokens" \
  README.md docs configs examples crates/robowbc-cli scripts \
  -g '*.md' -g '*.toml' -g '*.py' -g '*.rs'
```

### Step 2: Normalize the CLI command surface and decide tracking exposure
**Files:** `crates/robowbc-cli/src/main.rs`, `configs/sonic_g1.toml`,
`docs/configuration.md`, `docs/getting-started.md`, `docs/python-sdk.md`,
`scripts/generate_policy_showcase.py`, `scripts/roboharness_report.py`

The runtime core supports the tracking-placeholder path through empty
`MotionTokens`, but the CLI currently rejects empty `runtime.motion_tokens`.
Do not rely on empty-array magic at the user-facing config layer, and do not add
another ad-hoc runtime field without fixing the duplicated command logic first.

**Approach:**
1. Introduce one parsed runtime-command representation inside `robowbc-cli`
2. Make validation fail fast on conflicting runtime inputs and missing payloads
3. Make `build_runtime_command`, report serialization, template comments, and
   visual logging derive from that one parsed representation
4. Make an explicit product decision on the standing-placeholder tracking mode:
   - either keep CLI velocity-only and document runtime-only support
   - or expose a Gear-Sonic-specific alias such as `standing_placeholder_tracking`
5. Preserve downstream compatibility for `command_kind` / `command_data` in JSON
   reports and showcase consumers, either by keeping legacy values stable or by
   bumping `report_version` with a documented migration note

**Verification:**
```bash
cargo test -p robowbc-cli
```

### Step 3: Split benchmark semantics and measurement protocol
**Files:** `crates/robowbc-ort/benches/inference.rs`, `docs/benchmarks/README.md`

The current benchmark is stateful. On the velocity path, most ticks interpolate
and only replan ticks actually call `planner_sonic.onnx`. The tracking path is a
separate encoder + decoder mode. There is no current runtime path that executes
encoder -> planner -> decoder inside one `predict()` call.

**Change:**
1. Rename the existing benchmark so it describes the velocity path it actually measures
2. Add explicit benchmark modes for:
   - cold-start velocity tick
   - warm steady-state velocity tick
   - replan-only planner tick
   - standing-placeholder tracking tick (encoder + decoder)
3. Document the difference between:
   - models loaded into the policy
   - models actually executed on a given tick
   - benchmark latency numbers
   - end-to-end CLI + MuJoCo achieved Hz
4. Remove any "full 3-model pipeline" label unless a real benchmark executes all
   three models in one runtime path

**Verification:**
```bash
GEAR_SONIC_MODEL_DIR=/home/mi/ws/gogo/robowbc/models/gear-sonic \
  cargo bench -p robowbc-ort -- gear_sonic
```

### Step 4: Rewrite acceptance criteria and roadmap status
**Files:** `docs/benchmarks/README.md`, `docs/roadmap-2026-q2.md`, `README.md`

Treat the current `32.4 Hz` CPU result as an honest baseline, not a hidden miss.
Define the next performance milestone explicitly as a CUDA/TensorRT follow-up
instead of pretending the CPU path is already at 50 Hz. Also define whether
the future `50 Hz` goal means average achieved loop rate, dropped-frame budget,
or replan-tick latency/jitter.

**Verification:**
```bash
cargo run --release --bin robowbc -- run --config configs/sonic_g1.toml
```

## Test coverage

### Unit tests
- `crates/robowbc-cli/src/main.rs`: add tests for parsed runtime-command
  normalization, precedence, conflict errors, and report serialization
- `crates/robowbc-cli/src/main.rs`: cover the chosen standing-placeholder
  tracking decision, whether that is an explicit alias or an explicit rejection
- `crates/robowbc-cli/src/main.rs`: keep template output/comments aligned with
  actual runtime behavior

### Integration tests
- Keep `gear_sonic_real_model_inference` as the runtime ground truth
- Add one CLI-level command-construction/report smoke test that proves the chosen
  tracking decision maps to the expected runtime path and report schema
- Add one compatibility check for downstream report consumers if `report_version`
  or `command_kind` semantics change

### Benchmarks
- Cold-start velocity tick benchmark
- Warm steady-state velocity tick benchmark
- Replan-only planner tick benchmark
- Standing-placeholder tracking tick benchmark
- End-to-end CLI + MuJoCo achieved-Hz check
- Existing `ort/identity_*` and `ort/relu_*` baselines

### Docs / consistency checks
- Add a lightweight audit script or checklist that verifies the normative surfaces
  do not drift on the key GEAR-Sonic claims
- Include `README.md`, `docs/getting-started.md`, `docs/configuration.md`,
  template text, policy config comments, `docs/python-sdk.md`,
  `examples/python/gear_sonic_inference.py`, roadmap/status docs, showcase
  metadata, and selected outward-facing community pages

## Failure modes

1. **Docs updated, but config comments remain stale** → contributors keep learning
   contradictory truths
2. **CLI tracking exposure is added without command/report normalization** →
   config parsing, report JSON, visualization, and showcase metadata drift again
3. **Benchmark labels change, but warm ticks and replan ticks stay conflated** →
   published latency still overstates what is actually executed
4. **Report schema changes silently break showcase/report consumers** → generated
   artifacts keep rendering, but tell the wrong command story
5. **CPU `32.4 Hz` remains framed as “basically 50 Hz”** → the project hides its
   real next bottleneck and leaves the future target underspecified

## Parallelization

This work is dependency-heavy. Command/report taxonomy has to settle before
benchmark docs and public-status sweeps can be trusted.

| Step | Modules touched | Depends on |
|------|------------------|------------|
| A | `crates/robowbc-cli`, `configs`, `docs/configuration.md`, `docs/getting-started.md`, `scripts/*report*` | — |
| B | `crates/robowbc-ort/benches`, `docs/benchmarks/README.md` | A |
| C | `README.md`, `docs/roadmap-2026-q2.md`, `docs/python-sdk.md`, `examples/python`, `scripts/generate_policy_showcase.py`, selected `docs/community/*` | A + B |

Execution order:
- Run A first to settle command/report taxonomy and the tracking decision
- Run B second to settle benchmark names and measurement protocol
- Run C last as the consistency sweep across all public surfaces

Conflict flags:
- A, B, and C all touch the public GEAR-Sonic story, so trying to parallelize
  them across worktrees is likely to create merge conflicts or second-order drift.

---

## /autoplan Review Report

### Phase 0: Intake Summary
- **Platform:** GitHub
- **Base branch:** `main`
- **UI scope:** NO
- **DX scope:** YES
- **Design doc:** `~/.gstack/projects/MiaoDX-robowbc/mi-main-design-20260416-100806.md`
- **Restore point:** `~/.gstack/projects/MiaoDX-robowbc/main-autoplan-restore-20260417-152324.md`
- **Repo drift since plan commit `d749dad`:**
  - `04563af feat: ORT showcase improvements and honest MuJoCo sim`
  - `18f0387 refactor: use named constants instead of magic numbers in gear_sonic`
  - `ca3c772 docs: update getting-started to reflect integrated tracking contract`
  - `7ad0790 feat: add reset() to WbcPolicy trait`
  - `da15ac3 fix: honest GearSonic tracking contract and reset support`
  - `c514678 docs: update project documentation for v0.1.0.1`

### Phase 1: CEO Review (SELECTIVE EXPANSION)

#### 0A. Premise Challenge

The plan's core sentence is no longer true on current `main`.

| Claimed missing work | Verdict | Evidence |
|----------------------|---------|----------|
| Benchmark still broken for real GEAR-Sonic | **Rejected** | `crates/robowbc-ort/benches/inference.rs:194-208` already uses `WbcCommand::Velocity` for `bench_gear_sonic_predict` |
| Real encoder/decoder tracking contract still missing | **Rejected** | `crates/robowbc-ort/src/lib.rs:1324-1397` already implements `run_tracking_contract`, and `predict()` dispatches empty `MotionTokens` into it at `crates/robowbc-ort/src/lib.rs:1432-1437` |
| Real-model integration test only covers planner path | **Rejected** | `crates/robowbc-ort/src/lib.rs:1989-2024` already exercises the tracking path |
| Benchmark docs still have a template and no real data | **Rejected** | `docs/benchmarks/README.md:46-74` already contains measured P50/P99, RSS, and control-loop frequency |
| Real GEAR-Sonic proof still matters strategically | **Accepted** | This remains the right proof-of-value, but the implementation milestone has already been crossed |

**Actual user/business outcome now:** credibility, not plumbing. The repo already has a working real-model story, but the public surface is internally contradictory and the performance target is unresolved. Right now the trust gap is bigger than the code gap.

**What happens if we do nothing:** a new developer sees three different truths:
- `README.md:37` says the encoder/decoder tracking contract is still pending.
- `docs/getting-started.md:109-112` says the tracking contract is integrated.
- `configs/sonic_g1.toml:33-36` says tracking is not wired yet.

That is how a project starts looking made up.

#### 0B. Existing Code Leverage

| Sub-problem | Existing code | Reuse path |
|-------------|---------------|------------|
| Real planner-path benchmark | `crates/robowbc-ort/benches/inference.rs:148-208` | Already runs a real-model `Velocity` benchmark |
| Real tracking-path execution | `crates/robowbc-ort/src/lib.rs:1324-1397` | Already implemented behind empty `MotionTokens` |
| Real-model verification | `crates/robowbc-ort/src/lib.rs:1919-2024` | Integration test already covers both planner and tracking paths |
| MuJoCo sim execution | `crates/robowbc-sim/src/lib.rs`, `crates/robowbc-sim/src/transport.rs` | Already used by CLI and showcase runs |
| Benchmark publishing | `docs/benchmarks/README.md:46-74` | Real CPU numbers already written |
| Public onboarding | `README.md`, `docs/getting-started.md`, `configs/sonic_g1.toml` | Needs truth-alignment, not new runtime work |

**Finding:** the repo already contains the code the plan proposes to add. Re-implementing this plan as written would spend engineering effort to rediscover shipped behavior.

#### 0C. Dream State Mapping

```text
CURRENT STATE                     THIS PLAN AS WRITTEN              12-MONTH IDEAL
Real GEAR-Sonic runs on CPU,     Re-implements shipped             Honest multi-policy runtime
tracking exists, benchmark       benchmark/tracking/docs work      with aligned docs, reproducible
docs have real numbers, but      and leaves the real gap           benchmarks, clear path semantics,
docs/configs disagree and        untouched.                        and a credible acceleration story.
CPU only reaches 32.4 Hz.
```

As written, this plan moves sideways. The codebase needs truth-alignment and a performance decision, not another pass at already-landed integration.

#### 0C-bis. Implementation Alternatives (MANDATORY)

**APPROACH A: Close this plan as done, open a follow-on plan**
- Summary: Mark the original GEAR-Sonic integration milestone complete, then create a new plan for docs sync + benchmark semantics + performance closure.
- Effort: S
- Risk: Low
- Pros: Clean bookkeeping, no ambiguity about what shipped versus what remains.
- Cons: Splits momentum across two plan files.
- Reuses: All shipped code, current benchmark docs, current ignored integration test.

**APPROACH B: Rewrite this plan in place around the remaining gap**
- Summary: Retarget the plan from "make GEAR-Sonic real" to "make GEAR-Sonic honest and release-ready" by syncing docs/configs, separating planner-only versus tracking-path semantics, and redefining the 50 Hz exit criteria.
- Effort: M
- Risk: Medium
- Pros: Preserves the current plan slot, focuses on the actual blocker, and keeps the repo truthful.
- Cons: Requires an explicit product decision on CPU versus CUDA/TensorRT as the next performance milestone.
- Reuses: Existing benchmark harness, tracking implementation, docs, CLI report path.

**APPROACH C: Keep implementing the plan as written**
- Summary: Continue treating benchmark fix, tracking contract wiring, and benchmark documentation as missing.
- Effort: M
- Risk: High
- Pros: None beyond superficial momentum.
- Cons: Duplicates shipped work, obscures the real 32.4 Hz bottleneck, and increases repo drift.
- Reuses: Very little, because it ignores current state.

**RECOMMENDATION:** Choose **Approach B**. It is the cleanest way to preserve momentum while steering the plan back onto reality. The lake to boil is now truth plus performance, not missing code paths.

#### 0D. Mode-Specific Analysis (SELECTIVE EXPANSION)

**Complexity check:** the stale plan is not large, but it is pointed at the wrong target. Complexity is not the smell here. Direction is.

**Minimum set of changes that now achieves the real goal:**
1. Align `README.md`, `configs/sonic_g1.toml`, roadmap/status docs, and benchmark descriptions with shipped behavior.
2. Distinguish **planner-path** performance from **tracking-path** performance. They are not the same code path today.
3. Rewrite the exit criteria so the project stops pretending that "near 50 Hz on CPU" is already the right immediate check when the published CPU number is `32.4 Hz`.

**Expansion scan:**
- 10x check: add CUDA/TensorRT benchmarking and publish a real "CPU honest baseline vs GPU target" matrix.
- Delight opportunities:
  1. A truth table in docs for `Velocity`, `MotionTokens([])`, and `MotionTokens(non-empty)`.
  2. Separate benchmark names for planner-only and tracking-path latency.
  3. A single release-readiness checklist for GEAR-Sonic status docs.
  4. CLI copy-paste examples for all three execution paths.
  5. One policy-status page that every doc links to instead of duplicating status text.

**Cherry-pick decisions if the plan is retargeted:**
- **ACCEPT:** docs/config truth-alignment
- **ACCEPT:** benchmark semantics clarification
- **ACCEPT:** exit-criteria rewrite for CPU vs GPU/TensorRT
- **DEFER:** actual CUDA/TensorRT implementation work unless this plan is explicitly expanded

#### 0E. Temporal Interrogation
- **Hour 1 (foundations):** Decide whether this plan is being closed as done or rewritten around the remaining gap.
- **Hour 2-3 (core logic):** Decide whether `policy/gear_sonic_predict` is planner-only by design, or whether the project now needs a second benchmark for the tracking path.
- **Hour 4-5 (integration):** Decide what the public contract for tracking really is. Today empty `MotionTokens` means "standing-pose tracking placeholder," not a generic motion-reference interface.
- **Hour 6+ (polish/tests):** Synchronize every status-bearing doc/config file so contributors stop learning contradictory truths from different entry points.

#### 0F. Mode Selection
**SELECTIVE EXPANSION** remains the right mode.
- The plan is an enhancement to an already-shipping system.
- The baseline direction needs to be corrected.
- Small, high-leverage additions inside the blast radius are worth considering once the premise gate passes.

### Section 1: Architecture Review

The current architecture already supports the runtime paths the plan describes. The gap is semantic drift between those paths and the public story.

```text
Observation
   |
   +--> WbcCommand::Velocity
   |       |
   |       +--> GearSonicPolicy::run_velocity_planner
   |               |
   |               +--> planner_sonic.onnx
   |               |
   |               +--> JointPositionTargets
   |
   +--> WbcCommand::MotionTokens([])
           |
           +--> GearSonicPolicy::run_tracking_contract
                   |
                   +--> encoder obs_dict (zero-motion standing placeholder)
                   +--> decoder obs_dict (history + current observation)
                   +--> JointPositionTargets

CLI run --> MuJoCo transport --> JSON report + Rerun --> benchmark/docs/showcase
```

**Finding:** no architecture work is missing for the plan's listed steps. The architecture problem is that the benchmark/docs surface still conflates planner-only and tracking-path execution.

**AUTO-DECISION:** If the plan is retargeted, the architecture section should focus on path semantics and benchmark naming, not new runtime components. (Principle P5)

### Section 2: Error & Rescue Map

| Method / Codepath | What can go wrong | Exception / Error | Rescued? | Rescue action | User sees |
|-------------------|-------------------|-------------------|----------|---------------|-----------|
| `bench_gear_sonic_predict` | `GEAR_SONIC_MODEL_DIR` missing or model files absent | early benchmark skip | Y | prints a clear skip message | no benchmark sample |
| `GearSonicPolicy::run_velocity_planner` | planner contract missing or wrong `joint_count` | `UnsupportedCommand` / `InvalidObservation` | Y | explicit runtime error | immediate failure |
| `GearSonicPolicy::run_tracking_contract` | encoder/decoder contract missing or output dimension mismatch | `UnsupportedCommand` / `InferenceFailed` / `InvalidTargets` | Y | explicit runtime error | immediate failure |
| `README.md` / `configs/sonic_g1.toml` status text | user believes tracking is unavailable or planner benchmark is "full pipeline" | silent documentation drift | **N** | none today | contradictory guidance |

**CRITICAL GAP:** the code rescues its failure modes clearly, but the docs do not rescue user misunderstanding at all. Wrong docs are a silent failure path.

### Section 3: Security & Threat Model

No new attack surface is introduced by retargeting this plan. The remaining risk is credibility and operational truth, not auth, secrets, or untrusted-input handling.

**No issues found.**

### Section 4: Data Flow & Interaction Edge Cases

**Real data-flow distinction the current plan misses:**

```text
planner benchmark
input: Velocity command
  -> validation
  -> planner_sonic.onnx
  -> target positions
  -> criterion samples

tracking path
input: MotionTokens([])
  -> zero-motion standing placeholder for encoder
  -> encoder latent
  -> decoder with history + current observation
  -> target positions
```

**Unhandled edge cases in the repo story:**
- `README.md:37-38` says tracking is pending, while `docs/getting-started.md:109-112` says it is integrated.
- `docs/benchmarks/README.md:27` labels `policy/gear_sonic_predict` as the full 3-model pipeline, but the benchmark itself uses the `Velocity` planner path.
- The current plan assumes Step 2 builds encoder `obs_dict` directly from `Observation`, but `crates/robowbc-ort/src/lib.rs:1234-1238` intentionally feeds a zero-motion standing placeholder when no explicit motion reference is present.

**AUTO-DECISION:** the retargeted plan must explicitly document all three of these semantics. (Principle P1)

### Section 5: Code Quality Review

The runtime code is explicit and reasonably honest. The quality issue is cross-file truth drift.

Specific mismatch:
- `PLAN.md` Step 2 describes a generic `Observation`-driven tracking contract.
- `crates/robowbc-ort/src/lib.rs:1234-1238` documents a narrower behavior: empty `MotionTokens` means "track standing pose using a zero-motion placeholder."

That difference matters. It is the kind of quiet semantic drift that produces "works on my branch" software.

**AUTO-DECISION:** rewrite the plan to match shipped semantics first. If generic motion-reference tracking is still desired, it needs to be a new explicit requirement, not smuggled in under "finish the missing tracking contract." (Principle P5)

### Section 6: Test Review

The current plan proposes tests that now already exist or are partially satisfied.

```text
EXISTING CODEPATH COVERAGE
==========================
[+] Velocity planner path
    - Real-model integration test exists
    - Real-model benchmark exists

[+] Empty MotionTokens tracking path
    - Real-model integration test exists

[+] Non-empty MotionTokens fixture path
    - Covered only by the older fixture-style path

[ ] Benchmark semantics
    - No coverage that doc descriptions match the actual execution path

[ ] Public status truthfulness
    - No guard that README/config/getting-started agree on tracking support
```

**Test gaps if the plan is retargeted:**
1. Add a tracking-path latency benchmark if performance claims need to cover the real encoder/decoder path.
2. Add a small docs/status audit checklist or script for GEAR-Sonic status-bearing files.
3. Add a targeted regression note that `MotionTokens([])` and `Velocity` intentionally hit different paths.

**AUTO-DECISION:** the old "extend `gear_sonic_real_model_inference` to cover tracking" task is already complete and should be removed from the plan. (Principle P6)

### Section 7: Performance Review

This is now the real technical gap.

- `docs/benchmarks/README.md:53-60` publishes `~14.4 ms` P50 inference and `32.4 Hz` achieved control-loop frequency on CPU.
- The plan still says the end-to-end MuJoCo run "should produce valid joint targets at 50 Hz."
- Those statements cannot both be the near-term acceptance criterion for the same CPU path.

**Top remaining performance question:** is the next milestone
1. honest CPU baseline + documented `32.4 Hz`, or
2. true `50 Hz` with CUDA/TensorRT?

**AUTO-DECISION:** the plan's exit criteria must be rewritten before any further implementation work. (Principle P1)

### Section 8: Observability & Debuggability Review

Good news:
- CLI JSON reports already exist.
- Rerun recordings already exist.
- Benchmark docs already publish achieved frequency and RSS.

Gap:
- There is no single source of truth for which Gear-Sonic path is default, which is benchmarked, and which is experimental.

**AUTO-DECISION:** add a one-page truth table or policy-status matrix when retargeting the plan. (Principle P1)

### Section 9: Deployment & Rollout Review

There is no migration or rollout risk in code. The rollout risk is public trust:
- stale plan
- stale README/config comments
- benchmark semantics that can be misread

**No code deployment issues found.**

### Section 10: Long-Term Trajectory Review

If this plan is implemented as written, it will look foolish in 6 months because it spends time replaying shipped work and still leaves the actual blocker unresolved.

- **Technical debt introduced:** planning debt and documentation debt
- **Reversibility:** 4/5, because the fix is mostly to stop lying to ourselves in repo artifacts
- **1-year question:** a new engineer should not need git archaeology to learn whether GEAR-Sonic tracking is live

**AUTO-DECISION:** treat "real GEAR-Sonic integration" as completed engineering work and move planning energy onto truth-alignment plus performance closure. (Principle P3)

### Section 11: Design & UX Review
Skipped - no UI scope detected.

### CODEX SAYS (CEO - strategy challenge)

`codex exec` agreed with the primary review on the high-signal points:
- The current plan is stale relative to `main`.
- The benchmark fix, tracking-contract implementation, integration test extension, and benchmark documentation are already present.
- `docs/benchmarks/README.md:27` is misleading because `policy/gear_sonic_predict` currently exercises the `Velocity` planner path, not an encoder -> planner -> decoder end-to-end path.
- `README.md:37-38` and `configs/sonic_g1.toml:33-36` still say tracking is pending, which contradicts `docs/getting-started.md:109-112` and the live runtime code.
- The current tracking contract is narrower than the plan text implies: empty `MotionTokens` means a zero-motion standing placeholder, not a generic motion-reference interface.

### CEO DUAL VOICES - CONSENSUS TABLE

| Dimension | Primary Review | Codex | Consensus |
|-----------|----------------|-------|-----------|
| Premises valid? | Stale relative to `main` | Stale relative to `main` | **USER CHALLENGE** |
| Right problem to solve? | Retarget to truth + performance | Retarget to truth + performance | **USER CHALLENGE** |
| Scope calibration correct? | No, duplicates shipped work | No, duplicates shipped work | **USER CHALLENGE** |
| Alternatives sufficiently explored? | Needs close-as-done vs rewrite-in-place | Same | Confirmed |
| Competitive / market risk covered? | No, contradictory docs hurt trust | Same | Confirmed |
| 6-month trajectory sound? | Not if implemented as written | Not if implemented as written | **USER CHALLENGE** |

**Voice status:** `codex-only` outside voice. A Claude subagent was not dispatched in this session.

### NOT in Scope (if this plan is retargeted)
- CUDA/TensorRT implementation itself, unless the plan is explicitly expanded beyond truth-alignment and exit-criteria rewrite
- Generic motion-reference tracking beyond the current empty-`MotionTokens` standing-placeholder contract
- Hardware transport expansion beyond the existing `robowbc-comm` work
- Python / LeRobot distribution work beyond status-text alignment

### What Already Exists
- Real planner-path benchmark
- Real tracking-path runtime code
- Real-model integration test for both planner and tracking paths
- MuJoCo sim transport
- Benchmark docs with real CPU numbers
- CLI JSON and Rerun reporting

### Dream State Delta

This plan should no longer try to get RoboWBC to "real GEAR-Sonic." It should get RoboWBC to an **honest, coherent, benchmarked** GEAR-Sonic story:
- one truth in README, config comments, getting-started, roadmap, and community docs
- one explicit explanation of planner-path versus tracking-path semantics
- one explicit performance target decision: honest CPU baseline versus CUDA/TensorRT milestone

### Error & Rescue Registry

| Codepath | Failure mode | Rescued? | Rescue action | User sees |
|----------|--------------|----------|---------------|-----------|
| `bench_gear_sonic_predict` | models or env var missing | Y | benchmark prints skip reason | no sample |
| `run_velocity_planner` | wrong contract or wrong DOF count | Y | explicit runtime error | immediate failure |
| `run_tracking_contract` | contract/output mismatch | Y | explicit runtime error | immediate failure |
| README / config / benchmark descriptions | stale or inaccurate status text | **N** | none | contradictory guidance |

### Failure Modes Registry

| Codepath | Failure mode | Rescued? | Test? | User sees? | Logged? |
|----------|--------------|----------|-------|------------|---------|
| `README.md` status row | user thinks tracking is unavailable | N | N | misleading docs | N |
| `configs/sonic_g1.toml` comments | user never tries empty `MotionTokens` tracking path | N | N | misleading docs | N |
| `docs/benchmarks/README.md` benchmark label | user interprets planner-only latency as full 3-model latency | N | N | misleading benchmark claim | N |
| CPU end-to-end run | achieved frequency is `32.4 Hz`, not the old `50 Hz` target | Y | Y | honest performance number | Y |

**CRITICAL GAPS:** the first three rows are silent failures in the project narrative.

### CEO Review Completion Summary

```text
+====================================================================+
|            MEGA PLAN REVIEW - COMPLETION SUMMARY                  |
+====================================================================+
| Mode selected        | SELECTIVE EXPANSION                         |
| System Audit         | Plan stale relative to main                 |
| Step 0               | Reframe or close as done                    |
| Section 1  (Arch)    | 1 issue found                               |
| Section 2  (Errors)  | 4 paths mapped, 1 critical doc gap cluster  |
| Section 3  (Security)| 0 issues found                              |
| Section 4  (Data/UX) | 3 semantic/documentation gaps               |
| Section 5  (Quality) | 1 issue found                               |
| Section 6  (Tests)   | Coverage diagram produced, 3 gaps           |
| Section 7  (Perf)    | 1 critical exit-criteria issue              |
| Section 8  (Observ)  | 1 gap found                                 |
| Section 9  (Deploy)  | 0 code risks, 1 credibility risk            |
| Section 10 (Future)  | Reversibility: 4/5, debt items: 2           |
| Section 11 (Design)  | SKIPPED (no UI scope)                       |
+--------------------------------------------------------------------+
| NOT in scope         | written                                     |
| What already exists  | written                                     |
| Dream state delta    | written                                     |
| Error/rescue registry| written                                     |
| Failure modes        | written, 3 critical narrative gaps          |
| TODOS.md updates     | 0 proposed (repo has no TODOS.md)           |
| Scope proposals      | 3 retarget items accepted                   |
| CEO plan             | rewritten in place around truth + performance |
| Outside voice        | ran (codex-only)                            |
| Lake Score           | 8/8 decisions favored completeness          |
| Diagrams produced    | 4 (dream state, architecture, data flow, coverage) |
| Stale diagrams found | 0                                           |
| Unresolved decisions | 1 user challenge carried to final gate      |
+====================================================================+
```

### Premise Gate Resolution

**Resolved.** The premise gate was passed when the user replied `keep going` after
the plan had been rewritten around truth-alignment and performance closure.
`/autoplan` therefore continued with Approach B, rewrite in place around the
remaining gap instead of replaying already-shipped integration work.

### Phase 2: Design Review
Skipped - no UI scope detected. This plan changes runtime semantics, benchmarks,
docs, and developer experience, not frontend presentation.

### Phase 3: Eng Review (FULL REVIEW)

#### Scope Challenge

The retargeted plan is directionally correct, but it was still under-scoped in
three places:

1. Step 2 cannot stop at CLI command construction. `command_kind` and
   `command_data` are consumed outside the CLI by JSON reports, the showcase
   generator, and roboharness HTML summaries.
2. Step 3 cannot stop at planner-path versus tracking-path. The velocity path
   itself mixes replan ticks and interpolation-only ticks, and tracking is an
   encoder + decoder mode. There is no runtime encoder -> planner -> decoder
   path today.
3. Step 1 must name the normative truth surfaces explicitly, or the work either
   misses public contradictions or balloons into endless editorial cleanup.

**AUTO-DECISION:** keep the current plan scope, but tighten the command/report
contract, benchmark taxonomy, and truth-surface boundary. This is still a lake,
not an ocean. (Principles P1, P5)

#### Architecture Review

```text
config + docs + template + examples
        |
        v
  parse_runtime_command()
        |
        +--> validate precedence / conflicts
        +--> build WbcCommand for control loop
        +--> report command_kind / command_data
        +--> vis logging + showcase metadata
        +--> upgrade / migration notes
        |
        +--> Velocity path
        |      |
        |      +--> GearSonicPolicy::run_velocity_planner
        |              |
        |              +--> every 5th tick: planner_sonic.onnx replan
        |              +--> other ticks: interpolation only
        |
        +--> standing_placeholder_tracking? (explicit alias or runtime-only)
               |
               +--> WbcCommand::MotionTokens([])
               +--> GearSonicPolicy::run_tracking_contract
                       |
                       +--> encoder + decoder only

benchmarks + docs
        |
        +--> cold-start / warm steady-state / replan / tracking / control-loop Hz
```

The plan now needs one normalized command contract inside `robowbc-cli`. Without
that, every new mode repeats the current drift between validation,
`build_runtime_command`, report serialization, template comments, and visual
logging. The right shape is explicit and local: a CLI-level parsed command enum
or helper, not a new generic `WbcCommand` abstraction in `robowbc-core`.

There is also no new distribution artifact here. The existing CLI binary, docs,
reports, and benchmarks remain the distribution vehicle. That makes blast-radius
completeness cheap. Good. Do the whole thing.

#### CODEX SAYS (ENG - adversarial review)

`codex exec` agreed with the primary Eng review on the critical issues:
- Step 3 needs a deeper benchmark taxonomy because the current velocity
  benchmark mixes replan and interpolation ticks on a persistent policy instance.
- The current tracking semantics are narrower than generic tracking. Empty
  `MotionTokens` means a standing-pose placeholder, not a user-supplied motion
  reference.
- Step 2 needs a command/report compatibility plan before adding any new CLI
  surface because JSON reports, showcase metadata, and roboharness summaries
  already consume the serialized command fields.
- Step 1 needs an explicit boundary for what counts as a public truth surface,
  otherwise the plan either misses public contradictions or turns into unbounded
  cleanup.
- Verification must cover precedence errors, report compatibility, and benchmark
  protocol semantics, not only `cargo test -p robowbc-cli`.

#### ENG DUAL VOICES - CONSENSUS TABLE

| Dimension | Primary Review | Codex | Consensus |
|-----------|----------------|-------|-----------|
| Scope direction | Tighten the current scope, do not reopen shipped runtime work | Same | Confirmed |
| Command model | Normalize to one parsed CLI command representation | Same | Confirmed |
| Tracking exposure | Treat standing-placeholder tracking as a narrow product decision, not generic tracking | Same | Confirmed |
| Benchmark taxonomy | Split cold-start, warm steady-state, replan, and tracking | Same | Confirmed |
| Truth sweep boundary | Include template, showcase, Python, and outward-facing community surfaces | Same | Confirmed |
| Verification depth | Add schema, precedence, and benchmark-protocol tests | Same | Confirmed |

**Voice status:** `codex-only` outside voice. A Claude subagent was not dispatched
in this session.

#### Code Quality Review

The code-quality issue is not algorithmic complexity, it is repeated semantics.
Right now the CLI learns "what command is this run using?" in five different
places: `validate_config`, `report_command_kind`, `report_command_data`,
`build_runtime_command`, template comments, and visualization logging. That is
how repos drift while every individual function still looks reasonable.

The reviewed plan now explicitly fixes that by requiring one parsed command
representation that every downstream surface reuses. That is the pragmatic,
explicit path. New contributors can read it in 30 seconds. No clever abstraction
needed.

The second quality issue is duplicated truth outside Rust code. The Python SDK
docs, the Python GEAR-Sonic example, and the showcase metadata each tell their
own command story today. The plan now treats those as part of the same change
set, not as cleanup someone might remember later.

#### Test Review

Existing runtime baseline coverage is already better than the stale plan gave it
credit for:
- `gear_sonic_real_model_inference` covers the velocity planner path
- `gear_sonic_real_model_inference` also covers the standing-placeholder
  tracking path
- `bench_gear_sonic_predict` exists for the current warm velocity path

What is still missing is coverage for the *reviewed plan's new commitments*:

```text
CODE PATH COVERAGE
===========================
[+] CLI runtime command normalization
    |
    ├── [GAP] velocity -> parsed once, reused by run/report/vis/template
    ├── [GAP] kinematic_pose precedence over velocity / motion_tokens
    ├── [GAP] standing-placeholder tracking alias -> MotionTokens([])
    └── [GAP] conflicting runtime fields fail fast with one clear error

[+] Report / artifact compatibility
    |
    ├── [GAP] command_kind / command_data remain truthful for JSON reports
    ├── [GAP] showcase blocked-entry metadata matches the chosen CLI contract
    └── [GAP] roboharness summary renders the chosen command wording

[+] Benchmark protocol
    |
    ├── [GAP] cold-start velocity tick after reset()
    ├── [GAP] warm steady-state velocity tick
    ├── [GAP] replan-only velocity tick
    └── [GAP] standing-placeholder tracking tick

[+] Truth surfaces
    |
    ├── [GAP] README / getting-started / configuration / template / config sync
    ├── [GAP] roadmap / showcase / outward-facing community sync
    └── [GAP] Python SDK + example sync (29 DOF, command semantics)

─────────────────────────────────
COVERAGE: 0/14 new plan paths tested today
GAPS: 14 plan-specific paths need tests/checks
─────────────────────────────────
```

**Required test additions in the plan:**
1. CLI unit tests for parsed runtime-command precedence, conflict errors, and
   report serialization
2. CLI unit tests for the chosen standing-placeholder tracking decision, either
   explicit alias or explicit rejection
3. One CLI-level command-construction/report smoke test for the chosen tracking
   path semantics
4. Benchmark protocol coverage for cold-start, warm steady-state, replan, and
   tracking modes
5. One truth-audit script or checklist that covers the normative docs/examples
   and artifact metadata

**Eng test-plan artifact written to:**  
`~/.gstack/projects/MiaoDX-robowbc/mi-main-eng-review-test-plan-20260417-170915.md`

#### Performance Review

This is the real technical risk now.

The current `~14.4 ms` benchmark is honest, but easy to misread. It is a warm
`policy.predict()` number on a persistent policy instance. On the velocity path,
most ticks do not invoke `planner_sonic.onnx`; they interpolate between planned
frames. The current tracking path is encoder + decoder only. So a simple
"planner-path versus tracking-path" rename is still too loose.

The plan now requires the complete benchmark taxonomy:
1. Cold-start velocity tick, after reset
2. Warm steady-state velocity tick, as used by the control loop
3. Replan-only velocity tick, the expensive planner invocation
4. Standing-placeholder tracking tick, encoder + decoder
5. End-to-end CLI + MuJoCo achieved Hz on CPU

It also requires documentation to separate:
- models loaded
- models executed on a given tick
- latency per benchmark mode
- achieved loop rate
- future GPU/TensorRT target semantics

Without that, the repo keeps publishing numbers that are technically real and
operationally misleading. Not great.

#### Updated Worktree Strategy

Sequential implementation is the correct recommendation here. The command/report
taxonomy is upstream of the benchmark rewrite, and both are upstream of the final
truth sweep across docs and examples.

```text
Lane A: command/report normalization + tracking decision
Lane B: benchmark protocol rewrite (after A)
Lane C: public truth sweep across docs/examples/showcase/community (after A + B)
```

There is no safe parallel split until Lane A lands. The old A/B/C parallel table
looked nice, but it would create merge conflict theater, not speed.

#### Failure Modes Registry (Eng)

| Codepath | Failure mode | Test planned? | Error handling planned? | User sees | Critical gap? |
|----------|--------------|---------------|-------------------------|-----------|---------------|
| Parsed runtime command | Conflicting runtime fields silently choose the wrong mode | Y | Y | wrong behavior or ambiguous docs | **Yes, until fixed** |
| JSON report schema | `command_kind` / `command_data` drift breaks showcase/report consumers | Y | Partial | misleading generated artifacts | **Yes, until fixed** |
| Benchmark protocol | Warm ticks and replan ticks stay conflated under new names | Y | N/A | misleading latency claims | **Yes, until fixed** |
| Truth audit boundary | Template/Python/showcase/community surfaces stay stale | Y | N | contradictory onboarding | **Yes, until fixed** |
| Future 50 Hz milestone | Goal stays underspecified after CPU baseline rewrite | Y | N/A | recurring performance confusion | No |

#### NOT in Scope (Eng)
- New generic `WbcCommand` variant for motion-reference tracking, because the
  current runtime does not expose a real generic contract yet
- CUDA/TensorRT benchmark execution itself, because this plan only defines the
  honest next milestone
- Visual-report redesign, because only metadata/schema compatibility is needed
- New Python API surface, because the DX problem is current docs/examples drift

#### What Already Exists (Eng)
- `GearSonicPolicy::run_velocity_planner` and `GearSonicPolicy::run_tracking_contract`
- `GearSonicPolicy::reset()`, which makes cold-start benchmarking tractable
- Real-model integration coverage for both current runtime paths
- Existing JSON report and Rerun surfaces
- Existing `policy/gear_sonic_predict` benchmark harness
- Existing `decoupled_smoke` no-download path and CLI template generator

#### Eng Review Completion Summary

```text
+====================================================================+
|                 ENG REVIEW - COMPLETION SUMMARY                    |
+====================================================================+
| Scope challenge        | 3 hidden-complexity issues fixed in plan  |
| Architecture Review    | 3 issues found                            |
| Code Quality Review    | 2 issues found                            |
| Test Review            | diagram produced, 14 gaps                 |
| Performance Review     | 3 issues found                            |
| NOT in scope           | written                                   |
| What already exists    | written                                   |
| Test plan artifact     | written                                   |
| Failure modes          | 4 critical gaps flagged                   |
| Outside voice          | ran (codex-only)                          |
| Parallelization        | sequential by dependency                  |
| Lake Score             | 6/6 decisions favored completeness        |
| Unresolved decisions   | 2 taste decisions carried to final gate   |
+====================================================================+
```

### Phase 3.5: DX Review (DX POLISH)

#### Product Type + Target Developer Persona

Product type: open-source robotics CLI runtime with a Python SDK and public model
integration docs.

```text
TARGET DEVELOPER PERSONA
========================
Who:       OSS robotics engineer or contributor evaluating whether RoboWBC is a
           credible Rust runtime for public humanoid WBC policies.
Context:   They clone the repo from GitHub to see if they can run something real
           without a GPU, private models, or real hardware.
Tolerance: ~5 minutes to first useful output, ~15 minutes before they stop
           trusting the repo if docs disagree.
Expects:   One zero-download smoke path, one clearly labeled live-model path,
           exact prerequisites, and docs/examples that all agree.
```

#### Developer Empathy Narrative

I clone the repo because the README says "Unified inference runtime for humanoid
whole-body control policies." Good. I want proof fast, not a research paper. The
Quick start gives me `cargo build` and `configs/decoupled_smoke.toml`, so I can
get a no-download success path. That is the right instinct. Then I scroll to the
GEAR-Sonic path because that is the policy I actually care about. Now the repo
starts arguing with itself. The README says the tracking contract is still
pending. `docs/getting-started.md` says the real encoder/decoder tracking
contract is integrated if I pass empty motion tokens. `configs/sonic_g1.toml`
says tracking is not wired yet. The benchmark page says "full 3-model pipeline,"
but the current runtime numbers do not obviously line up with that story. The
CLI help only shows `run` and `init`, so there is no discoverable explanation of
command modes or precedence. If I check the Python docs for another angle, they
tell me the Unitree G1 is 23 DOF and show a motion-token example that does not
match the Rust path I just read. At this point I can probably make it work, but
I am no longer sure which part of the repo is telling the truth. That is the DX
problem, not missing code.

#### Competitive DX Benchmark

Reference benchmark fallback, no external search used:

```text
COMPETITIVE DX BENCHMARK
=========================
Tool              | TTHW      | Notable DX Choice                     | Source
Stripe API        | ~0.5 min  | Copy-paste first success             | reference benchmark
Vercel            | ~2 min    | One-command deploy magical moment    | reference benchmark
Docker            | ~5 min    | `docker run hello-world`             | reference benchmark
RoboWBC today     | ~4-6 min smoke, ~10-15 min live GEAR-Sonic | Good smoke path, but repo truth drifts | current repo
```

**Target tier:** Competitive. The reviewed plan should get the no-download path
under 5 minutes with a clearly separated live-model path instead of pretending
they are the same journey.

#### Magical Moment Specification

**Chosen delivery vehicle:** copy-paste demo command.

For this product, the magical moment is not "loaded three huge public models."
It is "I ran one command and saw meaningful joint targets plus runtime metrics."

The reviewed plan now locks in a two-stage magical moment:
1. **T0 magical moment:** `cargo run --bin robowbc -- run --config configs/decoupled_smoke.toml`
   from a fresh clone, with no model download required
2. **T1 credibility moment:** explicitly separate "live GEAR-Sonic" as the
   next step after model download, with honest CPU baseline and command semantics

#### Mode Selection

**DX POLISH** is the right mode here. This is an enhancement to an existing
developer-facing product. The scope is already right. The repo just needs every
touchpoint to stop disagreeing with every other touchpoint.

#### Developer Journey Map

| Stage | Developer does | Friction points | Status |
|-------|----------------|-----------------|--------|
| Discover | Opens `README.md`, scans Quick start and Policy status | README says tracking pending while other docs say integrated | Fix in plan |
| Install | Runs `cargo build` and reads prerequisites in `docs/getting-started.md` | Install path is mostly fine, but live-model prerequisites are mixed into hello-world expectations | Fix in plan |
| Hello World | Runs `configs/decoupled_smoke.toml` | Repo does not consistently frame this as the intended zero-download path | Fix in plan |
| Real Usage | Downloads GEAR-Sonic models and runs `configs/sonic_g1.toml` | Tracking semantics, benchmark claims, and CPU-vs-50-Hz language drift | Fix in plan |
| Debug | Tries to infer command modes from CLI help and errors | CLI help is minimal, precedence/tracking ambiguity is not explained | Fix in plan |
| Upgrade | Tries to reason about future config/report changes | No migration note for `command_kind` / `command_data` or config wording changes | Fix in plan |

#### First-Time Developer Confusion Report

```text
FIRST-TIME DEVELOPER REPORT
============================
Persona: OSS robotics engineer / contributor
Attempting: RoboWBC getting started + live GEAR-Sonic validation

CONFUSION LOG:
T+0:00  I open the README. Good repo framing, but GEAR-Sonic tracking says "still pending."
T+2:00  I find `docs/getting-started.md` and it says the tracking contract is integrated with empty motion tokens.
T+4:00  I open `configs/sonic_g1.toml` and it says tracking is not wired yet.
T+6:00  I check the benchmark page. It says "full 3-model pipeline," but I still do not know which runtime path actually runs what.
T+8:00  I look at CLI help and only see `run` and `init`. No command-mode explanation.
T+10:00 I inspect the Python docs and see a 23-DOF G1 example plus motion-token usage that drifts from the Rust story.
T+12:00 I can keep going, but now I am debugging the repo's truth, not the policy.
```

**AUTO-DECISION:** fix every confusion point above in the plan. They are all in
blast radius, and none require a separate product initiative. (Principles P1, P2)

#### Pass 1: Getting Started Experience

**Score:** 8/10

The repo already has the right raw ingredient, `configs/decoupled_smoke.toml`.
What it lacked was honest staging. The reviewed plan now fixes that by naming the
smoke path as the hello-world path and moving live GEAR-Sonic into a clearly
labeled second step. A 10 would also show exact expected output snippets and
time budgets in the docs.

#### Pass 2: API / CLI / SDK Design

**Score:** 8/10

The plan now treats command semantics as a product surface, not an implementation
detail. That is good. What keeps it below 10 is the remaining taste decision:
whether the standing-placeholder tracking path should be exposed as an explicit
Gear-Sonic alias or remain runtime-only until a real generic motion-reference
input exists.

#### Pass 3: Error Messages & Debugging

**Score:** 8/10

The current repo makes developers infer too much from sparse CLI usage text and
contradictory docs. The reviewed plan now requires fail-fast precedence errors,
explicit wording around tracking ambiguity, and one source of truth for command
modes. That is enough to fight uncertainty instead of multiplying it.

#### Pass 4: Documentation & Learning

**Score:** 9/10

This pass improved the most. The plan now treats README, getting-started,
configuration docs, template text, Python docs/examples, showcase metadata, and
selected outward-facing community pages as one truth set. That is what makes the
repo learnable by doing instead of by archaeology.

#### Pass 5: Upgrade & Migration Path

**Score:** 8/10

The plan now explicitly includes migration handling if `command_kind`,
`command_data`, or config wording changes. That was missing. It is still not a
10 because the plan does not promise codemods or a versioned migration guide
beyond the report/config compatibility note.

#### Pass 6: Developer Environment & Tooling

**Score:** 8/10

The no-download smoke path remains the right developer-environment choice. The
plan also keeps the ORT and MuJoCo prerequisites honest instead of hiding them in
the live-model path. That is solid. It stays below 10 because the CLI still has
minimal built-in discoverability until the help and validation text are updated.

#### Pass 7: Community & Ecosystem

**Score:** 7/10

This repo already has outward-facing community docs and a live showcase. The
problem is that those surfaces drift. The reviewed plan fixes the drift, but it
does not add new examples, channels, or ecosystem investments. Reasonable scope.

#### Pass 8: DX Measurement & Feedback Loops

**Score:** 8/10

The plan now distinguishes smoke-path TTHW, live-model TTHW, benchmark modes,
and achieved-Hz reporting. It also requires a truth-audit check. That gives the
repo a way to measure the onboarding story instead of hand-waving it. Good.

#### CODEX SAYS (DX - developer journey challenge)

`codex exec` agreed with the primary DX review on the main pain points:
- `decoupled_smoke` is the real no-download hello-world path, not live GEAR-Sonic
- Live GEAR-Sonic is slower and should not be framed as T0 onboarding
- CLI help is too minimal to teach command modes or precedence
- Error surfaces do not explain tracking-mode ambiguity
- Docs are hard to trust because README, getting-started, config comments,
  benchmark docs, Python docs/examples, and showcase metadata do not agree
- Any command/config/report change needs upgrade guidance, not silent drift

#### DX DUAL VOICES - CONSENSUS TABLE

| Dimension | Primary Review | Codex | Consensus |
|-----------|----------------|-------|-----------|
| TTHW framing | Smoke path first, live GEAR-Sonic second | Same | Confirmed |
| CLI discoverability | Current help is too thin for command modes | Same | Confirmed |
| Error clarity | Tracking ambiguity and precedence need explicit errors | Same | Confirmed |
| Docs boundary | Python/showcase/community surfaces belong in the truth sweep | Same | Confirmed |
| Upgrade guidance | Command/report changes need migration notes | Same | Confirmed |
| Persona fit | Target developer is a skeptical OSS robotics engineer, not a captive internal user | Same | Confirmed |

**Voice status:** `codex-only` outside voice. A Claude subagent was not dispatched
in this session.

#### NOT in Scope (DX)
- Hosted playground or browser sandbox, because the right magical moment for this
  repo is still the one-command CLI success path
- New Python SDK features, because the immediate issue is truth-alignment of the
  existing examples and command semantics
- Community expansion work, because this plan only needs to stop outward-facing
  collateral from contradicting the runtime
- Rich CLI subcommand redesign, because the repo only needs clear mode help and
  error messages in this scope

#### What Already Exists (DX)
- `README.md` already offers a quick-start smoke path
- `docs/getting-started.md` already separates smoke and live model sections, even
  if the wording is currently inconsistent
- `configs/decoupled_smoke.toml` is already the right zero-download demo asset
- JSON run reports already surface runtime metrics that can power the magical moment
- Python SDK docs and examples already exist, even if they currently drift
- Live showcase and community docs already exist, even if they currently drift

#### DX Scorecard

```text
+====================================================================+
|              DX PLAN REVIEW - SCORECARD                            |
+====================================================================+
| Dimension            | Score  | Prior  | Trend                     |
|----------------------|--------|--------|---------------------------|
| Getting Started      | 8/10   | —      | new                       |
| API/CLI/SDK          | 8/10   | —      | new                       |
| Error Messages       | 8/10   | —      | new                       |
| Documentation        | 9/10   | —      | new                       |
| Upgrade Path         | 8/10   | —      | new                       |
| Dev Environment      | 8/10   | —      | new                       |
| Community            | 7/10   | —      | new                       |
| DX Measurement       | 8/10   | —      | new                       |
+--------------------------------------------------------------------+
| TTHW                 | ~4-6 min smoke, ~10-15 min live GEAR-Sonic |
| Target TTHW          | <5 min smoke, clearly staged live path      |
| Competitive Rank     | Competitive                                  |
| Magical Moment       | designed via copy-paste demo command         |
| Product Type         | OSS robotics CLI + Python SDK                |
| Mode                 | DX POLISH                                    |
| Overall DX           | 8/10                                         |
+====================================================================+
| DX PRINCIPLE COVERAGE                                              |
| Zero Friction      | covered                                        |
| Learn by Doing     | covered                                        |
| Fight Uncertainty  | covered                                        |
| Opinionated + Escape Hatches | covered                              |
| Code in Context    | covered                                        |
| Magical Moments    | covered                                        |
+====================================================================+
```

#### DX Implementation Checklist

```text
DX IMPLEMENTATION CHECKLIST
============================
[ ] Time to hello world < 5 min via `configs/decoupled_smoke.toml`
[ ] Installation path keeps Rust preflight and zero-download smoke path explicit
[ ] First run produces meaningful output and runtime metrics
[ ] Magical moment is delivered via one copy-paste demo command
[ ] Every command-mode error includes problem + cause + fix
[ ] CLI naming is guessable without docs
[ ] Default path stays velocity; tracking mode, if exposed, is explicitly named
[ ] Docs have copy-paste examples that actually work
[ ] Examples show both smoke and live public-model paths
[ ] Upgrade path documents any `command_kind` / config wording changes
[ ] Breaking changes have deprecation warnings or report-version notes
[ ] Python docs/examples use the correct 29-DOF G1 expectations
[ ] Works in CI/CD and local CLI without special hardware
[ ] Smoke path requires no model downloads
[ ] Roadmap / benchmark language matches measured CPU reality
[ ] Truth-audit search or checklist verifies the public surfaces
[ ] Outward-facing community docs and showcase metadata are aligned
```

#### DX Review Completion Summary

```text
+====================================================================+
|                 DX REVIEW - COMPLETION SUMMARY                     |
+====================================================================+
| Persona              | OSS robotics engineer / contributor         |
| Product type         | CLI + docs + Python SDK                    |
| Mode                 | DX POLISH                                  |
| Current TTHW         | ~4-6 min smoke, ~10-15 min live path       |
| Target TTHW          | <5 min smoke, clearly staged live path     |
| Review passes        | 8 scored                                   |
| Persona card         | written                                    |
| Empathy narrative    | written                                    |
| Journey map          | written                                    |
| Confusion report     | written                                    |
| Scorecard            | written                                    |
| Checklist            | written                                    |
| Outside voice        | ran (codex-only)                           |
| Unresolved decisions | 1 taste decision carried to final gate     |
+====================================================================+
```

### Cross-Phase Themes

**Theme: Truth drift beats missing code** — flagged in CEO, Eng, and DX. The
repo already has more runtime reality than its public surfaces admit. The real
work is making every surface tell the same true story.

**Theme: Empty `MotionTokens` needs explicit ownership** — flagged in CEO, Eng,
and DX. The current runtime behavior is real but narrow. It must either stay
runtime-only or become an explicitly named Gear-Sonic CLI mode. It cannot remain
"magic if you know the right empty array."

**Theme: Performance honesty needs benchmark protocol, not nicer adjectives** —
flagged in CEO, Eng, and DX. The current CPU numbers are useful. The misleading
part is the taxonomy around them.

<!-- AUTONOMOUS DECISION LOG -->
## Decision Audit Trail

| # | Phase | Decision | Classification | Principle | Rationale | Rejected |
|---|-------|----------|----------------|-----------|-----------|----------|
| 1 | CEO | Reject "benchmark still broken" premise | Mechanical | P6 | Current benchmark already uses `Velocity` | Original stale assumption |
| 2 | CEO | Reject "tracking contract missing" premise | Mechanical | P6 | Current runtime already implements `run_tracking_contract` | Original stale assumption |
| 3 | CEO | Reject "benchmark docs still empty" premise | Mechanical | P6 | Real numbers already exist in `docs/benchmarks/README.md` | Original stale assumption |
| 4 | CEO | Recommend rewrite-in-place over reimplementation | User Challenge | P3 | Retargeting preserves momentum while matching reality | Keep implementing stale plan |
| 5 | CEO | Treat docs/config truth-alignment as part of the remaining lake | Mechanical | P1 | Silent narrative failures are still failures | Leaving drift in place |
| 6 | CEO | Separate planner-path and tracking-path semantics in the plan | Mechanical | P5 | Current repo already has two distinct execution paths | Conflated "full pipeline" language |
| 7 | CEO | Rewrite performance exit criteria around `32.4 Hz` reality | Mechanical | P1 | Published CPU number is below the old target | Pretending CPU already satisfies the target |
| 8 | CEO | Remove already-satisfied tracking-test extension from remaining work | Mechanical | P6 | Integration test already covers tracking | Re-adding completed work |
| 9 | Eng | Normalize CLI command/report semantics before exposing tracking | Mechanical | P5 | One parsed command surface is simpler and prevents drift | Another ad-hoc runtime field |
| 10 | Eng | Treat standing-placeholder tracking as a narrow product decision, not generic tracking | Taste | P5 | Current runtime semantics are real but policy-specific | Generic `tracking` surface |
| 11 | Eng | Preserve or version `command_kind` / `command_data` compatibility as part of Step 2 | Mechanical | P1 | JSON reports are consumed outside the CLI today | CLI-only change that ignores consumers |
| 12 | Eng | Expand benchmark taxonomy to cold-start, warm steady-state, replan, and tracking modes | Taste | P1 | A two-way split still hides the expensive planner path | Simple planner/tracking rename |
| 13 | Eng | Reclassify implementation order as sequential by dependency | Mechanical | P3 | Command/report taxonomy is upstream of benchmark/docs work | Parallel A+B+C lanes |
| 14 | Eng | Add template, showcase, Python, and outward-facing community surfaces to verification scope | Taste | P2 | Contributors and evaluators see those surfaces before the code | Audit only README/config/benchmarks |
| 15 | DX | Keep `decoupled_smoke` as the hello-world path and stage live GEAR-Sonic second | Mechanical | P1 | Fast trustworthy success matters more than brand-name model marketing | Frame live GEAR-Sonic as T0 onboarding |
| 16 | DX | Use a copy-paste demo command as the magical moment | Mechanical | P3 | This is a CLI-first product, not a hosted playground | Wait for sandbox or video-first onboarding |
| 17 | DX | Add explicit command-mode and precedence guidance to help/errors/docs | Mechanical | P1 | Current help text is too thin for a skeptical first-time contributor | Rely on docs archaeology |
| 18 | DX | Treat Python docs/examples as normative DX surfaces | Mechanical | P2 | Same runtime must tell one story across languages | Leave Python drift for later |
| 19 | DX | Add upgrade/migration notes for any command/report/config wording changes | Mechanical | P1 | Prevent silent breakage for report consumers and SDK users | Silent schema drift |
| 20 | DX | Target competitive-tier DX, <5 min smoke TTHW with a clearly slower live path | Mechanical | P3 | This is achievable inside the current blast radius | Accept today's blurred onboarding story |

### Phase 4: Final Approval Gate

#### Plan Summary

This plan no longer tries to "make GEAR-Sonic real." That already happened in
`main`. It now does the remaining work that actually matters: truth alignment,
explicit command semantics, honest benchmark taxonomy, and a trustworthy CPU
performance story.

#### Decisions Made: 20 total (16 auto-decided, 3 taste choices, 1 user challenge)

#### User Challenges

**Challenge 1: Rewrite instead of re-implement** (from CEO)  
You said: finish the missing benchmark fix, tracking contract, and benchmark docs.  
Both models recommend: treat those as already shipped, and retarget the plan to
truth-alignment plus performance closure.  
Why: the benchmark, tracking path, integration test, and benchmark docs are
already in `main`; the remaining gap is contradictory public status text plus
unresolved command/report and `32.4 Hz` versus `50 Hz` semantics.  
What we might be missing: you may intentionally want an audit of shipped behavior
rather than a new implementation plan, or you may know of a hidden edge case not
visible from the checked-in code.  
If we're wrong, the cost is: we close or rewrite a plan that was meant to drive
one last correctness pass on a subtle edge case.

Your call. Your original direction stands unless you explicitly change it.

#### Your Choices

**Choice 1: CLI exposure for standing-placeholder tracking** (from Eng)  
I recommend: expose it only as an explicit Gear-Sonic-specific alias, not as a
generic `tracking` surface. That keeps the runtime honest without polluting the
shared command model.  
If you instead keep the CLI velocity-only, the runtime stays narrower and safer,
but the docs must say clearly that tracking is runtime-internal today.

**Choice 2: Benchmark taxonomy depth** (from Eng)  
I recommend: ship the full cold-start / warm steady-state / replan / tracking
split. That is the complete version.  
If you instead do a simple planner/tracking rename, the docs get shorter, but
the numbers stay easier to misread.

**Choice 3: Truth sweep boundary** (from Eng/DX)  
I recommend: include README, getting-started, configuration, template text,
showcase metadata, Python docs/examples, and outward-facing community docs in
this plan.  
If you instead limit the sweep to the core docs/configs, the repo center gets
cleaner faster, but public collateral keeps drifting.

#### Auto-Decided: 16 decisions

See the Decision Audit Trail above.

#### Review Scores

- CEO: retargeted successfully from stale integration work to truth + performance closure
- CEO Voices: Codex confirmed the stale-premise diagnosis, Claude subagent skipped, Consensus 6/6 confirmed between primary review and Codex
- Design: skipped, no UI scope
- Design Voices: skipped
- Eng: command/report taxonomy, benchmark protocol, and execution order corrected, 4 critical gaps flagged
- Eng Voices: Codex confirmed the core architecture/test/performance concerns, Claude subagent skipped, Consensus 6/6 confirmed between primary review and Codex
- DX: smoke-first onboarding, command clarity, migration guidance, and truth-surface alignment added, overall target 8/10 DX
- DX Voices: Codex confirmed the journey/help/docs/migration concerns, Claude subagent skipped, Consensus 6/6 confirmed between primary review and Codex

#### Deferred to TODOS.md

No entries were written because this repo currently has no `TODOS.md`. Deferred
items remain explicit in the plan instead:
- CUDA/TensorRT implementation work, beyond milestone definition
- Generic motion-reference tracking beyond the standing-placeholder contract
- Visual-report redesign beyond metadata/schema compatibility
- New Python API surface beyond truth-alignment of existing docs/examples
