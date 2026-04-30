# /autoplan Restore Point
Captured: 2026-04-27 11:48:31 CST | Branch: dongxu-dev-0424 | Commit: 1e120d39

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
---
phase: 70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn
plan: 02
type: execute
wave: 2
depends_on:
  - 70.1-01
files_modified:
  - tmp/phase70_1_harness/g1_wbc/suite_report_representative.json
  - tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json
  - tmp/phase70_1_harness/realman/suite_report_representative.json
  - tests/visual_harness/g1_baseline.md
  - tests/realman/realman_baseline.md
  - docs/grasp/sonic_paths.md
  - docs/runtime.md
  - .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-02-SUMMARY.md
  - .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-VERIFICATION.md
  - .planning/ROADMAP.md
  - .planning/STATE.md
autonomous: false
requirements:
  - P70.1REQ-2
  - P70.1REQ-3
  - P70.1REQ-4
user_setup:
  - source activate.sh
  - Run on a GPU-capable workstation with an X11 display or `xvfb-run`
must_haves:
  truths:
    - "Phase 70.1 records three representative evidence sets: G1 WBC, G1 SONIC compat, and Realman collateral guard."
    - "The G1 SONIC compat result is compared explicitly against the validated WBC baseline; any regression is named with exact case IDs, failure taxonomy, and timing deltas."
    - "Realman remains the shared-path collateral regression guard and may not be skipped just because it has no SONIC backend."
    - "Compat evidence does not erase the current `sonic_path=native` blocker list; docs and verification keep that blocker list explicit unless it is actually closed."
  artifacts:
    - path: tmp/phase70_1_harness/g1_wbc/suite_report_representative.json
      provides: G1 supported-backend representative baseline captured at final HEAD
      contains: '"total_cases": 8'
    - path: tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json
      provides: G1 SONIC compat representative evidence captured at final HEAD
      contains: '"total_cases": 8'
    - path: tmp/phase70_1_harness/realman/suite_report_representative.json
      provides: Realman collateral regression guard captured at final HEAD
      contains: '"total_cases": 8'
    - path: .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-VERIFICATION.md
      provides: final verdict/taxonomy/metric analysis and native-blocker status
      contains: G1 sonic_compat representative suite
---

<objective>
Generate the actual evidence bundle that Phase 70.1 exists to produce: a
side-by-side G1 representative comparison of the supported WBC path versus the
new SONIC compat path, plus a Realman collateral-regression rerun and a written
verification analysis that records exact verdict, failure taxonomy, execution
health, and timing deltas. Keep `sonic_path=native` explicitly blocked in docs
and verification even if compat clears the evidence bar.
</objective>

<execution_context>
@/home/mi/.codex/get-shit-done/workflows/execute-plan.md
@/home/mi/.codex/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-CONTEXT.md
@.planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-RESEARCH.md
@.planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-PATTERNS.md
@.planning/phases/70-compare-sonic-compatible-vs-sonic-native-g1-waist-planning-p/70-VERIFICATION.md
@docs/grasp/sonic_paths.md
@docs/runtime.md
@tests/visual_harness/g1_runner.py
@tests/realman/visual_harness_runner.py
@tests/visual_harness/baseline.py
@tests/visual_harness/g1_baseline.md
@tests/realman/realman_baseline.md
@tmp/phase69_70_harness_serial/g1/0424_1931/suite_report_representative.json
@tmp/phase69_70_harness_serial/realman/0424_1944/suite_report_representative.json

Locked decisions for traceability:
- `LD-70.1-04`: Write all new representative evidence under `tmp/phase70_1_harness/*`,
  not under `.planning/`.
- `LD-70.1-05`: Refresh baseline markdown only from final chosen `HEAD`, or
  leave it untouched.
- `LD-70.1-06`: Realman is mandatory collateral-regression evidence, not an
  optional afterthought.
- `LD-70.1-07`: Compat proof does not imply native readiness; native blockers
  stay explicit in docs and verification.
</context>

