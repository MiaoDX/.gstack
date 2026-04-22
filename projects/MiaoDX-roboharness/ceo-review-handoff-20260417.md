# CEO Review Handoff — Roboharness Strategic Repositioning

**Date:** 2026-04-17  
**Branch:** main  
**Plan file:** `/home/mi/ws/gogo/roboharness/showcase-repo-plan.md`  
**Status:** Phase 1 (CEO Review) complete. Awaiting user final direction before Phase 2/3.

---

## What the User Originally Wanted

Aggressively remove `robots/` and `controllers/` from core roboharness repo. Move advanced demos (G1 WBC reach, SONIC planner, SONIC tracking, LeRobot G1) to showcase repo or delete them. Keep only core harness + `mujoco_grasp.py` + quickstart. Goal: clarify positioning as "visual testing harness for AI coding agents."

---

## What the CEO Review Actually Found

### Dual Voices Consensus

| Dimension | Consensus |
|-----------|-----------|
| Premises valid? | **DISAGREE on diagnosis** — both agree premises are weak. Claude: unvalidated assumptions. Codex: already solved at API layer. |
| Right problem to solve? | **CONFIRMED** — positioning confusion is real, but proposed solution is underambitious/misdirected. |
| Scope calibration correct? | **CONFIRMED risky** — deleting moat, keeping commodity. |
| Alternatives explored? | **CONFIRMED insufficient** — middle path (docs restructure) dismissed too quickly. |
| Competitive risks covered? | **CONFIRMED no** — larger platforms can subsume a stripped-down core. |
| 6-month trajectory sound? | **CONFIRMED questionable** — optimizes for tidiness before proving distribution. |

### Key Findings (Claude Subagent)
- **Critical:** Core premises are unvalidated — no evidence of user confusion, no PyPI stats, no GitHub import search, no user interviews.
- **High:** Two 6-month regret scenarios: (A) "boring generic tool" trap with no viral content; (B) "premature abstraction" trap with two-repo overhead.
- **High:** Three viable alternatives ignored: reframe-in-place, plugin architecture, packaging-only split.
- **Critical:** Competitive threats underanalyzed — W&B, Rerun, LeRobot, Isaac Lab, agent vendors.

### Key Findings (Codex)
- **Critical:** Problem already solved at package/API layer. `pip install roboharness` is core-only, heavy integrations are behind extras, top-level API does not export G1/SONIC controllers.
- **Critical:** "You are deleting the moat and keeping the commodity." A small harness core is easy for larger platforms to copy.
- **High:** Downgrades LeRobot, the dominant community platform already identified as existential.
- **High:** Conflicts with repo's own single-repo logic. This is fragmentation with better branding.
- **Medium:** Middle path (ruthless docs/nav restructuring) dismissed too quickly.

---

## The Real User Goal (Discovered Through Discussion)

The user's actual goal is NOT "less is more" or conceptual purity. It is:

> **Making users feel confident that roboharness integrates easily into their EXISTING pipeline without forcing them into a framework.**

This is why the showcase repo was created originally. The confusion is internal, not external.

---

## The Options Evaluated

### Option A: Keep all demos in core — messaging fix only
- Restructure README/docs to lead with "3-line integration into any existing pipeline."
- Demote G1/SONIC demos to "Reference Integrations" section.
- Keep optional extras so `pip install roboharness` stays lightweight.
- Archive or leave the showcase repo as-is (don't expand it).

**Pros:** Single repo, single CI, all viral demos stay, no cross-repo breakage, simplest maintenance, strongest proof of easy integration.
**Cons:** Core repo has more files (but this does not affect users because deps are optional extras).

### Option B: Full split — move all advanced demos to showcase repo
- Core becomes harness + backends + wrappers + lightweight examples only.
- Showcase repo becomes the home for G1/SONIC/GR00T integrations.

**Pros:** Cleanest conceptual separation.
**Cons:** Both CEO voices strongly opposed this for a pre-adoption, single-maintainer project. Deletes the moat. Doubles operational overhead. Loses viral content from main repo. Cross-repo API breakage risk.

### Option C: Partial split — keep mujoco_grasp + quickstart + maybe one LeRobot eval in core, move the rest to showcase
- This was the original plan's specific actions.

**Pros:** Core looks cleaner, still has some proof points.
**Cons:** Removes the most impressive integration proofs (G1 WBC reach, SONIC planner, SONIC tracking). Weakens the "easy integration into serious pipelines" message. Still introduces cross-repo coordination.

---

## My Recommendation

**Choose Option A.**

The CEO review consensus is clear: the problem is messaging and information architecture, not repo structure. The existing package/API layer already solves the "users feel forced into a framework" problem:
- `pip install roboharness` is core-only.
- Heavy integrations are behind optional extras.
- Top-level `__init__.py` does not export robot-specific controllers.

What is missing is ruthless README/docs restructuring. The README currently leads with robot GIFs. It should lead with the 3-line integration snippet and the promise: "Drop this into your existing Gymnasium / MuJoCo / LeRobot / Isaac Lab pipeline."

The G1/SONIC demos are not a distraction — they are the proof that this promise is true. Removing them makes roboharness look like a toy utility (`mujoco_grasp.py`) rather than infrastructure that serious robotics teams can adopt.

The showcase repo already exists. Don't delete it abruptly, but don't expand it either. Let it sit. If adoption grows and integrations become overwhelming in 6–12 months, revisit the split then.

---

## For the Codex Double-Check

If you want to run this past Codex, the key question is:

> "Given that the user's real goal is signaling 'easy integration into existing pipelines,' and given that the package/API layer already separates core from heavy integrations (optional extras, no G1/SONIC in top-level exports), should we restructure README/docs (Option A) or move code to a showcase repo (Option B/C)?"

The CEO review found that both independent voices agreed: restructure messaging, don't delete the moat.

---

## Next Step

Once the user confirms the direction (A, B, or C), update the plan file to reflect the chosen path and proceed to Phase 3 (Eng Review) and Phase 3.5 (DX Review) of the autoplan pipeline.
