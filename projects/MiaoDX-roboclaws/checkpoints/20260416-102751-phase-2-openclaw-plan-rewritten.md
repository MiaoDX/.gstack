---
status: in-progress
branch: main
timestamp: 2026-04-16T10:27:51+08:00
files_modified:
  - PLAN.md
---

## Working on: Phase 2 OpenClaw plan rewritten after autoplan review

### Summary

Ran the full autoplan review pipeline (CEO → Design → Eng ×2 → DX) on the original Phase 2 OpenClaw completion plan. The review found the plan was fixing the wrong layer (Gateway readiness vs host/container shared volume), proposing duplicate tests, and using hollow completion criteria. After user direction, the plan was rewritten to focus on a standalone `examples/openclaw_demo.py` that proves OpenClaw can control AI2-THOR robots, with local validation first and then CI. The revised plan is saved in `PLAN.md`.

### Decisions Made

- **Demo approach**: Create a separate `examples/openclaw_demo.py` instead of fixing the existing `--backend openclaw` switch in `territory_game.py`/`coverage_game.py`.
- **Transport contract**: Use a project-relative shared directory `./.openclaw-tmp` (override via `OPENCLAW_WORK_DIR`) so host and Gateway container can both access the frame JPEGs. CI must mount this with identical absolute paths inside the container.
- **Scope reduction**: Per-agent SOUL preset assignment via the Gateway is deferred to the future "long-running OpenClaw instances" work. The demo only needs to prove robot control.
- **CI strategy**: Replace the failing `openclaw-smoke` job with the simpler demo run.
- **Local-first**: Validate the demo on the local machine before pushing to `main`, per `CLAUDE.md` cloud/local split rules.

### Remaining Work

1. **Task 1**: Fix `OpenClawProvider` to use `./.openclaw-tmp` (or `OPENCLAW_WORK_DIR`) and update CI bind mount.
2. **Task 2**: Create `examples/openclaw_demo.py` — 2 agents, 20 steps, pure navigation, outputs replay GIF + report.
3. **Task 3**: Write `docs/openclaw-local.md` with exact Docker one-liner including bind mount.
4. **Task 4**: Replace `openclaw-smoke` CI job to run the new demo.
5. **Task 5**: Local validation — run demo, confirm GIF/report, push to `main`.
6. **Task 6**: Update README Phase 2 checkbox and demo section when Pages artifact is live.

### Notes

- The only file modified so far is `PLAN.md`. No code has been written yet.
- The original `--backend openclaw` path in `territory_game.py` and `coverage_game.py` is intentionally left untouched.
- Issue #52 (coverage gameplay mismatch) is explicitly out of scope for this plan.
- `tests/test_skill.py` already covers `AI2THORNavigatorSkill`; no new skill tests are needed.
