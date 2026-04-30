# /autoplan Restore Point
Captured: 2026-04-24T16:06:27+08:00 | Branch: main | Commit: 0ec155d

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
---
phase: 04-make-the-visual-harness-phase-aware-with-lag-selectable-phas
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - crates/robowbc-cli/src/main.rs
  - configs/showcase/gear_sonic_real.toml
  - configs/showcase/decoupled_wbc_real.toml
  - configs/showcase/wbc_agile_real.toml
  - scripts/roboharness_report.py
  - scripts/generate_policy_showcase.py
  - scripts/validate_site_bundle.py
  - tests/test_roboharness_report.py
  - tests/test_policy_showcase.py
  - tests/test_validate_site_bundle.py
  - docs/roboharness-integration.md
  - docs/configuration.md
autonomous: true
requirements: []
must_haves:
  truths:
    - "Velocity showcase runs carry explicit named phase metadata from the authored schedule through the CLI run artifacts."
    - "Proof packs capture phase midpoint and phase-end overlays plus bounded positive-lag variants (`+0..+5`), while generic evidence checkpoints remain secondary diagnostics."
    - "Tracking demos only become phase-aware when an explicit sibling `.phases.toml` sidecar is present; otherwise they keep the current generic checkpoint flow."
    - "The static detail page becomes phase-first, defaults to `+3` lag review, and is protected by deterministic Python tests plus the existing site-bundle smoke path."
  artifacts:
    - path: "crates/robowbc-cli/src/main.rs"
      provides: "Named velocity-phase metadata and tick windows in the run-report/replay artifacts."
      min_lines: 80
    - path: "scripts/roboharness_report.py"
      provides: "Phase-aware checkpoint selection, lag-variant screenshot capture, and proof-pack manifest fields."
      min_lines: 120
    - path: "scripts/generate_policy_showcase.py"
      provides: "Phase-first detail-page rendering and lag-selector UI."
      min_lines: 100
    - path: "tests/test_roboharness_report.py"
      provides: "Direct regression coverage for phase selection, lag bounding, and tracking-sidecar fallback."
      min_lines: 60
    - path: "docs/roboharness-integration.md"
      provides: "Operator-facing documentation of the phase-aware proof-pack contract and validation flow."
      min_lines: 25
  key_links:
    - from: "crates/robowbc-cli/src/main.rs"
      to: "scripts/roboharness_report.py"
      via: "shared `phase_timeline` / per-frame phase metadata in the emitted run artifacts"
      pattern: "phase_timeline|phase_name"
    - from: "scripts/roboharness_report.py"
      to: "scripts/generate_policy_showcase.py"
      via: "proof-pack manifest fields for phase checkpoints, lag options, and diagnostics"
      pattern: "phase_checkpoints|diagnostic_checkpoints|default_lag_ticks"
    - from: "scripts/generate_policy_showcase.py"
      to: "scripts/validate_site_bundle.py"
      via: "HTML ids and asset expectations for the phase-first detail page"
      pattern: "phase-timeline|phase-lag-selector|proof_pack_manifest"
---

<objective>
Make the proof-pack and showcase detail pages read like the staged locomotion
demo the user actually commanded.

Purpose: carry authoritative phase boundaries from the showcase configs through
the CLI artifacts, capture phase-aware midpoint/end screenshots with bounded
positive lag review, and present the result as a phase-first static detail page
instead of a flat checkpoint strip.
Output: named velocity schedule metadata, phase-aware proof-pack manifests and
lag assets, a phase-first detail-page contract, regression tests, and updated
docs.
</objective>

<execution_context>
@$HOME/.codex/get-shit-done/workflows/execute-plan.md
@$HOME/.codex/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/CONTEXT.md
@.planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-CONTEXT.md
@.planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-RESEARCH.md
@.planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-VALIDATION.md
@crates/robowbc-cli/src/main.rs
@configs/showcase/gear_sonic_real.toml
@configs/showcase/decoupled_wbc_real.toml
@configs/showcase/wbc_agile_real.toml
@configs/showcase/gear_sonic_tracking_real.toml
@scripts/roboharness_report.py
@scripts/generate_policy_showcase.py
@scripts/validate_site_bundle.py
@tests/test_policy_showcase.py
@tests/test_validate_site_bundle.py
@docs/roboharness-integration.md
@docs/configuration.md
@Makefile
</context>

<threat_model>
## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| Rust CLI artifacts -> Python proof-pack generator | `run_report.json` and `run_report_replay_trace.json` become the source of truth for phase review | phase names, tick windows, replay frames, relative artifact paths |
| Optional tracking sidecar -> proof-pack generator | checked-in phase metadata augments tracking demos without changing runtime semantics | phase names, start/end ticks, default lag |
| Proof-pack manifest -> static detail page | the site renders phase cards and lag selectors from manifest data only | relative image paths, lag options, diagnostics labels |

