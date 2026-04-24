# /autoplan Restore Point
Captured: 2026-04-23T13:00:51Z | Branch: dongxu-dev-0423 | Commit: 7870b66

## Re-run Instructions
1. Copy the "Original Plan Bundle State" sections below back to the corresponding phase plan files
2. Invoke /autoplan on .planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/

## Original Plan Bundle State

### .planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-01-contract-fixtures-and-suite-scaffold-PLAN.md

```md
---
phase: 4
plan: 01
slug: contract-fixtures-and-suite-scaffold
type: execute
wave: 1
depends_on: []
files_modified:
  - roboclaws/regression.py
  - scripts/capture_refactor_regression.py
  - tests/fixtures/replay_summary_reference.json
  - tests/test_refactor_regression_contracts.py
  - tests/test_capture_refactor_regression.py
autonomous: false
requirements_addressed: [A-08]

must_haves:
  truths:
    - "Phase 4 has one shared suite/row vocabulary instead of ad-hoc per-script result shapes."
    - "Critical exact contracts are frozen with tiny fixtures/tests: replay summary required keys, prompt image order/labels, and the already-frozen OpenClaw trace/snapshot rules stay explicit."
    - "The capture harness writes append-only rows and is monkeypatchable in tests via a suite registry."
    - "Plan 01 scaffolds how later plans call existing runners; it does not reimplement the example loops."
  artifacts:
    - path: "roboclaws/regression.py"
      provides: "Shared suite registry, stable pairing-key builder, and row helpers for refactor regression harnesses"
      contains: "class RegressionSuite"
    - path: "scripts/capture_refactor_regression.py"
      provides: "Thin capture CLI shell with registry-backed suite selection and append-only JSONL writing"
      contains: "--suite"
    - path: "tests/fixtures/replay_summary_reference.json"
      provides: "Frozen replay-summary required-key reference"
      contains: "\"summary\""
  key_links:
    - from: "scripts/capture_refactor_regression.py"
      to: "roboclaws/regression.py"
      via: "registry-backed suite lookup and row normalization"
      pattern: "RegressionSuite"
    - from: "tests/test_refactor_regression_contracts.py"
      to: "tests/fixtures/replay_summary_reference.json"
      via: "required-key contract"
      pattern: "replay_summary_reference"
---

<objective>
Create the exact-contract and suite-scaffolding layer that the rest of Phase 4
builds on. After this plan, the phase has a frozen list of hard contracts plus
a small, testable capture-harness shell ready for real suites.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-CONTEXT.md
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-RESEARCH.md
@PLAN.md
@roboclaws/core/views.py
@roboclaws/core/replay.py
@tests/fixtures/trace_schema_reference.json
@tests/test_openclaw_mcp_server.py
@tests/test_openclaw_demo.py
@tests/test_openclaw_nav_autonomous.py
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Add tiny contract fixtures and dedicated refactor-safety tests</name>
  <read_first>
    - roboclaws/core/views.py
    - roboclaws/core/replay.py
    - tests/fixtures/trace_schema_reference.json
    - tests/test_openclaw_mcp_server.py
    - tests/test_openclaw_demo.py
    - tests/test_openclaw_nav_autonomous.py
  </read_first>
  <behavior>
    - Freeze only stable contracts; do not snapshot volatile fields such as timestamps, wallclock totals, or real-model reasoning text.
    - Reuse existing targeted tests where they already pin a contract. Add dedicated Phase-4 tests only where an explicit fixture-backed contract is missing.
    - Keep the fixture tiny and human-reviewable.
  </behavior>
  <action>
    Add a small contract layer for the surfaces this phase exists to protect.

    Required work:

    1. Create `tests/fixtures/replay_summary_reference.json` capturing the
       required `replay.json` top-level `metadata` and `summary` keys that
       refactors must preserve.

    2. Add `tests/test_refactor_regression_contracts.py` covering:
       - `image_labels_for_variant("map-v2+chase")` remains
         `("fpv", "map_v2", "chase")`
       - `build_prompt_images()` preserves FPV / map / chase ordering
       - `ReplayRecorder.save()` emits a `replay.json` whose required
         `metadata` and `summary` keys are a superset of the new fixture
       - the existing `tests/fixtures/trace_schema_reference.json` is still
         treated as the source of truth for OpenClaw additive-vs-exact schema
         contracts

    3. Do NOT clone existing OpenClaw demo/autonomous tests into the new file.
       Instead, keep the new test focused on the cross-cutting contracts that
       Phase 4 needs to name explicitly.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_refactor_regression_contracts.py tests/test_openclaw_mcp_server.py -q</automated>
  </verify>
  <done>Phase 4 has an explicit, fixture-backed contract wall for replay summaries and prompt-image ordering.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Create the shared regression module and thin capture CLI scaffold</name>
  <read_first>
    - examples/view_experiment.py
    - scripts/analyze_view_experiment.py
    - .planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-RESEARCH.md
  </read_first>
  <behavior>
    - The shared module is small: registry, stable coordinate helpers, JSONL append helper, and row-normalization helpers only.
    - The CLI is a shell around the registry, not a giant new execution framework.
    - Later plans must be able to monkeypatch the registry in tests without invoking real AI2-THOR or live providers.
  </behavior>
  <action>
    Create the base Phase-4 harness surfaces:

    1. Add `roboclaws/regression.py` with:
       - a small `RegressionSuite` dataclass
       - a registry keyed by suite name
       - helpers to build stable pairing coordinates
       - append-only JSONL writing helpers
       - a small row-normalization helper that all later suites reuse

    2. Add `scripts/capture_refactor_regression.py` with the CLI shell for:
       - `--suite`
       - `--output-dir`
       - `--label baseline|candidate`
       - `--scenes`
       - `--seeds`
       - `--agents`
       - `--steps`
       - `--model`
       - `--allow-local`

       At plan-01 scope the CLI only needs registry-backed orchestration,
       append-only row writing, and clean error messages for unknown or
       local-only suites. Real suites land in later plans.

    3. Add `tests/test_capture_refactor_regression.py` using a fake suite
       registered entirely inside the test so the scaffold proves:
       - registry lookup works
       - rows append to `results.jsonl`
       - output dirs are created per suite / coordinate
       - local-only suites refuse to run without the explicit override
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_capture_refactor_regression.py -q</automated>
  </verify>
  <done>The Phase-4 scaffold exists and is testable without real runners.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Frozen contract fixtures ↔ future refactors | If the wrong fields are frozen, the harness either misses regressions or blocks harmless changes. |
| Suite scaffold ↔ later capture plans | The base registry and row helpers define the shape every later plan will extend. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-04-01 | Tampering | Contract fixtures | mitigate | Freeze only required keys and stable label ordering; keep volatile fields out of fixtures. |
| T-04-02 | Reliability | Suite scaffold | mitigate | Keep the base registry tiny and prove it with fake-suite tests before adding any real suites. |

No `high`-severity threats; plan proceeds.
</threat_model>

<verification>
- `pytest tests/test_refactor_regression_contracts.py tests/test_capture_refactor_regression.py -q` exits 0.
- The capture scaffold is thin, registry-backed, and append-only.
- Phase 4's exact contracts are explicit and reviewable.
</verification>

<success_criteria>
- The phase has a named contract wall instead of relying on scattered implicit knowledge.
- Later plans can add suites without inventing a new row schema each time.
- No runner logic has been duplicated yet.
</success_criteria>

```

