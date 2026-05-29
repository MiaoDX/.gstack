# /autoplan Restore Point
Captured: 2026-05-28T11:31:47+08:00 | Branch: dongxu/dev-0520 | Commit: f9de3524

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
---
plan_scope: g1-curobo-upstream-and-waist-latency
status: DRAFT
created: 2026-05-28
owners:
  - repo-owned manipulation layer
related:
  - docs/plans/refactor-g1-curobo-waist-candidate-architecture.md
  - docs/plans/decoupled-wbc-direct-pregrasp-experiments.md
---

# G1 cuRobo Upstream and Waist Latency Plan

## Goal

Reduce G1 grasp planner latency for both `decoupled_wbc` and `sonic_native`
without weakening the current accepted representative envelope:

- `7 pass / 1 known X50_Y10_Z0 goalset_infeasible / 0 execution_error`
- public grasp stages remain `pregrasp -> approach -> close -> lift`
- production deploy uses promoted execution profiles only
- experimental switches stay behind probe profiles and promotion gates

The immediate latency target is the manual E2E planner path where waist
candidate probing and verify/finalize can dominate the run at roughly 5-10s.
Harness-local planning can be faster, so promotion must compare like-for-like
manual or representative harness timing instead of relying on one cached pass.

## Non-Goals

- Do not add aliases or backward-compatibility names for cleaned probe fields.
- Do not expose arbitrary low-level probe switches in deploy entrypoints.
- Do not mutate `venders/curobo_v2` as a final delivery path; that tree is
  ignored by the parent repo and needs an explicit pin/fork/patch strategy if
  upstream changes are adopted.
- Do not promote strict selected-only, active-waist full-stage, active-waist
  corridor, active-waist PoseCostMetric, full planner fusion, or planner
  fusion prefix without a new planner-source fix and fresh L4 evidence.
- Do not treat cuRobo `BatchMotionPlanner` as a drop-in replacement for the
  locked-waist candidate loop unless it can express per-candidate waist state
  without weakening collision, branch, or finalization checks.

## Current Truth

- Current production defaults remain:
  - `decoupled_wbc`: `g1_decoupled_retained_preplan_quality_ranked_v1`
  - `sonic_native`: `g1_sonic_native_direct_pregrasp_v1`
- The selected-probe-only diagnostics are rejected:
  - strict decoupled: `3 pass / 5 fail / 0 execution_error`
  - strict Sonic: `2 pass / 6 fail / 0 execution_error`
  - softer selected-first/fallback can preserve `7/1/0`, but still uses
    direct-candidate resweep and has weaker pass-case precision than the
    retained-preplan quality-ranked default.
- The cuRobo v2 release checkout currently tracks `origin/curobov2_release`.
  That branch has no newer commits relative to the local checkout.
- Upstream `origin/main` exists and is newer. Its motion planner and batch
  planner Python logic are almost unchanged for this problem; the important
  possible win is lower-level CUDA kinematics kernel speed. That is worth an
  isolated benchmark, not a direct production switch.
- Current local environment probes show `warp-lang 1.11.0`, `cuda-core 0.7.0`,
  `torch 2.6.0`, and cuRobo imported from `venders/curobo_v2`.

## Boundary Decisions

These are the defaults from the latency discussion. Reopen them only when new
evidence would change what gets built, verified, or promoted.

| Question | Default decision | Why |
| --- | --- | --- |
| May we modify `venders/curobo_v2`? | Not as an untracked final change. Vendor edits are allowed only as an isolated spike or as a tracked fork/pin/patch-file delivery path with tests. | The parent repo ignores `venders/`, so a direct local edit is not reviewable or reproducible. |
| How much precision can be traded for latency? | No precision regression is accepted by default. A faster profile must preserve `7/1/0` and match or beat current close/holding quality unless a promotion record explicitly accepts the tradeoff. | Existing faster-looking paths often preserved pass count while weakening close/holding precision. |
| What is the next promotion gate? | Same-surface A/B timing plus representative 8-case evidence. A new profile must keep execution errors at zero, preserve the known X50 boundary, and report profile lineage clearly. | Planner latency claims are only useful if measured on the same manual or representative harness surface. |
| Is vendor patching parked or forbidden? | Parked, not forbidden. It becomes eligible after repo-owned fail-fast/cache/batch-proxy work is exhausted or a minimal vendor bugfix clearly unlocks true full-stage batch. | The risky part is maintenance and reproducibility, not the idea of changing vendor code itself. |

## cuRobo `main` Revalidation Rule

The rejected probe conclusions are current-stack conclusions, not permanent
theorems. If the upstream `main` spike materially changes planner-source
behavior, rerun the affected probes before relying on their old verdicts.