## STRIDE Register

| threat_id | category | component | disposition | mitigation_plan |
|-----------|----------|-----------|-------------|-----------------|
| T-04-01 | Tampering | proof-pack manifest and tracking sidecar metadata | mitigate | Accept only repo-relative sibling sidecars, keep lag/checkpoint asset paths relative to the proof-pack root, and fail tests/site-smoke when required phase or lag assets are missing. |
| T-04-02 | Repudiation | phase-end overlay provenance | mitigate | Serialize explicit `phase_name`, `phase_end_tick`, `lag_ticks`, and selection reasons in the manifest and surface them in the detail page/tests so every overlay is traceable back to a recorded tick. |
| T-04-03 | Denial of Service | screenshot capture fan-out | mitigate | Hard-cap positive lag capture to `+0..+5`, keep the existing three camera slots (`track`, `side`, `top`), and add report-script tests that enforce the bounded asset count. |
</threat_model>

<tasks>

<task type="auto">
  <name>Task 1: Add named phase metadata to showcase velocity schedules and CLI artifacts</name>
  <files>crates/robowbc-cli/src/main.rs, configs/showcase/gear_sonic_real.toml, configs/showcase/decoupled_wbc_real.toml, configs/showcase/wbc_agile_real.toml</files>
  <read_first>crates/robowbc-cli/src/main.rs, configs/showcase/gear_sonic_real.toml, configs/showcase/decoupled_wbc_real.toml, configs/showcase/wbc_agile_real.toml, .planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-CONTEXT.md, docs/configuration.md</read_first>
  <action>Extend `VelocityScheduleSegmentConfig` with `phase_name: Option<String>`. Compute a shared `phase_timeline` contract for `velocity_schedule` runs with `phase_name`, `start_tick`, `midpoint_tick`, and `end_tick`, and serialize that timeline plus a per-frame phase reference on both `RunReport` and `ReplayTrace`. Update the three showcase velocity configs so the segment order is exactly `stand` (1.0s, `[0.0, 0.0, 0.0]` -> `[0.0, 0.0, 0.0]`), `accelerate` (2.0s, `[0.0, 0.0, 0.0]` -> `[0.6, 0.0, 0.0]`), `turn` (1.0s, `[0.6, 0.0, -1.5707964]` -> `[0.6, 0.0, -1.5707964]`), `run` (3.0s, `[0.6, 0.0, 0.0]` -> `[1.0, 0.0, 0.0]`), and `settle` (1.0s, `[1.0, 0.0, 0.0]` -> `[0.0, 0.0, 0.0]`).</action>
  <verify>cargo test -p robowbc-cli velocity_schedule</verify>
  <acceptance_criteria>
    - each showcase velocity config contains `phase_name = "stand"`, `phase_name = "accelerate"`, `phase_name = "turn"`, `phase_name = "run"`, and `phase_name = "settle"`
    - `crates/robowbc-cli/src/main.rs` serializes a top-level `phase_timeline` for `velocity_schedule` runs
    - replay/report frames carry the current phase reference so downstream Python does not re-derive names from flattened numeric `command_data`
  </acceptance_criteria>
  <done>The authoritative run artifacts now know which phase each showcase tick belongs to</done>
</task>

<task type="auto">
  <name>Task 2: Generate phase-aware proof-pack checkpoints, lag variants, and direct report tests</name>
  <files>scripts/roboharness_report.py, tests/test_roboharness_report.py</files>
  <read_first>scripts/roboharness_report.py, crates/robowbc-cli/src/main.rs, .planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-CONTEXT.md, .planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-RESEARCH.md, tests/test_nvidia_benchmarks.py</read_first>
  <action>Teach `scripts/roboharness_report.py` to prefer the CLI-emitted `phase_timeline` for `velocity_schedule` runs, look for an optional sibling tracking sidecar named `<config stem>.phases.toml` for tracking demos, and otherwise fall back to the current generic checkpoint selector. Emit primary `phase_checkpoints` at each phase midpoint and phase end, keep anomaly evidence in a separate `diagnostic_checkpoints` collection, and for each phase-end checkpoint save positive-lag variants `lag_0` through `lag_5` that reuse the phase-end target frame and actual frames at `end_tick + offset`. Keep screenshot filenames `track_rgb.png`, `side_rgb.png`, and `top_rgb.png`, but retune the velocity-demo `track` and `top` camera presets so they behave like a chase/rear-three-quarter view and a path-oriented top view. Add `tests/test_roboharness_report.py` to lock the manifest keys `phase_timeline`, `phase_checkpoints`, `diagnostic_checkpoints`, `lag_options`, and `default_lag_ticks`, plus tracking-sidecar fallback behavior.</action>
  <verify>python3 -m unittest tests.test_roboharness_report</verify>
  <acceptance_criteria>
    - `proof_pack_manifest.json` records `phase_timeline`, `phase_checkpoints`, `diagnostic_checkpoints`, `lag_options`, and `default_lag_ticks`
    - lag captures are bounded to positive offsets `0..5` and stay under proof-pack-relative paths
    - tracking runs without a sidecar keep the current generic checkpoint flow, while a sidecar-backed test fixture produces named tracking phases
  </acceptance_criteria>
  <done>The proof-pack artifacts now tell a phase narrative and support deterministic lag-at-phase-end review</done>