### .planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-02-direct-vlm-and-game-capture-suites-PLAN.md

```md
---
phase: 4
plan: 02
slug: direct-vlm-and-game-capture-suites
type: execute
wave: 2
depends_on: ["04-01"]
files_modified:
  - roboclaws/regression.py
  - scripts/capture_refactor_regression.py
  - tests/test_capture_refactor_regression.py
autonomous: false
requirements_addressed: [A-08]

must_haves:
  truths:
    - "Direct-VLM suites reuse the existing `run_exploration`, `run_territory_game`, and `run_coverage_game` entrypoints instead of forking their control loops."
    - "Each captured row exposes stable pairing coordinates plus structured metrics from the existing result dicts and `replay.json` summaries."
    - "The capture harness preserves append-only `results.jsonl` semantics and per-run replay directories."
    - "Cloud-safe tests cover the suite registration and row extraction paths without live VLM calls or Unity."
  artifacts:
    - path: "roboclaws/regression.py"
      provides: "Registered direct-VLM / territory / coverage suites with normalized row extraction"
      contains: "explore-vlm"
    - path: "scripts/capture_refactor_regression.py"
      provides: "Capture CLI capable of running the direct-VLM suite matrix"
      contains: "territory-vlm"
    - path: "tests/test_capture_refactor_regression.py"
      provides: "Smoke coverage for direct suite registration and row output"
      contains: "coverage-vlm"
  key_links:
    - from: "roboclaws/regression.py"
      to: "scripts/capture_refactor_regression.py"
      via: "suite registry consumed by the CLI matrix loop"
      pattern: "explore-vlm"
---

<objective>
Wire the direct-VLM and game-path suites into the Phase-4 capture harness.
After this plan, a cloud-safe session can capture baseline/candidate rows for
the repo's direct exploration, territory, and coverage paths without inventing
new runner logic.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-CONTEXT.md
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-RESEARCH.md
@roboclaws/regression.py
@scripts/capture_refactor_regression.py
@examples/single_agent_explore.py
@examples/territory_game.py
@examples/coverage_game.py
@examples/view_experiment.py
@tests/test_view_experiment.py
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Register the direct-VLM suites and normalize their row shapes</name>
  <read_first>
    - examples/single_agent_explore.py
    - examples/territory_game.py
    - examples/coverage_game.py
    - examples/view_experiment.py
    - roboclaws/regression.py
  </read_first>
  <behavior>
    - The harness calls the existing public runners directly.
    - Row extraction uses the structured return dict plus `replay.json` summary when helpful; it does not parse stdout.
    - Each suite emits stable coordinates suitable for baseline-vs-candidate pairing.
  </behavior>
  <action>
    Extend `roboclaws/regression.py` with these capture suites:

    1. `explore-vlm`
       - runner: `run_exploration(...)`
       - metrics: `cells_visited`, `termination_reason`, `vlm_cost_usd`,
         `provider_status`, replay `total_steps`

    2. `territory-vlm`
       - runner: `run_territory_game(..., backend="vlm")`
       - metrics: `cells_claimed_total`, `blocking_events`,
         `termination_reason`, `vlm_cost_usd`, `provider_status`

    3. `coverage-vlm`
       - runner: `run_coverage_game(..., backend="vlm")`
       - metrics: `coverage_fraction`, `cells_covered`, `work_balance`,
         `termination_reason`, `vlm_cost_usd`, `provider_status`

    Required common coordinates:
    - `suite`
    - `backend`
    - `scene`
    - `seed`
    - `game`
    - `model`
    - `agents`
    - `variant` when present

    Seed the suites the same way `view_experiment.py` does: set both
    `random.seed(seed)` and `np.random.seed(seed)` before calling the runner.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_capture_refactor_regression.py -q</automated>
  </verify>
  <done>The direct-VLM suites exist and emit structured rows with stable pairing coordinates.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Extend the capture CLI for direct-VLM matrix runs</name>
  <read_first>
    - scripts/capture_refactor_regression.py
    - roboclaws/regression.py
    - examples/view_experiment.py
  </read_first>
  <behavior>
    - The CLI remains thin: suite selection, coordinate iteration, append-only row writing, and output-dir layout only.
    - Per-run artifacts stay under the suite-specific replay dir produced by the existing runner.
    - The CLI supports partial suite lists so operators can refresh one baseline slice without rerunning everything.
  </behavior>
  <action>
    Update `scripts/capture_refactor_regression.py` to run the direct-VLM suite
    matrix.

    Required behavior:

    1. Accept `--suite explore-vlm,territory-vlm,coverage-vlm` and iterate
       `suite × scene × seed`.

    2. Write output under:

       ```text
       output/refactor-regression/<label>/<suite>/<scene>-seed<N>/
       ```

       with `results.jsonl` at the `<label>/` root.

    3. Keep JSONL append-only semantics. If a run fails, write a row with:
       - `status=error`
       - `error_kind`
       - `error`
       - the same stable coordinates

    4. Add smoke tests proving:
       - multiple direct suites can run in one command
       - rows append in a deterministic order
       - failed runs log error rows and the loop continues

    Do NOT add a separate analyzer here. That belongs to Plan 04.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_capture_refactor_regression.py tests/test_refactor_regression_contracts.py -q</automated>
  </verify>
  <done>The capture CLI can run the direct-VLM suite matrix and produce replay dirs plus append-only rows.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Capture CLI ↔ existing example runners | The harness must call the shipped entrypoints exactly as they are, not a harness-only fork. |
| Structured result rows ↔ later analyzer | The row schema chosen here becomes the pairing surface for Plan 04. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-04-03 | Tampering | Direct-VLM capture suites | mitigate | Reuse the existing public runners and extract only structured outputs plus replay summaries. |
| T-04-04 | Repudiation | Append-only capture history | mitigate | Failed runs still emit error rows with full stable coordinates so baseline/candidate comparisons do not silently skip regressions. |

No `high`-severity threats; plan proceeds.
</threat_model>

<verification>
- `pytest tests/test_capture_refactor_regression.py tests/test_refactor_regression_contracts.py -q` exits 0.
- Direct-VLM capture runs reuse the current examples and produce append-only rows.
- Row shapes are stable enough for baseline-vs-candidate pairing.
</verification>

<success_criteria>
- A cloud-safe session can capture baseline or candidate runs for the direct-VLM stacks.
- The harness exposes the metrics Phase 4 actually cares about instead of raw text logs.
- No duplicate runner implementation exists.
</success_criteria>

```

