# /autoplan Restore Point
Captured: 2026-04-16 21:41:52 UTC | Branch: main | Commit: bb89253

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
<!-- /autoplan restore point: /home/mi/.gstack/projects/MiaoDX-roboharness/main-autoplan-restore-20260416-214152.md -->
# Showcase Repository Plan

## Problem
GR00T N1.6, Pi0, LeRobot, SONIC, etc. need to demonstrate roboharness integration, but embedding them in the core repo would bloat it and explode dependencies. Core repo must stay self-contained.

## Solution
Create a dedicated `github.com/roboharness/showcase` repository for integration demos.

## Structure

```
roboharness/showcase/
├── README.md                     # Overview + run instructions
├── groot-n16/
│   ├── README.md
│   ├── requirements.txt          # groot deps + roboharness
│   ├── run.sh                    # one-command demo
│   ├── harness_config.yaml       # checkpoint definitions
│   └── groot_visual_test.py      # thin wrapper, not a submodule
├── pi0-libero/
│   ├── README.md
│   ├── requirements.txt
│   ├── run.sh
│   └── pi0_eval_harness.py
├── lerobot-g1/
│   └── ...
├── sonic-locomotion/
│   └── ...
├── capx-comparison/              # Comparison demo with CaP-X
│   └── ...
└── .github/
    └── workflows/
        └── showcase-ci.yml       # Each showcase is an independent CI job
```

## Key Design Decisions

- **No git submodules**: Use requirements.txt + runtime pip install to pull dependencies. Avoids submodule hell.
- **Each showcase is self-contained**: `cd showcase/groot-n16 && pip install -r requirements.txt && python groot_visual_test.py`
- **roboharness itself is a pip dependency**, not a submodule. Showcase always uses the PyPI release.
- **CI matrix**: Each showcase is an independent job — one failing doesn't block others.

## Benefits

- Core repo stays self-contained, no bloat
- Users see "roboharness works with GR00T / Pi0 / LeRobot" with runnable demos
- Issue #91 scope shifts from "support every framework in roboharness" to "showcase integrations in a dedicated repo"
- Version changes in large models/frameworks don't break core CI

## Prerequisites

GitHub org `roboharness` already exists (no repos yet).

## Action Items

- [ ] Create `.github` repo in `roboharness` org with `profile/README.md` (org landing page)
- [ ] Create `roboharness/showcase` repo
- [ ] Initialize skeleton: README + groot-n16/ + pi0-libero/ + lerobot-g1/ directories
- [ ] First runnable showcase: extract `examples/lerobot_g1_native.py` as standalone showcase
- [ ] Set up CI: GitHub Actions matrix, one job per showcase

## Exit Criteria

Showcase repo has at least 3 runnable showcases (GR00T, Pi0, LeRobot), each with CI.
