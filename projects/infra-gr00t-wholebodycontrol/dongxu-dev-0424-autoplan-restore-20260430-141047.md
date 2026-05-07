# /autoplan Restore Point
Captured: 2026-04-30T06:10:47Z | Branch: dongxu-dev-0424 | Commit: 365c70a7

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
---
phase: 71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras
plan: 03
type: execute
wave: 3
depends_on:
  - 71-01
  - 71-02
files_modified:
  - tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json
  - tmp/phase71_e2e/official/X36_Y28_Z13/official_sonic_report.json
  - tmp/phase71_harness/g1_wbc/suite_report_representative.json
  - tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json
  - tmp/phase71_harness/realman/suite_report_representative.json
  - tmp/phase71_harness/g1_sonic_native/suite_report_representative.json
  - .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-COMPAT-TRIALS.md
  - .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-DECISION.md
  - .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md
  - docs/grasp/sonic_paths.md
  - docs/runtime.md
  - docs/architecture.md
  - .planning/ROADMAP.md
  - .planning/STATE.md
  - .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-03-SUMMARY.md
autonomous: false
requirements:
  - P71REQ-1
  - P71REQ-4
  - P71REQ-5
  - P71REQ-6
  - P71REQ-7
user_setup:
  - Run the same-case and representative visual-harness evidence commands on a GPU-capable workstation.
  - Use a live X11 display or `xvfb-run -a` where the runner supports it.
  - Do not run heavy simulations from a constrained remote executor.
  - Only run native same-case or representative commands if Plan 02 intentionally relaxed the guard for evidence collection and `71-NATIVE-AUDIT.md` plus `71-UPSTREAM-BUGS.md` justify it.
must_haves:
  truths:
    - "The final Phase 71 decision is either `go-native-candidate` with same-case plus representative parity proof, or `keep-native-blocked` with exact blocker analysis."
    - "`sonic_path=compat` must actually complete at least one named simulated G1 grasp at `HEAD`; a routed command, non-pass report, or screenshot-only artifact is not enough."
    - "If `sonic_path=compat` does not pass on the first attempt, the executor must spend a bounded extra compute/time budget on instrumented trials before declaring it blocked."
    - "The 8-case visual harness runs for both G1 and Realman are required no-regression gates."
    - "Performance analysis includes exact case IDs and before/after values for `planning_time_s`, `reik_wall_time_s` when present, `combined_effective_latency_s` when present, and `whole_case_wall_time_s`."
    - "Any accepted regression records a root-cause hypothesis and an acceptance boundary; otherwise regressions block the native decision."
    - "Compat, official sim2sim, and native-candidate evidence remain separate artifacts and separate claims."
  artifacts:
    - path: .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-DECISION.md
      provides: final native go/no-go decision
      contains: Decision
    - path: .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md
      provides: same-case, representative, and performance analysis
      contains: Performance Regression Analysis
    - path: tmp/phase71_harness/g1_wbc/suite_report_representative.json
      provides: final supported G1 WBC representative guard
      contains: '"total_cases": 8'
    - path: tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json
      provides: final G1 sonic_compat simulated grasp proof
      contains: '"pass_count"'
    - path: .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-COMPAT-TRIALS.md
      provides: bounded trial-and-error ledger for sonic_compat same-case and representative attempts
      contains: Trial Ledger
    - path: tmp/phase71_harness/realman/suite_report_representative.json
      provides: final Realman representative collateral guard
      contains: '"total_cases": 8'
  key_links:
    - from: .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-DECISION.md
      to: .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-NATIVE-AUDIT.md
      via: decision references every open or closed native blocker ID
      pattern: NATIVE-FEEDBACK-FRAMING|NATIVE-REPRESENTATIVE-PARITY
    - from: .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md
      to: tmp/phase71_harness/realman/suite_report_representative.json
      via: Realman suite is the collateral no-regression gate for shared-path changes
      pattern: Realman representative suite
---