### .planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-03-openclaw-capture-suites-PLAN.md

```md
---
phase: 4
plan: 03
slug: openclaw-capture-suites
type: execute
wave: 3
depends_on: ["04-02"]
files_modified:
  - roboclaws/regression.py
  - scripts/capture_refactor_regression.py
  - tests/test_capture_refactor_regression.py
autonomous: false
requirements_addressed: [A-08]

must_haves:
  truths:
    - "Push-model OpenClaw suites reuse `openclaw_demo.py` and the existing territory/coverage example runners with `backend=\"openclaw\"`."
    - "The autonomous OpenClaw suite extracts structured metrics from `run_result.json` and `summary.json`; it does not diff raw transcript prose."
    - "Every OpenClaw suite is explicitly marked local-dev only and refuses to run without the operator override."
    - "Captured rows never log secrets and only persist artifact paths plus structured metrics."
  artifacts:
    - path: "roboclaws/regression.py"
      provides: "OpenClaw push-model and autonomous suite definitions with structured metric extraction"
      contains: "openclaw-autonomous"
    - path: "scripts/capture_refactor_regression.py"
      provides: "Capture CLI capable of local-only OpenClaw suite execution"
      contains: "--allow-local"
    - path: "tests/test_capture_refactor_regression.py"
      provides: "Synthetic suite tests for OpenClaw row extraction and guardrails"
      contains: "openclaw-demo"
  key_links:
    - from: "roboclaws/regression.py"
      to: "scripts/render_autonomous_replay.py"
      via: "summary metrics consumed from `summary.json`"
      pattern: "transcript_source"
