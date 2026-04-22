# /autoplan Restore Point
Captured: 2026-04-17T17:50:48+08:00 | Branch: main | Commit: aabcad2

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
<!-- /autoplan restore point: /home/mi/.gstack/projects/MiaoDX-roboharness/main-autoplan-restore-20260417-145527.md -->
# Roboharness Unattended Refactor Harness Plan

Status: Draft rewritten on 2026-04-17 from the earlier showcase-split plan.

This document replaces the previous strategy of aggressively moving advanced demos
out of the core repo. The CEO review and follow-up discussion found that repo
cleanliness is not the real bottleneck. Trust is.

## Product Statement

Roboharness is not selling a clean repo.

It is selling **trust compression**:

`2-3 hours unattended Claude/Codex refactor work -> 3 minutes final approval queue`

The user should be able to:

1. `uv add roboharness` into an existing robot codebase
2. wire a small setup layer into the current grasp/debug pipeline
3. give the agent a task plus approval contract
4. walk away
5. come back to a small proof pack showing only materially changed or ambiguous cases

If the system works, the user does not watch the run. They only inspect the final
evidence surface and decide whether to bless the result.

## Why The Earlier Plan Was Wrong

The previous plan asked the wrong question: "Should we move `robots/` and
`controllers/` to a showcase repo?"

That question optimizes for conceptual purity, not user trust.

What the CEO review found:

- `pip install roboharness` is already lightweight at the package layer
- heavy integrations are already behind optional extras
- top-level exports already keep the core API relatively clean
- the most impressive demos are not just noise, they are proof that adoption tax is low

So the product problem is not "too many files in core." The product problem is:

**Can an agent refactor robot code unattended and return a proof surface that a human
can trust quickly?**

## Core User Outcome

The target user outcome is:

> "When I give roboharness to Claude Code or Codex, my grasp pipeline gains a visual
> and metric harness so the agent can iterate through a hard refactor without me in
> the loop. When it finishes, I inspect the final visual harness results and know
> whether we are in good condition."

That means roboharness is not just visual testing. It is an **unattended refactor
gate for robot behavior**.

## Product Modes

Roboharness needs two explicit approval modes.

### 1. Regression Mode

Use when behavior should stay materially the same.

- old baseline remains authoritative
- visible differences are suspicious by default
- hard metric regressions fail immediately
- surfaced cases are regressions and ambiguities

### 2. Migration Mode

Use when the user wants new behavior, not just code cleanup.

Example:
- reuse the existing 8 scenes
- change bottle side-grasp to ball top-down grasp
- require the final top-down view to show the palm-down pose

In migration mode:

- scenes may remain fixed while expected behavior changes
- the user prompt is compiled into an explicit contract before the run starts
- the agent may propose a new baseline
- the user blesses that baseline once at the end

This is not optional. Without explicit migration mode, an agent can launder a wrong
behavior change as "intended."

## Contract-First Model

Natural language is not the source of truth.

Before an unattended run starts, roboharness compiles the user prompt into a small
JSON contract. If the prompt is ambiguous, the skill must stop and ask the user
before execution.

Minimum contract fields:

- `mode`: `regression` or `migration`
- `cases`: which harness cases are in scope, and whether they are immutable
- `rules`: the approval rules for this run
- `runtime_policy`: ambiguity handling and stop policy
- `approval_policy`: what gets surfaced to the user at the end

Every rule must include:

- `judge`: `metric`, `visual`, or `hybrid`
- `evidence_at`: phase, view, or motion window used to verify the rule

If a rule cannot be grounded to `judge + evidence_at`, the run must not start.

### Rule Types

Three rule types are enough for v1:

1. `metric_gate`
   - hard pass/fail condition
   - example: object must be grasped at lift, failed alarm count must be zero

2. `visual_goal`
   - intended behavior to show the user and the agent
   - example: final grasp must show palm-down over the ball in the top-down view

3. `anti_goal`
   - bad-but-plausible behavior that should never be accepted
   - examples:
     - still side-grasping while the arm approaches from above
     - fingers touching the bottle instead of the ball
     - final sharp snap motion to fake the target pose

## Runtime Verdict Semantics

RoboHarness needs explicit verdicts, not fuzzy agent prose.

Required v1 verdicts:

- `PASS`
- `FAIL`
- `AMBIGUOUS`
- `CONTRACT_INVALID`

Key rule:

**`AMBIGUOUS` may trigger more evidence gathering, but it can never self-promote to
`PASS`.**

This is the trust boundary. The agent can keep working, but if ambiguity remains,
the run hands that ambiguity back to the user honestly.

## Stop Policy

The stop policy should combine smart agent behavior with a boring hard guard.

### Soft Stop

Claude/Codex may stop early when the goal is clearly unreachable.

### Hard Stop

V1 should stop when the same failure signature repeats, with a coarse rerun cap
behind it.

Recommended default:

- repeat failure signature limit: 2
- failure signature:
  - `case_id`
  - `phase_id`
  - `violated_rule_id`