<objective>
Make the Phase 71 native-path decision from evidence. This plan runs the
same-case and representative no-regression gates, compares performance metrics,
and writes either a go decision for a native candidate or an explicit
keep-blocked closeout. It cannot close on screenshots or prose-only reasoning.
</objective>

<execution_context>
@/home/mi/.codex/get-shit-done/workflows/execute-plan.md
@/home/mi/.codex/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-RESEARCH.md
@.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VALIDATION.md
@.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-NATIVE-AUDIT.md
@.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-UPSTREAM-BUGS.md
@gr00t_manipulation/automation/g1_sonic_e2e.py
@scripts/automation/g1_sonic_e2e.py
@tests/visual_harness/g1_runner.py
@tests/realman/visual_harness_runner.py
@tests/manipulation/test_visual_harness_contract.py
@tmp/phase70_2_harness/g1_wbc/suite_report_representative.json
@tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json
@tmp/phase70_2_harness/realman/suite_report_representative.json

Locked decisions for traceability:
- `LD-71-08`: both robot 8-case representative suites are required for closeout.
- `LD-71-09`: native can only become a candidate after same-case plus representative evidence, not from transport proof alone.
- `LD-71-10`: accepted regressions require detailed numeric analysis.
- `LD-71-11`: Phase 71 success requires `sonic_compat` to actually grasp in sim on at least one named G1 case.
- `LD-71-12`: `sonic_compat` grasp success gets a bounded workstation-local trial budget before the phase is allowed to declare it blocked.
</context>