---

<objective>
Extend the Phase-4 capture harness to the shipped OpenClaw paths. After this
plan, the same baseline/candidate workflow can capture push-model Gateway runs
and the autonomous MCP path while preserving the repo's cloud/local boundary.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-CONTEXT.md
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-RESEARCH.md
@roboclaws/regression.py
@scripts/capture_refactor_regression.py
@examples/openclaw_demo.py
@examples/territory_game.py
@examples/coverage_game.py
@examples/openclaw_nav_autonomous.py
@scripts/render_autonomous_replay.py
@tests/test_openclaw_demo.py
@tests/test_openclaw_nav_autonomous.py
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Add the push-model OpenClaw capture suites</name>
  <read_first>
    - examples/openclaw_demo.py
    - examples/territory_game.py
    - examples/coverage_game.py
    - roboclaws/regression.py
  </read_first>
  <behavior>
    - Reuse the current runners exactly as the user would run them.
    - Mark these suites as local-dev only.
    - Extract only structured result data and replay summary fields.
  </behavior>
  <action>
    Extend `roboclaws/regression.py` with push-model OpenClaw suites:

    1. `openclaw-demo`
       - runner: `run_openclaw_demo(...)`
       - metrics: `visited_cells`, `steps_executed`, `termination_reason`,
         `provider_status`

    2. `territory-openclaw`
       - runner: `run_territory_game(..., backend="openclaw")`
       - metrics: `cells_claimed_total`, `blocking_events`,
         `termination_reason`, `provider_status`

    3. `coverage-openclaw`
       - runner: `run_coverage_game(..., backend="openclaw")`
       - metrics: `coverage_fraction`, `cells_covered`, `work_balance`,
         `termination_reason`, `provider_status`

    All three suites must:
    - set `backend="openclaw"` explicitly
    - carry the same stable coordinates as the direct-VLM suites
    - be flagged `local_dev_only=True`
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_capture_refactor_regression.py tests/test_openclaw_demo.py -q</automated>
  </verify>
  <done>Push-model OpenClaw suites exist and are guarded as local-only.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Add the autonomous OpenClaw suite and local-only CLI guardrails</name>
  <read_first>
    - examples/openclaw_nav_autonomous.py
    - scripts/render_autonomous_replay.py
    - tests/test_openclaw_nav_autonomous.py
    - tests/test_render_autonomous_replay.py
    - scripts/capture_refactor_regression.py
  </read_first>
  <behavior>
    - The autonomous suite captures structured data only: `run_result.json`,
      `summary.json`, and artifact paths.
    - The CLI refuses local-only suites unless the operator explicitly opts in.
    - Tests stay synthetic: no real Gateway, Docker, or AI2-THOR.
  </behavior>
  <action>
    Finish the OpenClaw capture surface:

    1. Add `openclaw-autonomous` to `roboclaws/regression.py`:
       - runner: `run_autonomous_navigation(...)`
       - row extraction sources:
         - `run_result.json`
         - `summary.json`
       - required metrics:
         - `terminated_by`
         - `transcript_source`
         - `tool_calls_by_type`
         - `frames_unseen_by_agent`
         - `decision_modes`
         - `wallclock_seconds`
         - `view_variant`

    2. Update `scripts/capture_refactor_regression.py` so any suite marked
       `local_dev_only=True` hard-fails without `--allow-local`, with a message
       that points back to the repo's cloud/local split.

    3. Add synthetic tests proving:
       - autonomous row extraction works from canned `run_result.json` and
         `summary.json`
       - `--allow-local` is required for OpenClaw suites
       - failed local-only runs still emit error rows with stable coordinates

    Do NOT add threshold logic here. That belongs to Plan 04.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_capture_refactor_regression.py tests/test_openclaw_nav_autonomous.py tests/test_render_autonomous_replay.py -q</automated>
  </verify>
  <done>The harness can capture structured OpenClaw push-model and autonomous metrics without weakening the local-only boundary.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Local-only OpenClaw suites ↔ cloud sessions | The capture harness must not imply that cloud can validate real Gateway behavior. |
