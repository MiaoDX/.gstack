# Design Audit: MuJoCo Grasp Report

- Target: `http://127.0.0.1:8766/report.html`
- Source branch under review: `codex/design-review-mujoco-20260415`
- Runtime source: fresh `MUJOCO_GL=egl python examples/mujoco_grasp.py --output-dir tmp/design_review_report --report`
- Classifier: App UI / static diagnostic artifact
- Status: DONE

## Headline

This page is not AI slop. It is a functional diagnostics surface with a few sharp edges that were actively hurting trust.

The real problems were not taste problems. They were user problems:
- the page advertised interactive 3D scenes and then loaded five dead iframes
- mobile and tablet users had to pan sideways because the report let wide tables and a fixed-width viewer blow out the page
- pass-state metadata contradicted the main action copy and still hinted at a rerun plus a fake root cause

Those are fixed on the review branch.

## First Impression

The site communicates **deterministic robot-debug artifact, not marketing fluff**.

I notice **the verdict and summary hit fast, but the page used to lose credibility lower down because the "Interactive 3D Scene" blocks were broken and the report overflowed on small screens**.

The first 3 things my eye goes to are: **the page title**, **the PASS verdict banner**, **the Current vs Baseline summary section**.

If I had to describe this in one word: **useful**.

## Inferred Design System

- Fonts: `-apple-system, sans-serif` for the page, `monospace` for code-like contract fields
- Colors: cool utilitarian palette, blue summary surface, green pass states, amber warnings, red failure states
- Heading scale: `h1 32px`, `h2 24px`, `h3 ~18.7px`, `h4 16px`
- Layout style: carded report shell, dense but readable, internal-tool energy rather than brand-forward design

No repo `DESIGN.md` was found. This audit used the rendered system and general usability rules.

## Scores

- Design Score: `C` -> `B+`
- AI Slop Score: `A` -> `A`

Why it moved: the shipped artifact already had decent hierarchy and zero SaaS-template nonsense. The big drag was broken interaction trust plus bad responsive containment, not brand or composition.

## Findings

### FINDING-001
- Impact: High
- Title: Embedded 3D scenes were broken in the shipped report
- What was wrong: the report HTML lived at the output root, but iframe `src` values pointed at `plan/meshcat_scene.html`, `pre_grasp/meshcat_scene.html`, and friends instead of the real `mujoco_grasp/trial_001/...` paths. The page loaded five 404s.
- Why it matters: users see a block labeled `Interactive 3D Scene` and get a dead surface. That makes the whole artifact feel unreliable.
- Fix: compute meshcat paths relative to the report output root and add iframe titles.
- Source commit: `4598c97`
- Regression commit: `393b202`
- Verification: browser console no longer shows 404 errors for meshcat scenes. Remaining console output is WebGL performance warning noise, not missing resources.
- Evidence: `screenshots/finding-001-after.png`

### FINDING-002
- Impact: High
- Title: Report forced horizontal scrolling on mobile and tablet
- What was wrong: on a `375px` viewport the document `scrollWidth` was `761px`. The evaluation table overflowed the page, the artifact pack table could expand on long code tokens, and the 3D viewer held a fixed `480px` width.
- Why it matters: users on a phone or split-screen laptop had to pan sideways just to read the report. That's a broken reading experience.
- Fix: wrap wide tables in `.table-scroll`, allow code tokens to wrap, remove the hard `min-width: 300px` view strip, and make the meshcat viewer/iframe width responsive.
- Source commit: `645313e`
- Regression commit: `b7d36b8`
- Follow-up formatting commit: `f99e897`
- Verification: mobile `scrollWidth` now equals viewport width (`375`), and the meshcat viewer collapses to `295px` wide instead of forcing page overflow.
- Evidence: `screenshots/finding-002-before-mobile.png`, `screenshots/finding-002-after-mobile.png`, `screenshots/finding-002-after-tablet.png`, `screenshots/finding-002-after-desktop.png`

### FINDING-003
- Impact: Medium
- Title: Pass-state metadata contradicted the main action copy
- What was wrong: the agent panel correctly said `No rerun required`, but the Artifact Pack card still showed `Root cause trajectory_regression` and `Rerun hint restore:plan` on successful runs.
- Why it matters: this makes users second-guess a passing report and gives agents bad machine-facing hints in the pass path.
- Fix: successful manifests now emit `suspected_root_cause="none"`, `rerun_hint="not_required"`, and empty evidence paths. The summary card renders `Root cause: none` and `Rerun hint: No rerun required`.
- Source commit: `54e986e`
- Regression commit: `8c7068a`
- Follow-up formatting commit: `f99e897`
- Verification: live page text now shows `Root cause none` and `Rerun hint No rerun required`, with no `trajectory_regression` or `restore:plan` on the pass-state report.
- Evidence: `screenshots/finding-003-after.png`

## Trunk Test

- What site is this: PASS
- What page am I on: PASS
- What are the major sections: PASS
- What are my options at this level: PARTIAL, this is a static artifact so there are few explicit actions
- Where am I in the scheme of things: PASS, the phase timeline and checkpoint sections are obvious
- How can I search: N/A for this artifact

Verdict: `PASS` for a static debug report. The page is scannable without extra explanation.

## Quick Wins Applied

1. Fixed embedded 3D scene URLs so the report stops shipping dead panes.
2. Contained wide tables and fixed-width viewer blocks so mobile no longer overflows.
3. Removed misleading pass-state rerun and root-cause metadata.

## Verification

- `ruff check .`
- `ruff format --check .`
- `mypy src/`
- `pytest -q` -> `466 passed, 9 skipped`, coverage `95.44%`
- Live browser verification on local report artifact:
  - embedded meshcat 404s removed
  - mobile `scrollWidth` reduced from `761` to `375`
  - pass-state metadata now aligns with `No rerun required`

## Deferred

No additional high-impact or medium-impact visual issues were left open after this pass.

A future nice-to-have, not a blocker: export a repo `DESIGN.md` if you want a stable visual contract for report artifacts and project pages instead of inferring the system each time.

## PR Summary

Design review found `3` issues, fixed `3`. Design score `C` -> `B+`, AI slop score `A` -> `A`.
