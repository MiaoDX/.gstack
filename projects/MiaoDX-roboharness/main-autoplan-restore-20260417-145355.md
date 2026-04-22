# /autoplan Restore Point
Captured: 2026-04-17T06:53:55Z | Branch: main | Commit: aabcad2

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State

# Roboharness Strategic Repositioning Plan


## Problem Statement

The core tension is: what is roboharness actually selling?

Right now it looks like three different products shoved into one repo:

1. A visual testing harness — "pause → capture → resume" for any gym env
2. A robot controller library — GrootLocomotionController, SonicLocomotionController, WBC IK
3. A demo suite — G1 walking, SONIC tracking, MuJoCo grasp

The showcase repo exists because #2 and #3 were making users think "this is a framework I have to live inside." But here's the harder question: why do the robot controllers exist in core at all?

GrootLocomotionController downloads ONNX models from HuggingFace and runs inference. That's not a harness. That's a robot policy. Same for SonicLocomotionController and HolosomaLocomotionController. They have nothing to do with "visual testing for AI coding agents."

## Proposed Change

If the positioning is to be clean, the answer is aggressive: remove robots/ and controllers/ from core entirely. Move them to showcase or kill them. The core repo should be:

- core/ — Harness, Checkpoint, Capture
- backends/ — MuJoCo/Meshcat, maybe Isaac Lab adapter
- wrappers/ — RobotHarnessWrapper
- evaluate/ — constraint evaluation, maybe the LeRobot plugin if it's generic enough
- storage/, reporting/, cli/

That's it.

## Tradeoff Analysis

Those controllers were built so the examples could run. The examples generate the GitHub Pages reports. The reports are the visual proof that roboharness works. If the controllers are ripped out, the G1 demos in core CI are lost.

So the real question is: what is the minimum viable demo that proves roboharness works?

mujoco_grasp.py is perfect for this. It's small, needs no external models, runs headless in CI, and produces beautiful multi-view checkpoints. It fully demonstrates the harness value proposition.

The G1 demos are impressive but they're actually demos of NVIDIA's SONIC policy and GR00T WBC, not demos of roboharness. The harness part is just RobotHarnessWrapper(env, checkpoints=[...]) — 3 lines.

## Specific Actions

- Core repo: keep mujoco_grasp.py, quickstart_gymnasium.py, and maybe one LeRobot eval example
- Move or delete: lerobot_g1.py, lerobot_g1_native.py, sonic_locomotion.py, sonic_tracking.py, g1_wbc_reach.py
- Extract to showcase: robots/unitree_g1/ and controllers/
- Showcase repo: becomes the home for "here's how you integrate roboharness with GR00T / SONIC / LeRobot G1"

## Counter-Argument

"But those G1 demos are cool. They get attention. They make the README look impressive."

That's true. But attention for what? If a user stars the repo because of the G1 walking GIF, then discovers the library is actually a testing harness, they feel baited. If they star it because mujoco_grasp.py solved their "I need visual regression testing for my robot policy" problem, they're the right user.

## Open Questions

1. Are the GitHub Pages visual reports important for acquisition? If yes, can mujoco_grasp.py + one simple env carry that alone?
2. Is anyone externally actually importing roboharness.robots.unitree_g1? If no, the break is free.
3. Do you want roboharness to be cited in robotics research, or used by AI coding agent builders? The G1 demos help with the first. The harness API helps with the second.