<tasks>
<task type="auto">
  <name>Task 1: Collect same-case evidence for compat, official, and native only if native is intentionally unblocked for evidence</name>
  <files>tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json, tmp/phase71_e2e/compat_trials/trial_01/X36_Y28_Z13/sonic_compat_report.json, tmp/phase71_e2e/official/X36_Y28_Z13/official_sonic_report.json, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-COMPAT-TRIALS.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md</files>
  <read_first>.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-NATIVE-AUDIT.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-UPSTREAM-BUGS.md, gr00t_manipulation/automation/g1_sonic_e2e.py, scripts/automation/g1_sonic_e2e.py, tmp/phase70_3_actual_compat/X36_Y28_Z13/sonic_compat_report.json, tmp/official_sonic_e2e_api_v11/X36_Y28_Z13/official_sonic_report.json</read_first>
  <behavior>
    - Compat and official same-case evidence are refreshed under Phase 71 artifact roots.
    - The compat same-case is an acceptance gate: `X36_Y28_Z13` must produce a passing `sonic_compat` report, or the phase cannot be marked successful.
    - If the first compat same-case attempt fails, the executor spends up to 10 focused workstation-local trials before declaring the compat grasp blocked.
    - Every compat same-case trial is logged in `71-COMPAT-TRIALS.md` with command, artifact root, hypothesis, repo-owned change or parameter, verdict, failure reason, `planning_time_s`, `whole_case_wall_time_s`, and next action.
    - Native same-case evidence is run only if the guard was intentionally relaxed by Plan 02; otherwise the verification records `native_same_case_status=not_run_guard_blocked`.
    - Evidence claims remain separate: compat report fields cannot satisfy official or native fields.
  </behavior>
  <action>Create or refresh `71-COMPAT-TRIALS.md` with `# Trial Ledger` before running compat evidence. Run `UV_PROJECT_ENVIRONMENT=.venv uv run python scripts/automation/g1_sonic_e2e.py compat --case-id X36_Y28_Z13 --output-root tmp/phase71_e2e/compat` and archive the resulting `tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json`. Do not use `--allow-fail-verdict` for this canonical acceptance run. If it passes, log the pass in `71-COMPAT-TRIALS.md` as the acceptance attempt and continue. If it fails or produces a non-pass report, spend up to 10 focused same-case compat trials under `tmp/phase71_e2e/compat_trials/trial_01` through `tmp/phase71_e2e/compat_trials/trial_10`. Each trial must test one explicit hypothesis using a repo-owned code change or runtime parameter, run the same compat command shape against that trial output root, optionally add `--allow-fail-verdict` only to archive diagnostics for a failed trial, and append a ledger entry with the exact command, artifact path, hypothesis, changed variable, verdict, failure reason, `planning_time_s`, `whole_case_wall_time_s`, and next action. When a trial passes, rerun the winning configuration into the canonical `tmp/phase71_e2e/compat` root and require that canonical artifact to pass. Stop only after the canonical compat same-case passes or after all 10 trials are exhausted with a concrete blocker in `71-COMPAT-TRIALS.md`; exhaustion blocks Phase 71 success. The compat acceptance artifact must contain `expected_execution_backend=sonic_compat`, `case_verdict=pass`, `grasp_case_completed=true`, and a successful named grasp terminal state such as `grasp_terminal_state=HOLDING`; missing terminal-grasp fields block acceptance even when `case_verdict=pass`. Run `UV_PROJECT_ENVIRONMENT=.venv uv run python scripts/automation/g1_sonic_e2e.py official --case-id X36_Y28_Z13 --output-root tmp/phase71_e2e/official --auto-startup --allow-fail-verdict` from a valid live-display environment and archive `tmp/phase71_e2e/official/X36_Y28_Z13/official_sonic_report.json`. If Plan 02 did not intentionally relax native for evidence collection, do not invent a native run; write `native_same_case_status=not_run_guard_blocked` in `71-VERIFICATION.md`. If Plan 02 did relax native for evidence collection, run the native same-case through the exact native entrypoint created by that plan, archive it under `tmp/phase71_e2e/native/X36_Y28_Z13/`, and require the report to contain `case_id=X36_Y28_Z13`, `expected_execution_backend=sonic_native` or equivalent native identity, fresh feedback, startup truth, and terminal grasp status. In `71-VERIFICATION.md`, write a table comparing compat, official, and native same-case verdicts, startup fields, feedback fields, lower-body/grounding fields, and timing fields.</action>
  <verify>
    <automated>UV_PROJECT_ENVIRONMENT=.venv uv run python scripts/automation/g1_sonic_e2e.py compat --case-id X36_Y28_Z13 --output-root tmp/phase71_e2e/compat</automated>
    <automated>UV_PROJECT_ENVIRONMENT=.venv uv run python scripts/automation/g1_sonic_e2e.py official --case-id X36_Y28_Z13 --output-root tmp/phase71_e2e/official --auto-startup --allow-fail-verdict</automated>
    <automated>UV_PROJECT_ENVIRONMENT=.venv uv run python -c "import json, pathlib; compat=pathlib.Path('tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json'); official=pathlib.Path('tmp/phase71_e2e/official/X36_Y28_Z13/official_sonic_report.json'); assert compat.exists(); assert official.exists(); c=json.loads(compat.read_text()); o=json.loads(official.read_text()); assert c['case_id']=='X36_Y28_Z13'; assert c['expected_execution_backend']=='sonic_compat'; assert c['case_verdict']=='pass'; assert c['grasp_case_completed'] is True; assert c['grasp_terminal_state'] in ('HOLDING', 'PASS', 'pass'); assert o.get('case_id')=='X36_Y28_Z13'"</automated>
  </verify>
  <acceptance_criteria>
    - `tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json` exists
    - `tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json` contains `"expected_execution_backend": "sonic_compat"`
    - `tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json` contains `"case_verdict": "pass"`
    - `tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json` contains `"grasp_case_completed": true`
    - `tmp/phase71_e2e/compat/X36_Y28_Z13/sonic_compat_report.json` contains a successful `grasp_terminal_state`
    - `71-COMPAT-TRIALS.md` contains `# Trial Ledger`
    - If the first compat attempt failed, `71-COMPAT-TRIALS.md` records every attempted `trial_01` through the final attempted trial with hypothesis, artifact path, verdict, metrics, and next action
    - `tmp/phase71_e2e/official/X36_Y28_Z13/official_sonic_report.json` exists
    - `71-VERIFICATION.md` contains `Same-Case Evidence`
    - `71-VERIFICATION.md` contains `sonic_compat_same_case_status=pass`
    - `71-VERIFICATION.md` contains `native_same_case_status`
    - `71-VERIFICATION.md` does not use compat evidence as official or native evidence
  </acceptance_criteria>
  <done>Phase 71 has refreshed same-case evidence or an explicit guard-blocked native status.</done>