| OpenClaw artifact extraction ↔ later analyzer | The analyzer depends on truthful structured metrics, not hidden secrets or prose parsing. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-04-05 | Information Disclosure | OpenClaw capture rows | mitigate | Persist only structured metrics and artifact paths; never write bearer tokens or raw Authorization headers. |
| T-04-06 | Spoofing | Local-only suite execution | mitigate | Require an explicit `--allow-local` override for every local-only OpenClaw suite. |
| T-04-07 | Tampering | Autonomous metric extraction | mitigate | Derive metrics from `run_result.json` and `summary.json`, not from fragile transcript text parsing. |

No `high`-severity threats; plan proceeds.
</threat_model>

<verification>
- `pytest tests/test_capture_refactor_regression.py tests/test_openclaw_demo.py tests/test_openclaw_nav_autonomous.py tests/test_render_autonomous_replay.py -q` exits 0.
- OpenClaw suites are explicit, local-only, and structured.
- The capture harness still does not own any runner logic itself.
</verification>

<success_criteria>
- The Phase-4 capture workflow now spans the shipped OpenClaw surfaces.
- The cloud/local boundary is enforced by code, not just by documentation.
- The analyzer will receive stable OpenClaw metrics instead of raw prose blobs.
</success_criteria>

```

### .planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-04-compare-analyzer-and-local-baseline-workflow-PLAN.md

```md
---
phase: 4
plan: 04
slug: compare-analyzer-and-local-baseline-workflow
type: execute
wave: 4
depends_on: ["04-03"]
files_modified:
  - scripts/analyze_refactor_regression.py
  - tests/test_analyze_refactor_regression.py
  - docs/refactor-regression.md
  - .planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-LOCAL-PROBE-RESULTS.md