<tasks>
<task type="auto">
  <name>Task 1: Run and compare the G1 representative suites for WBC and SONIC compat</name>
  <files>tmp/phase70_1_harness/g1_wbc/suite_report_representative.json, tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json, .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-02-SUMMARY.md, .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-VERIFICATION.md</files>
  <read_first>tests/visual_harness/g1_runner.py, tests/visual_harness/baseline.py, tests/visual_harness/g1_baseline.md, .planning/phases/70-compare-sonic-compatible-vs-sonic-native-g1-waist-planning-p/70-VERIFICATION.md, tmp/phase69_70_harness_serial/g1/0424_1931/suite_report_representative.json</read_first>
  <action>Run the representative G1 suite twice: once with `--control-backend wbc --output-root tmp/phase70_1_harness/g1_wbc` and once with `--control-backend sonic --sonic-path compat --output-root tmp/phase70_1_harness/g1_sonic_compat`. After both runs finish, compare `suite_report_representative.json` plus the per-case `autonomous_report.json` files against the Phase 70 G1 baseline at `tmp/phase69_70_harness_serial/g1/0424_1931/suite_report_representative.json`. Record exact case-status deltas, failure-taxonomy deltas, mean `planning_time_s`, mean `reik_wall_time_s`, and mean `combined_effective_latency_s`. If SONIC compat regresses versus WBC, write the exact regression boundary into `70.1-VERIFICATION.md` instead of claiming parity.</action>
  <verify>
    <automated>xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase70_1_harness/g1_wbc --control-backend wbc</automated>
    <automated>xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase70_1_harness/g1_sonic_compat --control-backend sonic --sonic-path compat</automated>
  </verify>
  <acceptance_criteria>
    - `tmp/phase70_1_harness/g1_wbc/suite_report_representative.json` contains `"total_cases": 8`
    - `tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json` contains `"total_cases": 8`
    - `tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json` contains `"execution_backend": "sonic_compat"`
    - `70.1-VERIFICATION.md` contains `G1 WBC representative suite` and `G1 sonic_compat representative suite`
  </acceptance_criteria>
  <done>The phase has a truthful side-by-side G1 evidence bundle instead of only a deploy/runtime assertion.</done>
</task>

<task type="auto">
  <name>Task 2: Rerun the Realman representative suite as the collateral-regression guard</name>
  <files>tmp/phase70_1_harness/realman/suite_report_representative.json, .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-VERIFICATION.md</files>
  <read_first>tests/realman/visual_harness_runner.py, tests/realman/realman_baseline.md, .planning/phases/70-compare-sonic-compatible-vs-sonic-native-g1-waist-planning-p/70-VERIFICATION.md, tmp/phase69_70_harness_serial/realman/0424_1944/suite_report_representative.json</read_first>
  <action>Run the Realman representative suite into `tmp/phase70_1_harness/realman`, compare its verdict envelope, fail taxonomy, execution-error count, and `planning_time_s` deltas against `tmp/phase69_70_harness_serial/realman/0424_1944/suite_report_representative.json`, and record the exact comparison in `70.1-VERIFICATION.md`. If Realman shows a pass regression or new execution error after the shared harness/reporting changes, stop and record that as a blocking collateral regression.</action>
  <verify>
    <automated>xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.realman.visual_harness_runner --suite representative --output-root tmp/phase70_1_harness/realman</automated>
  </verify>
  <acceptance_criteria>
    - `tmp/phase70_1_harness/realman/suite_report_representative.json` contains `"total_cases": 8`
    - `70.1-VERIFICATION.md` contains `Realman representative suite`
    - `70.1-VERIFICATION.md` records either `no pass regressions` or the exact blocking case IDs and failure codes
  </acceptance_criteria>
  <done>The shared-path collateral guard is re-proven instead of assumed.</done>
</task>