</task>

<task type="auto">
  <name>Task 3: Render phase-first detail pages and harden bundle validation</name>
  <files>scripts/generate_policy_showcase.py, scripts/validate_site_bundle.py, tests/test_policy_showcase.py, tests/test_validate_site_bundle.py</files>
  <read_first>scripts/generate_policy_showcase.py, scripts/validate_site_bundle.py, tests/test_policy_showcase.py, tests/test_validate_site_bundle.py, scripts/roboharness_report.py</read_first>
  <action>Consume the new proof-pack manifest fields in `scripts/generate_policy_showcase.py` and render a phase-first detail-page contract: `id="phase-timeline"` for the ordered phase narrative, `id="phase-lag-selector"` with buttons `+0` through `+5`, `data-default-lag="3"` for the default selection, and a smaller diagnostics section for anomaly checkpoints. Only show tracking-phase UI when the manifest has explicit phase entries from a sidecar; otherwise keep the current generic checkpoint presentation. Update `scripts/validate_site_bundle.py` plus the existing HTML tests so MuJoCo detail pages fail validation when phase-aware manifest fields or lag assets are missing.</action>
  <verify>python3 -m unittest tests.test_policy_showcase tests.test_validate_site_bundle</verify>
  <acceptance_criteria>
    - velocity detail pages contain `id="phase-timeline"` and `id="phase-lag-selector"` with buttons labeled `+0` through `+5`
    - detail pages keep the local `run.rrd` embed path and `proof_pack_manifest.json` link contract intact
    - `scripts/validate_site_bundle.py` rejects MuJoCo proof packs that are missing phase-aware manifest sections or lag assets
  </acceptance_criteria>
  <done>The static site now presents the new phase-aware review flow and the bundle validator enforces it</done>
</task>

<task type="auto">
  <name>Task 4: Document the phase-aware proof-pack and config contract</name>
  <files>docs/roboharness-integration.md, docs/configuration.md</files>
  <read_first>docs/roboharness-integration.md, docs/configuration.md, .planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-CONTEXT.md, .planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-RESEARCH.md, Makefile</read_first>
  <action>Update `docs/roboharness-integration.md` and `docs/configuration.md` to document `runtime.velocity_schedule.segments[].phase_name`, the emitted proof-pack manifest fields `phase_timeline`, `phase_checkpoints`, `diagnostic_checkpoints`, `lag_options`, and `default_lag_ticks`, the optional sibling tracking sidecar format `<config stem>.phases.toml`, the default `+3` lag review flow, and `make showcase-verify` as the end-to-end validation command for the phase-aware bundle.</action>
  <verify>rg -n "phase_name|phase_timeline|diagnostic_checkpoints|phases\\.toml|showcase-verify|\\+3" docs/roboharness-integration.md docs/configuration.md</verify>
  <acceptance_criteria>
    - docs describe how named schedule phases reach the proof pack and static site
    - docs describe the optional tracking sidecar format and the fallback to generic checkpoints when it is absent
    - docs call out the default `+3` lag review behavior and `make showcase-verify`
  </acceptance_criteria>
  <done>Operators and contributors can understand and validate the new phase-aware contract without reading the implementation first</done>
</task>

</tasks>

<verification>
- [ ] `cargo test -p robowbc-cli velocity_schedule` passes
- [ ] `python3 -m unittest tests.test_roboharness_report tests.test_policy_showcase tests.test_validate_site_bundle` passes
- [ ] `python3 -m py_compile scripts/roboharness_report.py scripts/generate_policy_showcase.py scripts/validate_site_bundle.py` passes
- [ ] `make showcase-verify` passes
</verification>

<success_criteria>
- All tasks completed
- Velocity demos read as a staged phase narrative with explicit phase-end lag review
- Tracking demos stay generic unless an explicit `.phases.toml` sidecar is present
- The CLI artifacts, proof-pack manifest, detail-page HTML, and site validator all agree on one phase-aware contract
- The phase-aware bundle is covered by deterministic Python tests and the existing end-to-end showcase verification path
</success_criteria>

<output>
After completion, create `.planning/phases/04-make-the-visual-harness-phase-aware-with-lag-selectable-phas/04-01-SUMMARY.md`
</output>