- `selected-only`: unlikely to change from kinematics speed alone. The observed
  failure is semantic: selected branches that require `retract_then_waist`
  become `seg_a_collision` when fallback/direct-candidate resweep is disabled.
  cuRobo `main` can only change this conclusion if it changes selected branch
  feasibility enough to remove those retract-prefix failures in representative
  evidence.
- `active-waist`: possible but not assumed. `main` kinematics kernels might
  improve speed or numerical behavior, but the active-waist probes failed before
  backend execution with `grasp_ik_failed` / `goalset_infeasible` envelopes.
  Reopen only if the main spike shows planner-source success changes, not just
  faster failed solves.
- `planner fusion` and `planner fusion prefix`: possible but not assumed. The
  inspected `main` motion-planner Python logic is not meaningfully different
  for this problem, so a reversal would need fresh L4 evidence, not commit
  messages alone.
- `BatchMotionPlanner`: `main` does not currently appear to add public per-row
  locked-joint support. Treat true full-stage batch as a separate failed-row
  fallback / vendor-patch question.

## Direction Matrix

| Direction | Why it matters | Plan | Promotion gate | Current call |
| --- | --- | --- | --- | --- |
| cuRobo `origin/main` isolated spike | Main includes updated kinematics kernels advertised as faster on humanoids. It may reduce every IK/TrajOpt call cost. | Use a temporary checkout/worktree. Compile/import main, run vendor motion/batch tests, then G1 focused tests and representative timing. Do not alter the production vendor checkout. | No build/import regression; no G1 envelope loss; meaningful latency improvement on representative/manual timing. If accepted, define a tracked fork/pin/patch delivery path. | Worth testing. Not a production dependency yet. |
| Effective dry-run/fail-fast budgets | Current wrapper exposes knobs like `timeout`, `partial_ik_opt`, `ik_fail_return`, and `parallel_finetune`, but the v2 path primarily honors attempts and seed counts. Silent no-op knobs make latency tuning misleading. | Make the actual v2 budget surface explicit: `max_attempts`, IK seeds, TrajOpt seeds, and any real early-return behavior. Emit metrics that prove the budget used by each candidate phase. | Focused tests prove config maps to real v2 calls; visual/manual timing improves without losing `7/1/0` or precision gates. | Highest-priority repo-owned change. |
| Warm/cache `BatchMotionPlanner` for batch pregrasp | The current multi-env batch wrapper constructs and destroys a planner per call. That can hide the value of batch pregrasp probes behind setup cost. | Cache warmed `BatchMotionPlanner` instances keyed by robot config, collision-world shape, batch size, goalset size, and relevant planner config. Add explicit destroy/reset behavior. | Resource-release tests pass; no stale-world leakage; batch pregrasp timing improves in representative cases. | High-priority, lower risk than vendor patching. |
| Shared Sonic/Decoupled candidate strategy | The two backends should differ at execution only when the backend contract requires it. Waist candidate selection should be profile-driven and shared where evidence supports it. | Keep production defaults profile-driven. Align shared internals around selected preplan plus quality-ranked direct candidates, with probe-only variants for batch proxy, selected-first/fallback, and latency comparisons. | Both backends preserve their accepted envelopes and reporting semantics; production deploy exposes no arbitrary low-level switches. | Good cleanup direction. Do not force selected-only. |
| Frame-transformed batch proxy ranking | Existing batch pregrasp avoids per-row locked joints by transforming targets/world into one reference waist frame. It can rank candidates before serial verify. | Keep as a probe. Combine with bounded top-K serial verify/finalize, then compare timing and close/holding quality against retained-preplan quality ranking. | Must preserve `7/1/0` and match or beat current precision/latency on representative runs. | Useful probe, not current default. |
| True full-stage batch grasp | If vendor batch `plan_grasp` can safely handle failed rows, it could remove more serial verify/finalize work. | First reproduce and isolate failed-row fallback behavior. If a small fix exists, test it in a temporary vendor branch or patch file, not as an untracked vendor edit. | Vendor tests plus G1 full-stage batch visual harness pass; no CUDA device-side asserts; better latency than serial full-stage verify. | P2 after fail-fast/cache work. |
| Per-row locked-joint batch in cuRobo | This would let batch planning represent one locked waist yaw per candidate directly. | Park unless all repo-owned batch/active-waist approaches fail and the latency problem remains severe. If reopened, treat it as a cuRobo fork-level design. | Needs a tracked upstream/fork strategy, vendor tests, G1 harness evidence, and clear maintenance ownership. | Parked. Too deep for next slice. |
| Waist as normal active joint | Simplest conceptual model, but current probes collapse before backend execution. | Do not continue tuning under the waist-selection scope. Reopen only with a new grasp-IK or approach-stage fix. | Fresh planner-source fix plus representative L4 evidence for both backends. | Rejected for current scope. |
| Planner fusion / upstream `plan_grasp` | Could simplify stages if upstream generated the full sequence. Current evidence fails representative cases. | Keep diagnostic profiles demoted. Reopen only under an approach/finalizer geometry scope, not latency-only waist selection. | New fix plus fresh L4 evidence. | Rejected for current scope. |

