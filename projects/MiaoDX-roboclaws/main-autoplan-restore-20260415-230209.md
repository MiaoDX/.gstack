# /autoplan Restore Point
Captured: 2026-04-15T23:02:09Z | Branch: main | Commit: $(git rev-parse --short HEAD)

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
<!-- /autoplan restore point: /home/mi/.gstack/projects/MiaoDX-roboclaws/main-autoplan-restore-20260415-225110.md -->
# Phase 2: OpenClaw Integration — Completion Plan

## Problem Statement

Phase 2 aims to make each simulation agent a first-class OpenClaw instance with persistent memory and a SOUL-driven personality. The core code exists (`bridge.py`, `skill.py`, SOUL presets, skill definition), but the integration is not yet proven end-to-end in CI. The ephemeral OpenClaw smoke test fails on every `main` run because the Gateway Docker container does not become ready within 60 seconds.

**Premises:**
1. The OpenClaw Gateway Docker image contract (workspace path, env vars, health endpoints) is the most likely drift point.
2. Local developers need a documented, reproducible way to run the Gateway before they can validate OpenClaw behavior.
3. The `AI2THORNavigatorSkill` class has no direct unit tests — only the HTTP bridge is tested.
4. GitHub Pages publishing already handles best-effort OpenClaw artifacts; the blocker is purely artifact generation.

## Scope

### In scope
- Diagnose and fix the ephemeral OpenClaw Gateway startup in CI
- Add unit tests for `AI2THORNavigatorSkill` (`roboclaws/openclaw/skill.py`)
- Add a local OpenClaw Gateway quick-start guide to `CLAUDE.md` or a new `docs/openclaw-local.md`
- Ensure OpenClaw smoke reports (territory + coverage) generate and upload successfully on `main`
- Update `README.md` Phase 2 checkbox from "in progress" to "complete" once CI is green

### Not in scope
- Railway OpenClaw integration hardening (already optional and working when secrets are present)
- Changes to Phase 1 VLM logic or game rules
- Phase 3 (Isaac Lab migration)
- New SOUL presets beyond the existing three

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AI2-THOR Engine                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Agent 0     │  │ Agent 1     │  │ Agent 2     │         │
│  │ frame       │  │ frame       │  │ frame       │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                 │
│         ▼                ▼                ▼                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         OpenClawProvider (VLMProvider adapter)       │   │
│  │  Writes frames to temp dir → POST /tools/invoke     │   │
│  └─────────────────────────────────────────────────────┘   │
│         │                │                │                 │
│         ▼                ▼                ▼                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         OpenClaw Gateway (Docker / localhost)        │   │
│  │  Session: roboclaws-agent-0  → SOUL + MEMORY        │   │
│  │  Session: roboclaws-agent-1  → SOUL + MEMORY        │   │
│  │  Session: roboclaws-agent-2  → SOUL + MEMORY        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Implementation Plan

### Task 1: Diagnose CI Gateway startup failure

**Investigation steps:**
1. Add `docker logs openclaw-gateway` output to the "Wait for Gateway readiness" step so failures are actionable.
2. Check if the Gateway image `ghcr.io/openclaw/openclaw:latest` has changed its expected skill mount path or boot command.
3. Verify whether `/healthz` vs `/readyz` behavior changed — some versions report 200 on `/healthz` before the skill scanner finishes.
4. Test locally: `docker run` with the exact same flags as CI and observe logs.

**Expected fix:** One of:
- Mount path update (e.g., `/home/node/.openclaw/workspace/skills/` → new path)
- Additional env var required (e.g., `OPENCLAW_WORKSPACE=/home/node/.openclaw/workspace`)
- Readiness probe should hit `/healthz` first, then `/readyz`
- Image tag pin to a known-working version instead of `latest`

### Task 2: Add OpenClaw skill unit tests

**New file:** `tests/test_openclaw_skill.py`

