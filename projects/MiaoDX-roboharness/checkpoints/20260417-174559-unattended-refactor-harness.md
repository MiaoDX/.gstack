---
status: in-progress
branch: main
timestamp: 2026-04-17T17:45:59+08:00
session_duration_s: 7611
files_modified:
  - assets/g1/
  - docs/designs/unattended-refactor-harness-v1.md
  - showcase-repo-plan.md
---

## Working on: Unattended Refactor Harness

### Summary

Reframed the product discussion away from "clean up the repo" and toward a contract-first unattended refactor harness. The accepted direction is: `uv add roboharness`, compile a natural-language task into a JSON contract, let Claude/Codex run unattended for 2-3 hours, then return a changed-cases approval queue with proof. Rewrote the old `showcase-repo-plan.md` around this product story and mirrored the accepted design into `docs/designs/unattended-refactor-harness-v1.md`.

### Decisions Made

- The real product is trust compression, not repo purity: `2-3 hours unattended agent work -> 3 minutes final approval review`.
- The user wants roboharness to support unattended refactor and migration runs, then re-enter only at the final approval boundary.
- Two explicit run modes are required: `regression` and `migration`.
- Natural-language prompts are not authoritative; the skill/compiler must turn them into a compact JSON contract before execution.
- Every contract rule must be grounded with `judge` (`metric`, `visual`, or `hybrid`) plus `evidence_at` (phase/view/motion window). If a rule cannot be grounded, the run must stop and ask the user.
- Metrics are the hard safety floor. Visual evidence is the intent proof and the human trust surface.
- Runtime verdicts must include `PASS`, `FAIL`, `AMBIGUOUS`, and `CONTRACT_INVALID`.
- `AMBIGUOUS` may trigger more evidence gathering, but it may never self-promote to `PASS`.
- The first screen should be an approval queue that surfaces only materially changed or ambiguous cases. Unchanged cases stay in folders, with just a compact count in the summary.
- Migration runs may propose a new baseline, but the user blesses that baseline once at the end.
- Soft stop remains model-driven, but v1 should still have a boring hard guard: repeated failure signature (`case_id + phase_id + violated_rule_id`) with a coarse rerun ceiling behind it.
- The showcase repo is still useful as external proof of pip-install integration, but it is no longer the main strategic move. Do not aggressively move demos out of core until the contract-first loop lands.

### Remaining Work

1. Turn the contract-first design into an implementation plan for the existing MuJoCo grasp wedge, specifically how to compile prompts, persist contracts, and render the changed-cases approval queue.
2. Decide how v1 handles motion-window anti-goals like "late sharp snap" when still images are insufficient. The compiler should force a metric, explicit motion artifact, or a user clarification.
3. Rewrite the README/front door so the public story matches the new product statement instead of leading with demo-gallery energy only.
4. Decide which current MuJoCo wedge pieces stay example-local versus which ones should be promoted into shared code only after a second use case proves the abstraction.

### Notes

- This session was strategy/docs only. No tests were run.
- `showcase-repo-plan.md` was fully rewritten into a product plan. The old repo-split framing is gone from that file.
- New repo-local design artifact: `docs/designs/unattended-refactor-harness-v1.md`.
- `assets/g1/` was already untracked in the worktree and was intentionally left alone.
- The user explicitly prefers skill-driven flows, and for compile-time ambiguity wants existing AskUser tooling when running through the skill.
- The user does not want to think about low-level stop-signature details. The product should pick boring defaults and keep the UX out of the weeds.