</task>

<task type="auto">
  <name>Task 2: Run both robots' 8-case visual harnesses and compare performance metrics</name>
  <files>tmp/phase71_harness/g1_wbc/suite_report_representative.json, tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json, tmp/phase71_harness/g1_sonic_compat_trial_01/suite_report_representative.json, tmp/phase71_harness/realman/suite_report_representative.json, tmp/phase71_harness/g1_sonic_native/suite_report_representative.json, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-COMPAT-TRIALS.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md</files>
  <read_first>tests/visual_harness/g1_runner.py, tests/realman/visual_harness_runner.py, tests/manipulation/test_visual_harness_contract.py, tmp/phase70_2_harness/g1_wbc/suite_report_representative.json, tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json, tmp/phase70_2_harness/realman/suite_report_representative.json</read_first>
  <behavior>
    - G1 WBC, G1 compat, and Realman representative suites each run exactly 8 cases under deterministic Phase 71 roots.
    - The G1 compat representative suite proves actual simulated grasp behavior: `pass_count>=7`, `execution_error_count=0`, and `X36_Y28_Z13` remains pass.
    - If the first G1 compat representative suite fails for any reason, the executor spends up to 3 representative compat reruns under deterministic trial roots before declaring the representative gate blocked.
    - Representative compat reruns are logged in `71-COMPAT-TRIALS.md` with exact pass counts, execution-error counts, changed variable, metric deltas, and next action.
    - Native representative evidence is run only if native was intentionally relaxed for evidence collection; otherwise the report records `native_representative_status=not_run_guard_blocked`.
    - Performance comparison includes pass-case means and case-level deltas for `planning_time_s`, `reik_wall_time_s` when present, `combined_effective_latency_s` when present, and `whole_case_wall_time_s`.
    - New execution errors, pass-to-fail flips, or material timing regressions block the decision unless explicitly accepted with detailed analysis.
  </behavior>
  <action>Run the supported G1 WBC representative suite with `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase71_harness/g1_wbc --control-backend wbc --no-timestamp-output`. Run the G1 compat representative suite with `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase71_harness/g1_sonic_compat --control-backend sonic --sonic-path compat --no-timestamp-output`. The G1 compat suite is a success gate, not only a comparison artifact: it must report `total_cases=8`, `pass_count>=7`, `execution_error_count=0`, and `X36_Y28_Z13` must be `pass`. If the first compat representative suite fails for any reason, run up to 3 full representative compat reruns under `tmp/phase71_harness/g1_sonic_compat_trial_01` through `tmp/phase71_harness/g1_sonic_compat_trial_03`, log each rerun in `71-COMPAT-TRIALS.md`, and rerun the winning configuration into the canonical `tmp/phase71_harness/g1_sonic_compat` root before accepting it. Stop only after the canonical compat representative suite satisfies the gate or after all 3 representative reruns are exhausted with a concrete blocker. Run the Realman representative suite with `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.realman.visual_harness_runner --suite representative --output-root tmp/phase71_harness/realman --no-timestamp-output`. If native was intentionally relaxed for evidence collection, run the G1 native representative suite into `tmp/phase71_harness/g1_sonic_native` with the exact same G1 runner shape and `--control-backend sonic --sonic-path native`; otherwise write `native_representative_status=not_run_guard_blocked` in `71-VERIFICATION.md`. Compare G1 WBC against `tmp/phase70_2_harness/g1_wbc/suite_report_representative.json`, G1 compat against `tmp/phase70_1_harness/g1_sonic_compat/suite_report_representative.json`, and Realman against `tmp/phase70_2_harness/realman/suite_report_representative.json`. Treat any new `execution_error`, pass-to-non-pass flip, or pass-case mean increase larger than both `10%` and `0.05s` for `planning_time_s`, `reik_wall_time_s`, or `combined_effective_latency_s`, or larger than both `5%` and `2.0s` for `whole_case_wall_time_s`, as material. In `71-VERIFICATION.md`, include exact case IDs, before values, after values, deltas, and status `no regression`, `blocking regression`, or `accepted regression` for every metric family.</action>
  <verify>
    <automated>xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase71_harness/g1_wbc --control-backend wbc --no-timestamp-output</automated>
    <automated>xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase71_harness/g1_sonic_compat --control-backend sonic --sonic-path compat --no-timestamp-output</automated>
    <automated>xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.realman.visual_harness_runner --suite representative --output-root tmp/phase71_harness/realman --no-timestamp-output</automated>
    <automated>UV_PROJECT_ENVIRONMENT=.venv uv run python -c "import json, pathlib; paths=['tmp/phase71_harness/g1_wbc/suite_report_representative.json','tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json','tmp/phase71_harness/realman/suite_report_representative.json']; [(__import__('builtins').print(p), (_ for _ in ()).throw(AssertionError(p))) for p in paths if not pathlib.Path(p).exists()]; reports=[json.loads(pathlib.Path(p).read_text()) for p in paths]; assert all(r['total_cases']==8 for r in reports); compat=reports[1]; assert compat['execution_error_count']==0; assert compat['pass_count']>=7; assert any(row['case_id']=='X36_Y28_Z13' and row['status']=='pass' for row in compat['results'])"</automated>
  </verify>
  <acceptance_criteria>
    - `tmp/phase71_harness/g1_wbc/suite_report_representative.json` contains `"total_cases": 8`
    - `tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json` contains `"total_cases": 8`
    - `tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json` has `pass_count >= 7`
    - `tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json` has `execution_error_count == 0`
    - `tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json` records `X36_Y28_Z13` with `status=pass`
    - If representative compat reruns were needed, `71-COMPAT-TRIALS.md` records every attempted `g1_sonic_compat_trial_01` through the final attempted rerun with pass counts, error counts, metric deltas, and next action
    - `tmp/phase71_harness/realman/suite_report_representative.json` contains `"total_cases": 8`
    - `71-VERIFICATION.md` contains `G1 WBC representative suite`
    - `71-VERIFICATION.md` contains `G1 sonic_compat representative suite`
    - `71-VERIFICATION.md` contains `Realman representative suite`
    - `71-VERIFICATION.md` contains `Performance Regression Analysis`
    - `71-VERIFICATION.md` contains `planning_time_s`
    - `71-VERIFICATION.md` contains `whole_case_wall_time_s`
  </acceptance_criteria>
  <done>Phase 71 has dual-robot 8-case evidence and performance-regression analysis.</done>