## Execution Plan

### Slice 1: Upstream Main Spike

1. Create a temporary cuRobo main checkout or worktree from `origin/main`.
2. Build/import with the repo runtime environment.
3. Run minimal vendor checks:
   - motion planner batch tests
   - graph planner regression if dependencies are available
   - a focused kinematics or IK smoke that exercises G1-like humanoid shape
4. Run repo focused checks:
   - `tests/manipulation/test_g1_profiles.py`
   - `tests/visual_harness/test_g1_native_probe_reporting.py`
   - `tests/visual_harness/test_g1_runner.py`
   - `tests/grasp/planning/test_motion_gen_stage_pipeline.py::TestG1SonicNativeDirectPregraspPlanning`
5. Run a small timing comparison for current release vs main:
   - batch pregrasp probe timing
   - direct candidate loop timing
   - representative case planning wall time
6. Decide:
   - discard if no material speedup or if build/API risk is high
   - continue only if speedup is large enough to justify a tracked delivery path

### Slice 2: Real Budget Surface

1. Audit the v2 wrapper and identify which per-call fields are actually used by
   `MotionPlanner` and `BatchMotionPlanner`.
2. Rename or demote misleading no-op knobs in repo-owned code where safe.
3. Add explicit budget fields for dry-run and verify phases:
   - attempts
   - IK seed count
   - TrajOpt seed count
   - graph usage where actually supported
4. Add metrics per planning phase:
   - requested budget
   - effective budget
   - attempts used
   - candidate index/order
   - wall time
5. Verify with focused unit tests and one representative harness run.

### Slice 3: Warm Batch Pregrasp

1. Add a small planner-cache layer inside the repo v2 facade or motion planner
   wrapper.
2. Cache only batch planner instances whose config and world-shape are known to
   be reusable.
3. Add explicit cleanup and resource-release tests.
4. Compare before/after batch pregrasp timing.
5. Keep this behind probe/profile evidence until representative timing and
   precision are clear.

### Slice 4: Backend Alignment Cleanup

1. Keep production deployment profile-driven:
   - backend chooses the production default
   - probe profile chooses evidence variants
2. Move shared candidate-selection logic into backend-neutral helpers where the
   behavior is truly common.
3. Preserve backend-specific execution semantics only after the source plan is
   selected.
4. Remove dead or duplicated branches that only encode old decoupled/Sonic
   naming differences.
5. Run focused profile, runner, and planner tests, then a representative harness
   comparison for both backends if behavior changes.

### Slice 5: True Full-Stage Batch Follow-Up

1. Reproduce the failed-row fallback/CUDA assert in a minimal vendor or repo
   test.
2. Decide whether the fix is:
   - repo-side guard that avoids unsafe rows
   - vendor patch file applied by script
   - fork/pin to upstream main or a repo-owned branch
3. Only continue if the fix can be tracked and tested.
4. Run vendor batch tests plus G1 full-stage batch probe evidence.

## Verification Gates

- L0 static:
  - `git diff --check`
  - import/compile checks for touched repo modules
- L1 focused tests:
  - profile expansion and runner reporting
  - v2 wrapper budget mapping
  - batch planner resource release
- L2 report-contract checks:
  - probe profile identity
  - execution profile identity
  - no production exposure of low-level probe switches
- L4 visual/manual evidence:
  - representative 8-case G1 harness for both `decoupled_wbc` and
    `sonic_native` when behavior changes
  - manual E2E timing for the Windows/interactive path that surfaced the
    planner latency issue

## Stop Conditions

Stop and keep current production defaults if:

- cuRobo main does not produce a meaningful G1 latency gain, or introduces
  build/API risk;
- real budget and warm batch pregrasp do not preserve the accepted envelope;
- any candidate improves speed only by accepting selected-only failures,
  active-waist collapse, or weaker close/holding precision;
- a vendor patch becomes necessary but cannot be tracked, tested, and owned.

Promote a new execution profile only if:

- it preserves the accepted representative envelope;
- it improves latency on the same evidence surface;
- it matches or beats current close/holding quality, or the quality tradeoff is
  explicitly accepted in a promotion record;
- it keeps probe and execution profile lineage clear.