autonomous: false
requirements_addressed: [A-08]

must_haves:
  truths:
    - "Baseline-vs-candidate comparison pairs runs on stable coordinates and exits non-zero on threshold breaches."
    - "Threshold policy is suite-specific, reviewable in code, and separate from capture."
    - "`docs/refactor-regression.md` tells operators how to capture baselines, run comparisons, and which suites are local-only."
    - "`04-LOCAL-PROBE-RESULTS.md` records the first real baseline refresh commands, artifact paths, and any threshold adjustments justified by live evidence."
    - "No large baseline artifacts or secrets are committed to the repo."
  artifacts:
    - path: "scripts/analyze_refactor_regression.py"
      provides: "Machine-readable and markdown baseline-vs-candidate analyzer"
      contains: "ThresholdPolicy"
    - path: "docs/refactor-regression.md"
      provides: "Operator workflow for baseline capture, candidate capture, and comparison"
      contains: "local-only suites"
    - path: ".planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-LOCAL-PROBE-RESULTS.md"
      provides: "Dated local evidence for the first real baseline refresh"
      contains: "Artifact paths"
  key_links:
    - from: "docs/refactor-regression.md"
      to: ".planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-LOCAL-PROBE-RESULTS.md"
      via: "operator evidence link"
      pattern: "04-LOCAL-PROBE-RESULTS"
---

<objective>
Finish the Phase-4 workflow by adding the baseline-vs-candidate analyzer,
operator documentation, and the first real local baseline refresh. After this
plan, maintainers can use the harnesses to answer "did the refactor preserve
behavior within tolerance?" on the real stacks that matter.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-CONTEXT.md
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-RESEARCH.md
@.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-VALIDATION.md
@roboclaws/regression.py
@scripts/capture_refactor_regression.py
@examples/view_experiment.py
@scripts/analyze_view_experiment.py
@scripts/render_autonomous_replay.py
@docs/openclaw-local.md
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Implement the baseline-vs-candidate analyzer with suite-specific threshold policies</name>
  <read_first>
    - scripts/analyze_view_experiment.py
    - roboclaws/regression.py
    - .planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-RESEARCH.md
  </read_first>
  <behavior>
    - The analyzer is separate from capture.
    - Pairing is driven only by stable coordinates.
    - Threshold policy is explicit per suite; no single global threshold.
    - The analyzer emits both human-readable markdown and machine-readable JSON.
  </behavior>
  <action>
    Create `scripts/analyze_refactor_regression.py` and
    `tests/test_analyze_refactor_regression.py`.

    Required analyzer behavior:

    1. Inputs:
       - `--baseline <results.jsonl>`
       - `--candidate <results.jsonl>`
       - optional `--output-dir`

    2. Pair rows on:
       - `suite`
       - `backend`
       - `scene`
       - `seed`
       - `game`
       - `model`
       - `agents`
       - `variant`

    3. Enforce suite policies in code, with at least these first-pass rules:
       - `explore-vlm` / `openclaw-demo`: visited cells no worse than `-1`,
         cost `<= +25%`, wallclock `<= +50%`
       - `territory-*`: total claimed no worse than `-2`, blocking no worse
         than `+2`, no new provider-failure termination
       - `coverage-*`: coverage fraction no worse than `-0.05`, work balance
         no worse than `-0.10`, steps no worse than `+20%`
       - `openclaw-autonomous`: `transcript_source` exact,
         `tool_calls_by_type` within `±2`, `frames_unseen_by_agent` no worse
         than `+2`

    4. Outputs:
       - `summary.md`
       - `summary.json`
       - non-zero exit code if any threshold is breached

    5. Tests must cover:
       - successful pairing
       - missing baseline or candidate rows
       - threshold breaches
       - suite-specific exact checks such as `transcript_source`
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_analyze_refactor_regression.py -q</automated>
  </verify>
  <done>The repo can compare baseline vs candidate capture sets and fail fast on real regressions.</done>