<task type="auto">
  <name>Task 3: Refresh final docs, baseline markdown, and the phase verification write-up from the chosen HEAD</name>
  <files>tests/visual_harness/g1_baseline.md, tests/realman/realman_baseline.md, docs/grasp/sonic_paths.md, docs/runtime.md, .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-02-SUMMARY.md, .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-VERIFICATION.md, .planning/ROADMAP.md, .planning/STATE.md</files>
  <read_first>tests/visual_harness/g1_baseline.md, tests/realman/realman_baseline.md, docs/grasp/sonic_paths.md, docs/runtime.md, .planning/phases/70-compare-sonic-compatible-vs-sonic-native-g1-waist-planning-p/70-VERIFICATION.md</read_first>
  <action>Only after the final chosen representative outputs are available, refresh `tests/visual_harness/g1_baseline.md` and `tests/realman/realman_baseline.md` from those outputs or leave them untouched if they are not being committed. Update `docs/grasp/sonic_paths.md` and `docs/runtime.md` so they describe the new representative evidence truthfully: whether SONIC compat matched the WBC envelope or where it regressed, and that `sonic_path=native` is still blocked with its current blocker list. Write `70.1-VERIFICATION.md` and `70.1-02-SUMMARY.md` with the final verdict envelope, failure taxonomy, timing deltas, accepted regressions if any, and the explicit native blocker status. Update roadmap/state only after those artifacts reflect the final chosen outputs.</action>
  <verify>
    <automated>rg -n "sonic_path=compat|sonic_path=native" docs/grasp/sonic_paths.md docs/runtime.md</automated>
    <automated>rg -n "G1 WBC representative suite|G1 sonic_compat representative suite|Realman representative suite|native blocker" .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-VERIFICATION.md</automated>
  </verify>
  <acceptance_criteria>
    - `docs/grasp/sonic_paths.md` contains `sonic_path=compat` and `sonic_path=native`
    - `docs/runtime.md` contains `--control-backend sonic --sonic-path compat`
    - `70.1-VERIFICATION.md` contains `native blocker` or an equivalent explicit blocker section
    - any committed baseline markdown files contain a final `Generated:` timestamp and `Suite`: `representative`
  </acceptance_criteria>
  <done>The phase closes with final-head evidence, not stale artifact carry-forward.</done>
</task>
</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| final HEAD -> generated evidence | Stale or mixed-run artifacts can be mistaken for final verification evidence |
| G1 primary evidence -> Realman collateral evidence | Shared-path regressions can be hidden if Realman is skipped |
| compat evidence -> native docs | A successful compat run can be overclaimed as native readiness |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-70.1-02-1 | Tampering | `tmp/phase70_1_harness/*` artifacts | mitigate | Use stable output roots per backend and compare them against the explicit Phase 70 baseline artifacts before updating docs |
| T-70.1-02-2 | Repudiation | Realman collateral evidence | mitigate | Make the Realman representative suite mandatory and record exact pass/fail and execution-error deltas in `70.1-VERIFICATION.md` |
| T-70.1-02-3 | Spoofing | `docs/grasp/sonic_paths.md` / `docs/runtime.md` | mitigate | Keep the `native` blocker list explicit unless the phase truly closes those blockers, and never let compat evidence imply native readiness |
</threat_model>

<verification>
1. `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase70_1_harness/g1_wbc --control-backend wbc`
2. `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase70_1_harness/g1_sonic_compat --control-backend sonic --sonic-path compat`
3. `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.realman.visual_harness_runner --suite representative --output-root tmp/phase70_1_harness/realman`
4. Compare `tmp/phase70_1_harness/g1_wbc/suite_report_representative.json` against `tmp/phase69_70_harness_serial/g1/0424_1931/suite_report_representative.json` for missing cases, pass regressions, and new execution errors before treating WBC as the refreshed baseline.
5. Compare `tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json` against the refreshed WBC run plus the per-case `autonomous_report.json` files for failure-taxonomy and timing deltas.
6. Compare `tmp/phase70_1_harness/realman/suite_report_representative.json` against `tmp/phase69_70_harness_serial/realman/0424_1944/suite_report_representative.json` for missing cases, pass regressions, new execution errors, and `planning_time_s` movement.
7. `rg -n "sonic_path=compat|sonic_path=native" docs/grasp/sonic_paths.md docs/runtime.md`
8. `rg -n "G1 WBC representative suite|G1 sonic_compat representative suite|Realman representative suite|native blocker" .planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-VERIFICATION.md`
</verification>

<success_criteria>
- Three final representative evidence sets exist: G1 WBC, G1 SONIC compat, and Realman collateral guard.
- The phase records exact verdict envelopes, failure taxonomy, execution-health deltas, and timing deltas instead of vague parity claims.
- Any accepted SONIC compat regression is documented explicitly with case IDs and metric movement.
- Docs and verification keep `sonic_path=native` blocked unless the blocker list is truly closed.
</success_criteria>

<output>
After completion, create:
- `.planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-02-SUMMARY.md`
- `.planning/phases/70.1-close-phase-70-sonic-integration-gap-with-8-case-visual-harn/70.1-VERIFICATION.md`
</output>