</task>

<task type="auto">
  <name>Task 3: Write the final native go/no-go decision and update current-state docs</name>
  <files>.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-DECISION.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-COMPAT-TRIALS.md, docs/grasp/sonic_paths.md, docs/runtime.md, docs/architecture.md, .planning/ROADMAP.md, .planning/STATE.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-03-SUMMARY.md</files>
  <read_first>.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-NATIVE-AUDIT.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-UPSTREAM-BUGS.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md, .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-COMPAT-TRIALS.md, docs/grasp/sonic_paths.md, docs/runtime.md, docs/architecture.md, .planning/ROADMAP.md, .planning/STATE.md</read_first>
  <behavior>
    - The final decision has one explicit value: `go-native-candidate` or `keep-native-blocked`.
    - `go-native-candidate` is allowed only if same-case evidence, native representative evidence, G1 supported-path guard, Realman guard, G1 `sonic_compat` simulated-grasp proof, and performance metrics all pass or have accepted-regression analysis.
    - Phase 71 success is blocked if `sonic_compat` cannot produce a canonical same-case pass after the 10-trial same-case budget, or cannot produce canonical representative evidence after the 3-rerun representative budget.
    - If any blocker remains open or native evidence was not run because the guard stayed blocked, the decision is `keep-native-blocked`.
    - Docs and planning state reflect the decision without hiding accepted regressions or blocked evidence.
  </behavior>
  <action>Create `71-DECISION.md` with sections `## Decision`, `## Evidence Inputs`, `## Compat Sim Grasp Success`, `## Compat Trial Budget`, `## Native Blocker Status`, `## Regression Decision`, `## Accepted Regressions`, `## Submodule Boundary`, and `## Next Step`. If `sonic_compat` same-case or representative evidence does not prove a named simulated grasp pass, set the phase closeout status to blocked even if the native decision is keep-blocked. If the first compat attempt failed, the `## Compat Trial Budget` section must summarize the 10-trial same-case budget and, if used, the 3-rerun representative budget from `71-COMPAT-TRIALS.md`, including why the loop stopped. If native same-case or native representative evidence was not run because the guard stayed blocked, set `Decision: keep-native-blocked` and list the open blocker IDs from `71-NATIVE-AUDIT.md`. If native evidence was run and passes, set `Decision: go-native-candidate` only if `tmp/phase71_harness/g1_sonic_native/suite_report_representative.json` has `total_cases=8`, no new execution errors, no supported-path pass regressions, the G1 `sonic_compat` simulated grasp proof passes, and all material timing regressions are either absent or accepted with detailed case-level analysis. Update `docs/grasp/sonic_paths.md`, `docs/runtime.md`, and `docs/architecture.md` so they reflect the final Phase 71 decision and still keep compat, official sim2sim, and native claims separate. Update `.planning/ROADMAP.md` and `.planning/STATE.md` with the Phase 71 outcome, but do not mark native runnable unless `71-DECISION.md` says `Decision: go-native-candidate`. Write `71-03-SUMMARY.md` with the commands run, evidence artifacts, compat trial budget used, compat sim grasp verdict, performance verdict, and final decision.</action>
  <verify>
    <automated>rg -n "Decision: (go-native-candidate|keep-native-blocked)|Evidence Inputs|Compat Sim Grasp Success|Compat Trial Budget|Native Blocker Status|Regression Decision|Accepted Regressions|Submodule Boundary" .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-DECISION.md</automated>
    <automated>rg -n "sonic_path=native|sonic_compat|official SONIC|Phase 71" docs/grasp/sonic_paths.md docs/runtime.md docs/architecture.md .planning/ROADMAP.md .planning/STATE.md</automated>
    <automated>rg -n "Performance Regression Analysis|G1 WBC representative suite|Realman representative suite|planning_time_s|whole_case_wall_time_s" .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md</automated>
    <automated>rg -n "no regression|blocking regression|accepted regression|before|after|delta" .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md</automated>
  </verify>
  <acceptance_criteria>
    - `71-DECISION.md` contains either `Decision: go-native-candidate` or `Decision: keep-native-blocked`
    - `71-DECISION.md` contains `## Compat Sim Grasp Success`
    - `71-DECISION.md` contains `## Compat Trial Budget`
    - `71-DECISION.md` contains `## Native Blocker Status`
    - `71-DECISION.md` contains `## Regression Decision`
    - `71-DECISION.md` contains `## Accepted Regressions`
    - docs and planning state contain `Phase 71`
    - docs and planning state contain `sonic_path=native`
    - `71-VERIFICATION.md` contains `Performance Regression Analysis`
  </acceptance_criteria>
  <done>The repo has a current-state native decision backed by evidence and dual-robot regression analysis.</done>