</task>

<task type="auto">
  <name>Task 2: Write the operator workflow in `docs/refactor-regression.md`</name>
  <read_first>
    - scripts/capture_refactor_regression.py
    - scripts/analyze_refactor_regression.py
    - AGENTS.md § 1 and § 7
    - docs/openclaw-local.md
  </read_first>
  <behavior>
    - The doc is concise and operational.
    - It distinguishes cloud-safe suites from local-only suites.
    - It explains baseline refresh, candidate capture, analyzer use, and how to investigate a failure.
  </behavior>
  <action>
    Add `docs/refactor-regression.md` with these sections:

    1. `## What this harness protects`
       - direct VLM
       - territory / coverage
       - OpenClaw push-model
       - OpenClaw autonomous

    2. `## Capture a baseline`
       - example `scripts/capture_refactor_regression.py` commands

    3. `## Capture a candidate`
       - matching commands with a different label/output root

    4. `## Compare them`
       - `scripts/analyze_refactor_regression.py` command
       - where `summary.md` / `summary.json` land

    5. `## Local-only suites`
       - explain `--allow-local`
       - point back to `AGENTS.md` cloud/local split and `docs/openclaw-local.md`

    6. `## Evidence`
       - link to `04-LOCAL-PROBE-RESULTS.md`

    Keep the doc operator-facing. Do not turn it into a changelog.
  </action>
  <verify>
    <automated>bash -c 'f=/home/mi/ws/gogo/roboclaws/docs/refactor-regression.md; rg -n "What this harness protects|Capture a baseline|Capture a candidate|Compare them|Local-only suites|04-LOCAL-PROBE-RESULTS" "$f"'</automated>
  </verify>
  <done>The repo has one operator doc for baseline capture, candidate capture, and comparison.</done>
</task>

<task type="checkpoint:human-verify" gate="blocking">
  <name>Task 3: Pre-flight — confirm a real local baseline refresh is possible</name>
  <read_first>
    - AGENTS.md § 1 and § 7
    - docs/openclaw-local.md
    - docs/refactor-regression.md
  </read_first>
  <behavior>
    - This task does not proceed in a cloud-only session.
    - The operator confirms Docker, AI2-THOR, and at least one real provider key are available.
    - No stale `openclaw-gateway` container is blocking the expected ports unless the operator intends to reuse it.
  </behavior>
  <action>
    Run and paste the outputs of:

    1. `docker --version`
    2. `python -c "import ai2thor; print(ai2thor.__version__)"`
    3. `[[ -n "$KIMI_API_KEY" || -n "$OPENAI_API_KEY" || -n "$ANTHROPIC_API_KEY" ]] && echo provider-set || echo provider-missing`
    4. `docker ps -a --format '{{.Names}}' | grep -x openclaw-gateway || echo absent`
    5. `env -i PATH=".venv/bin:/usr/bin:/bin" HOME=$HOME .venv/bin/pytest tests/test_capture_refactor_regression.py tests/test_analyze_refactor_regression.py -q`

    If this is a cloud session, stop here and defer the local refresh.
  </action>
  <verify>
    <automated>MISSING — checkpoint task; operator confirms local readiness</automated>
  </verify>
  <done>Operator confirms a real local baseline-refresh session is available.</done>
