# Eng Review Test Plan — Phase 02.8 Split-Model Vision

## Target

Phase bundle: `.planning/phases/02.8-split-model-vision`

## Affected Files

- `spike/split_model_probe.py`
- `.planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md`
- `roboclaws/openclaw/vision_bridge.py`
- `roboclaws/openclaw/mcp_server.py`
- `skills/ai2thor-navigator/SKILL.md`
- `examples/openclaw_nav_autonomous.py`
- `scripts/openclaw-bootstrap.sh`
- `Makefile`
- `examples/openclaw_interactive.py`
- `scripts/tail-openclaw-chat.py`
- `docs/openclaw-local.md`
- `tests/test_openclaw_vision_bridge.py`
- `tests/test_openclaw_mcp_server.py`
- `tests/test_openclaw_bootstrap.py`
- `tests/test_openclaw_interactive.py`
- `tests/test_tail_openclaw_chat.py`
- `tests/test_openclaw_nav_autonomous.py`

## Key Interactions to Verify

- Plan 01 proves whether the Gateway `image` tool is actually usable for the two-image navigation case, not just exposed in a tool list.
- Plan 02 keeps image-bearing observe results for vision-capable models and switches text-only models to an exact two-text-block observe contract.
- Plan 02 emits truthful observe metadata: `observe_delivery`, `bridge_model`, `image_labels`, `bridge_latency_s`, and `bridge_error`.
- Bridge failure never falls back to raw image passthrough for text-only models.
- Plan 03 makes selector/bootstrap help text truthful about split-model behavior and keeps `profile: minimal` closed by default.
- Plan 03 makes interactive/operator output explicit about `text-bridge` delivery.
- Plan 04 local validation records both split-model and direct-vision control results on the same scene/budget.

## Edge Cases

- `ROBOCLAWS_OBSERVE_MODE=auto` picks the wrong branch for a text-only model.
- `image_labels` still advertise raw images in text-bridge mode.
- Observe response shape collapses into one text blob and breaks downstream tooling or operator diagnosis.
- Bridge model is unset and the fallback path is ambiguous.
- Bridge failure leaks raw image blocks to a text-only main model.
- Validation passes after only one lucky move instead of a non-trivial run.
- Spike helper/file-path drift makes the validation map lie about what exists.
- Tail output stays opaque (`parts=[...]`) even after text-bridge behavior ships.

## Critical Paths

- `roboclaws__observe` image path: engine -> prompt bundle -> MCP image blocks -> agent.
- `roboclaws__observe` text-bridge path: engine -> prompt bundle -> vision bridge -> two text blocks -> agent.
- Selector path: `Makefile` -> `scripts/openclaw-bootstrap.sh` -> Gateway config -> interactive/autonomous entrypoints.
- Operator proof path: `examples/openclaw_interactive.py` -> Gateway Control UI -> `scripts/tail-openclaw-chat.py`.
- Local gate path: focused pytest slice -> split-model autonomous run -> direct-vision control run -> docs update.