</task>
</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| evidence artifacts -> final decision | missing or blocked artifacts can be overclaimed as parity |
| G1 native decision -> Realman collateral guard | shared-path changes can regress Realman if only G1 is tested |
| performance metrics -> accepted regression | timing regressions can be hidden behind summary verdicts |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-71-03-1 | Spoofing | native same-case evidence | mitigate | require separate native artifact or explicit `not_run_guard_blocked` status |
| T-71-03-2 | Tampering | representative suite evidence | mitigate | require deterministic Phase 71 roots and `total_cases=8` for both robots |
| T-71-03-3 | Repudiation | performance closeout | mitigate | record exact case IDs, before/after values, deltas, and acceptance boundaries |
| T-71-03-4 | Denial of Service | compat trial loop | mitigate | cap same-case trials at 10 and representative reruns at 3, with every attempt recorded in `71-COMPAT-TRIALS.md` |
| T-71-03-5 | Information Integrity | current-state docs | mitigate | update docs and planning state from `71-DECISION.md`, not from prose-only assumptions |
</threat_model>

<verification>
1. `UV_PROJECT_ENVIRONMENT=.venv uv run python scripts/automation/g1_sonic_e2e.py compat --case-id X36_Y28_Z13 --output-root tmp/phase71_e2e/compat`
2. `UV_PROJECT_ENVIRONMENT=.venv uv run python scripts/automation/g1_sonic_e2e.py official --case-id X36_Y28_Z13 --output-root tmp/phase71_e2e/official --auto-startup --allow-fail-verdict`
3. `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase71_harness/g1_wbc --control-backend wbc --no-timestamp-output`
4. `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.visual_harness.g1_runner --suite representative --output-root tmp/phase71_harness/g1_sonic_compat --control-backend sonic --sonic-path compat --no-timestamp-output`
5. `xvfb-run -a env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 UV_PROJECT_ENVIRONMENT=.venv uv run --extra dev --extra visual python -m tests.realman.visual_harness_runner --suite representative --output-root tmp/phase71_harness/realman --no-timestamp-output`
6. `UV_PROJECT_ENVIRONMENT=.venv uv run python -c "import json, pathlib; paths=['tmp/phase71_harness/g1_wbc/suite_report_representative.json','tmp/phase71_harness/g1_sonic_compat/suite_report_representative.json','tmp/phase71_harness/realman/suite_report_representative.json']; reports=[json.loads(pathlib.Path(p).read_text()) for p in paths]; assert all(r['total_cases']==8 for r in reports); compat=reports[1]; assert compat['execution_error_count']==0; assert compat['pass_count']>=7; assert any(row['case_id']=='X36_Y28_Z13' and row['status']=='pass' for row in compat['results'])"`
7. `rg -n "Trial Ledger|trial_01|planning_time_s|whole_case_wall_time_s" .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-COMPAT-TRIALS.md`
8. `rg -n "Decision: (go-native-candidate|keep-native-blocked)|Compat Sim Grasp Success|Compat Trial Budget|Performance Regression Analysis|G1 WBC representative suite|Realman representative suite|planning_time_s|whole_case_wall_time_s" .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-DECISION.md .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md`
9. `rg -n "no regression|blocking regression|accepted regression|before|after|delta" .planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md`
</verification>

<success_criteria>
- The final Phase 71 decision is explicit and backed by machine-readable evidence.
- `sonic_compat` actually completes at least one named simulated G1 grasp at `HEAD`.
- If the first `sonic_compat` attempt fails, the executor uses the bounded compute/time budget before declaring it blocked: up to 10 focused same-case trials and up to 3 representative compat reruns, all logged in `71-COMPAT-TRIALS.md`.
- Both robot 8-case representative suites are used as no-regression gates.
- Performance metrics are compared numerically, and any accepted regression has detailed analysis.
- Docs, ROADMAP, and STATE reflect the final decision without collapsing compat, official, and native claims.
</success_criteria>

<output>
After completion, create:
- `.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-03-SUMMARY.md`
- `.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-VERIFICATION.md`
- `.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-DECISION.md`
- `.planning/phases/71-deliver-actual-sonic-compat-integration-for-the-live-g1-gras/71-COMPAT-TRIALS.md`
</output>