</task>

<task type="auto">
  <name>Task 4: Refresh the first real baselines and write `04-LOCAL-PROBE-RESULTS.md`</name>
  <read_first>
    - docs/refactor-regression.md
    - scripts/capture_refactor_regression.py
    - scripts/analyze_refactor_regression.py
    - examples/openclaw_demo.py
    - examples/openclaw_nav_autonomous.py
  </read_first>
  <behavior>
    - Capture at least one real direct-VLM run, one push-model OpenClaw run, and one autonomous OpenClaw run.
    - Use the analyzer on at least one baseline/candidate pair and record the result honestly.
    - Any threshold change made after live evidence must be explained in the local results file.
    - No raw secrets are pasted into the repo.
  </behavior>
  <action>
    Run the first local baseline refresh and record it under
    `.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-LOCAL-PROBE-RESULTS.md`.

    Minimum evidence set:

    1. One real direct-VLM suite capture, for example `territory-vlm`
    2. One real push-model OpenClaw suite capture, for example `openclaw-demo`
    3. One real `openclaw-autonomous` capture
    4. One analyzer run comparing a baseline vs candidate pair

    The write-up must include:
    - date and host context
    - exact commands run (with env var placeholders, not secrets)
    - artifact paths
    - the suites refreshed
    - analyzer outcome (`pass` / `fail`) and the specific threshold result
    - any threshold adjustment justified by live evidence

    If live evidence forces a threshold adjustment, update
    `scripts/analyze_refactor_regression.py` before marking the task done and
    rerun the focused pytest slice:

    ```bash
    env -i PATH=".venv/bin:/usr/bin:/bin" HOME=$HOME .venv/bin/pytest \
      tests/test_capture_refactor_regression.py \
      tests/test_analyze_refactor_regression.py -q
    ```
  </action>
  <verify>
    <automated>test -f /home/mi/ws/gogo/roboclaws/.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-LOCAL-PROBE-RESULTS.md && rg -n "Artifact paths|Analyzer outcome|openclaw-autonomous|openclaw-demo|territory-vlm" /home/mi/ws/gogo/roboclaws/.planning/phases/04-refactor-regression-harnesses-for-vlm-territory-coverage-and/04-LOCAL-PROBE-RESULTS.md</automated>
  </verify>
  <done>The first real baseline refresh is documented with commands, artifact paths, and analyzer outcomes.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Baseline/candidate analyzer ↔ operator decisions | The analyzer will decide whether a refactor is acceptable; bad pairing or bad thresholds make it untrustworthy. |
| Local probe evidence ↔ repo docs | The docs and local results file must describe only what was actually validated. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-04-08 | Reliability | Threshold policy | mitigate | Keep the threshold table explicit, per suite, synthetic-test covered, and adjustable only with recorded live evidence. |
| T-04-09 | Tampering | Row pairing | mitigate | Pair only on stable coordinates and fail loudly on missing baseline/candidate rows. |
| T-04-10 | Information Disclosure | Local evidence / docs | mitigate | Use env-var placeholders in docs and local-results notes; never paste secrets or large baseline artifacts into the repo. |

No `high`-severity threats; plan proceeds.
</threat_model>

<verification>
- `pytest tests/test_analyze_refactor_regression.py -q` exits 0.
- `docs/refactor-regression.md` exists and documents the operator workflow.
- `04-LOCAL-PROBE-RESULTS.md` records the first real baseline refresh and any threshold adjustments.
</verification>

<success_criteria>
- Maintainers can capture baselines, capture candidates, and compare them with one documented workflow.
- Phase 4 ends with at least one real local proof that the harness is usable on the actual stacks.
- Threshold policy is evidence-backed rather than hand-wavy.
</success_criteria>

```