- optional backstop: `max_reruns`

Views belong in the evidence pack, not the failure signature. The important
question is whether the agent is stuck on the same contract violation, not whether
the screenshots differ slightly.

## Final User Surface: Approval Queue

The first screen should not be a giant dashboard.

It should be an **approval queue**.

The first screen should surface only:

- materially changed cases
- ambiguous cases

It should hide unchanged cases by default, while still reporting the count:

- `8 total cases`
- `2 surfaced`
- `6 unchanged by contract`

For each surfaced case, the first screen should show:

- case id
- intended change / regression / ambiguity status
- one canonical proof panel
- the rules that passed, failed, or stayed ambiguous
- the hard metric verdicts
- one short sentence explaining why this case is surfaced

Unchanged cases can remain in the folder tree and deeper report views. The user
explicitly said that is enough.

## Material Change Policy

V1 should surface a case when any of these happen:

- a hard metric fails
- a visual judgment is ambiguous
- the run claims an intended migration success and needs review
- an anti-goal is hit at any point

Cases with no material change should stay off the first screen.

## Existing Code Leverage

This direction should build on what already exists instead of starting over.

Relevant existing assets:

- `autonomous_report.json` as machine-readable verdict source
- `phase_manifest.json` for failing phase, primary views, and rerun hints
- `alarms.json` for evaluator-backed failures
- `report.html` and the current `Current vs Baseline` evidence surface
- `examples/_mujoco_grasp_wedge.py` and its phase-local evidence model
- the existing `RobotHarnessWrapper`, `Harness`, and report-generation flow

This is important: the current MuJoCo wedge already looks like a seed of the right
product. The job is to generalize the trust contract, not to throw away the wedge
and reorganize folders for aesthetic reasons.

## Showcase Repo Role

The showcase repo is still useful, but its role changes.

It is not the primary strategic move anymore.

Recommended role:

- external proof that roboharness works as a pip-installed dependency
- secondary distribution surface for framework-specific integrations
- not the main answer to the core product question

That means:

- keep the showcase repo
- do not make "move everything out of core" the main plan
- do not expand showcase aggressively until the contract-first product loop lands
- keep load-bearing integration proof in the core repo for now

## In Scope

### Product Scope

- define a compiled JSON contract for unattended regression and migration runs
- make rule grounding mandatory with `judge + evidence_at`
- formalize `PASS / FAIL / AMBIGUOUS / CONTRACT_INVALID`
- build the changed-cases approval queue as the main return surface
- keep unchanged cases out of the first screen
- preserve human blessing as the final approval boundary

### Implementation Scope

- prototype the contract-first loop in the existing MuJoCo grasp wedge first
- use skill-driven AskUser behavior when a contract clause cannot be compiled safely
- keep leveraging current report artifacts instead of inventing a second reporting stack

## Not In Scope

- deleting `robots/` or `controllers/` in the first pass
- moving all advanced demos out of the core repo
- making the showcase repo the primary product bet
- letting natural-language prompts execute without compilation
- allowing ambiguous runs to self-pass
- showing every unchanged case on the first approval screen
- full multi-project orchestration across repos before the core approval loop works

## Phased Delivery

### Phase 1: Contract + Report Spec

- finalize contract schema
- finalize final-report schema
- document rule grounding and ambiguity semantics

### Phase 2: MuJoCo Grasp Wedge Prototype

- compile migration/regression contracts for the grasp example
- add surfaced-case approval queue
- keep folder-level deep drill-down for unchanged cases

### Phase 3: Skill Flow

- compile user prompts into JSON contracts
- use AskUser when required clauses are missing or ambiguous
- fail closed when the user does not resolve ambiguity

### Phase 4: Second Integration

- extend from the MuJoCo wedge into one higher-value integration path
- ideal candidate: LeRobot evaluation or another already-validated harness case

## Success Metrics

The right metrics are product-trust metrics, not repo-tidiness metrics.

Success looks like:

- a user can describe a run in natural language and get a compiled contract in minutes
- every contract clause either compiles cleanly or asks for clarification before run start
- an unattended run returns `PASS`, `FAIL`, or `AMBIGUOUS` with machine-readable reasons
- the first screen shows only surfaced cases and unchanged-case count
- a user can review the surfaced approval queue quickly without replaying the whole run
- migration runs produce a proposed new baseline that the user can bless once

## Open Questions

These are real and should stay explicit:

1. How should motion-window evidence be represented for anti-goals like "late sharp
   movement" when still images are insufficient?
2. What is the exact baseline blessing flow for accepted migration runs?
3. Which parts of the current MuJoCo wedge should be promoted into shared code, and
   which should stay example-local until a second use case proves the abstraction?
4. How small can the approval queue stay while still feeling honest to the user?

## Immediate Recommendation

Do not spend the next cycle deleting demos.

Spend it making unattended runs trustworthy:

- contract compilation
- ambiguity handling
- changed-case approval queue
- baseline blessing for migration mode

That is much closer to the product the user actually described.
