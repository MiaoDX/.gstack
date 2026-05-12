# /autoplan Restore Point
Captured: 2026-05-07T16:27:27+08:00 | Branch: dongxu-dev-0507 | Commit: cfe30f1

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
# MolmoSpaces Manipulation Spike

**Status:** Draft pre-plan
**Workflow:** Matt-style plan first; optionally split with `to-issues`; ingest into GSD only when ready to execute
**Created:** 2026-05-07

## Why This Exists

Roboclaws is shifting its strategic center from direct VLM game loops toward
OpenClaw / coding-agent robot control through a small tool contract.

AI2-THOR remains the stable baseline and regression backend. MolmoSpaces is the
future-default substrate for manipulation work because it has richer scenes,
manipulable objects, grasp data, and a path toward MuJoCo / ManiSkill / Isaac.

This spike should prove one agent can manipulate objects before we attempt
multi-agent territory/coverage on the new substrate.

## Decisions

- Use a dual-backend transition: keep AI2-THOR, add MolmoSpaces.
- Direct coding-agent MCP path comes first.
- OpenClaw path comes second after direct MCP works.
- Territory/coverage stay AI2-THOR-only until one-agent MolmoSpaces
  manipulation works well.
- Room randomization happens before the run, outside MCP.
- Semantic/scripted manipulation is acceptable for v0.
- Split-model navigation work is paused until this path proves useful.

## Layer 1: Smallest Demo

Goal: deterministic pick/place with one object and one receptacle.

Example:

```text
Pick up the book from the floor and place it on the table.
```

Expected tool flow:

```text
observe -> scene_objects -> goto/reach -> pick -> place -> observe -> done
```

Pass criteria:

- Object ends on the required receptacle.
- `trace.jsonl` records all tool calls.
- `run_result.json` records backend, scenario id, final status, and artifact paths.
- `report.html` shows before/after snapshots and the tool trace.
- The report states whether `pick` / `place` used real MolmoSpaces APIs, a
  scripted planner, or a temporary shim.

## Layer 2: Open Cleanup Demo

Goal: natural-language room cleanup.

Operator prompt:

```text
帮我整理这个房间
```

Pre-run setup should create a seeded messy room, outside MCP:

```bash
python scripts/prepare_molmospaces_room.py \
  --scenario cleanup-room \
  --seed 7 \
  --messiness easy \
  --output output/runs/<id>/scenario.json
```

Private scoring manifest example:

```json
{
  "misplaced": [
    {
      "object_type": "Book",
      "start": "floor",
      "valid_targets": ["Bookshelf", "Desk", "CoffeeTable"]
    },
    {
      "object_type": "Cup",
      "start": "floor",
      "valid_targets": ["Table", "CounterTop"]
    }
  ]
}
```

The agent must not see this private manifest. It should infer the room state
through `observe` and `scene_objects`.

Pass criteria:

- At least 3 of 5 misplaced objects move to valid receptacles.
- No high-severity failure: lost object, impossible placement marked as success,
  or timeout with no progress.
- `report.html` shows initial room, final room, restored/missed object table,
  and tool trace.

## Initial MCP Surface

Keep the tool surface small:

- `observe(label="")`
- `scene_objects(filter_types="")`
- `goto` or `reach`
- `pick(object_id)`
- `place(receptacle_id | location_id)`
- `open(object_id)`
- `close(object_id)`
- `done(reason)`

The tool contract is the abstraction boundary. Do not build a broad backend
framework before the MolmoSpaces API shape is known.

## Proposed Task Slices

These are draft vertical slices. If this plan is approved after review, run
`to-issues` to turn them into tracker issues before GSD ingest, unless the team
decides to skip the issue layer and go straight to GSD.

1. **Capability spike**
   Install/run the smallest MolmoSpaces/MuJoCo example, identify APIs for scene
   load, object inventory, object state changes, camera frames, and scripted
   manipulation.

2. **Direct MCP Layer 1**
   Add the minimal MolmoSpaces MCP server/entry point and complete deterministic
   pick/place with artifacts.

3. **Scenario builder and scoring**
   Add seeded room messiness outside MCP, private manifest, state-delta scoring,
   and report rendering.

4. **Direct MCP Layer 2**
   Run `帮我整理这个房间` through a coding agent and validate 3-of-5 cleanup.

5. **OpenClaw follow-up**
   Reuse the working MCP surface through OpenClaw. Validate Layer 1 first, then
   Layer 2.

6. **Docs and ADR**
   Reframe README / ARCHITECTURE / technical design after evidence exists. Add
   an ADR that AI2-THOR remains baseline and MolmoSpaces is the next substrate
   for manipulation.

## Deferred

- Multi-agent MolmoSpaces.
- Territory/coverage on MolmoSpaces.
- Low-level arm control and VLA action experts.
- Isaac Lab humanoid migration.
- Full backend-neutral simulator abstraction.
- Split-model navigation optimization.

## GSD Handoff Trigger

Ingest this plan into GSD only after the capability spike identifies the actual
MolmoSpaces APIs and the implementation slices are no longer speculative.

At that point:

- Optionally run `to-issues` first if the work should be divided across multiple
  agents or tracked in GitHub Issues.
- Add/update the phase in `.planning/ROADMAP.md`.
- Create `.planning/phases/<phase>/` from this doc.
- Let GSD own execution, validation, summaries, and shipped state.

During implementation, use `tdd` inside the slices where behavior needs to drive
the code: scenario scoring, manifest parsing, MCP tool contracts, artifact schema,
and any regression found during local MolmoSpaces validation.