**Test cases:**
- `test_skill_loads_preset_soul` — verify `aggressive`, `defensive`, `cooperative` presets load from `skills/ai2thor-navigator/souls/`
- `test_skill_accepts_custom_soul_string` — pass raw markdown, verify it is used verbatim
- `test_skill_encodes_frames` — verify `encode_frame` produces a valid base64 JPEG
- `test_skill_calls_provider_with_soul_in_state` — mock provider, assert `state["soul"]` is populated
- `test_skill_validates_unknown_actions` — mock provider returns `"Jump"`, skill should fall back to `"MoveAhead"`
- `test_list_souls` — verify `AI2THORNavigatorSkill.list_souls()` returns the 3 built-in names

### Task 3: Local OpenClaw dev documentation

**New file:** `docs/openclaw-local.md`

**Contents:**
- One-liner Docker run command matching CI
- Expected Gateway logs at startup
- How to verify readiness with `curl`
- How to run `examples/territory_game.py --backend openclaw` locally
- Troubleshooting: "Gateway did not become ready" → check mount path, check image tag, check token

**Update `CLAUDE.md`:** Add a cross-reference under the "Cloud vs local development" section.

### Task 4: Verify OpenClaw report publishing end-to-end

**CI changes:**
- Ensure `openclaw-smoke` job uploads `report-openclaw` artifact when successful
- Confirm `publish-pages` best-effort download slots it into `site/openclaw/`
- No code changes expected unless artifact paths drifted

**Validation:** After merge to `main`, verify:
- `https://miaodx.github.io/roboclaws/openclaw/territory/report.html` loads
- `https://miaodx.github.io/roboclaws/openclaw/coverage/report.html` loads

### Task 5: Mark Phase 2 complete

**Update `README.md`:**
- Change `- [ ] **Phase 2**: OpenClaw integration` to `- [x] **Phase 2**: OpenClaw integration`

## Test Plan

| Codepath | Test | Location |
|----------|------|----------|
| Bridge construction | `test_bridge_uses_defaults` | `tests/test_bridge.py` (existing) |
| Bridge HTTP errors | `test_bridge_step_*` | `tests/test_bridge.py` (existing) |
| Provider adapter | `test_provider_get_action` | `tests/test_bridge.py` (existing) |
| Skill SOUL loading | `test_skill_loads_preset_soul` | `tests/test_openclaw_skill.py` (new) |
| Skill frame encoding | `test_skill_encodes_frames` | `tests/test_openclaw_skill.py` (new) |
| Skill provider integration | `test_skill_calls_provider_with_soul_in_state` | `tests/test_openclaw_skill.py` (new) |
| Skill action validation | `test_skill_validates_unknown_actions` | `tests/test_openclaw_skill.py` (new) |
| CI Gateway startup | Ephemeral smoke job | `.github/workflows/ci.yml` |
| Report publishing | OpenClaw reports on GitHub Pages | `publish-pages` job |

## Error & Rescue Registry

| Error | Cause | Rescue |
|-------|-------|--------|
| `Gateway did not become ready within 60s` | Docker image path/env drift | Read logs, update mount or env vars |
| `OpenClawUnavailable: Gateway unreachable` | Local Gateway not running | Point user to `docs/openclaw-local.md` |
| Unknown action from Gateway | Skill or bridge parsing mismatch | Fallback to `MoveAhead` (already implemented) |

## Failure Modes

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| OpenClaw upstream changes image contract again | CI breaks | Pin to a specific image tag once stable |
| Railway secret missing | Optional job skips | Already handled with `continue-on-error` |
| Local developer lacks Docker | Can't run Gateway | Document that `--backend vlm` is the local fallback |

## Effort Estimate

- Task 1 (CI fix): 30–60 min
- Task 2 (skill tests): 30 min
- Task 3 (docs): 20 min
- Task 4 (publish verification): 10 min
- Task 5 (README update): 5 min

**Total: ~2 hours**
