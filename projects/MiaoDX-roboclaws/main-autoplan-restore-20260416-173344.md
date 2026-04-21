# /autoplan Restore Point
Captured: 20260416-173344 | Branch: main | Commit: 765f48a

## Re-run Instructions
1. Copy 'Original Plan State' below back to PLAN.md
2. Invoke /autoplan

## Original Plan State

<!-- /autoplan restore point: /home/mi/.gstack/projects/MiaoDX-roboclaws/main-autoplan-restore-20260415-230209.md -->
# Phase 2: OpenClaw Integration — Completion Plan

## Problem Statement

Phase 2 aims to prove that AI2-THOR simulated robots can be controlled through an OpenClaw Gateway, producing a visible demo that populates the README's third layer. The existing `--backend openclaw` path in `territory_game.py` and `coverage_game.py` is broken due to a host/container path mismatch, over-complicated for a first demo, and bypasses the `AI2THORNavigatorSkill` wrapper entirely.

## Scope

### In scope
- Create a standalone `examples/openclaw_demo.py` that runs a simple multi-agent AI2-THOR scenario via `OpenClawProvider`
- Fix the host/container shared image directory contract (`work_dir`)
- Write `docs/openclaw-local.md` with the exact Docker run command including the bind mount
- Validate the demo locally first, then update CI `openclaw-smoke` to run the new demo
- Ensure the demo produces replay GIF + `report.html` that can be published to GitHub Pages
- Update README Phase 2 checkbox when the demo is live

### Not in scope
- Per-agent SOUL preset assignment via the Gateway (deferred to long-running instance work — the new `examples/openclaw_demo.py` uses the Gateway's default skill configuration)
- Territory / Coverage game logic over OpenClaw (the new demo is pure navigation; game modes return in a later phase with long-running Gateway instances)
- Fixing the coverage gameplay mismatch tracked in issue #52
- Phase 3 (Isaac Lab migration)
- Remote/Railway OpenClaw integration (the `openclaw-railway-smoke` job is dropped in Task 7 — remote Gateways can't share a host bind-mount for frame paths; they'll need a different transport)

## Architecture

```
┌─────────────────────────────────────────────┐
│              AI2-THOR Engine                │
│  agent-0 frame ──► OpenClawProvider         │
│  agent-1 frame ──► (writes to ./.openclaw-  │
│                   │   tmp/agent-{id}.jpg)   │
│                   ▼                         │
│         ┌─────────────────────┐             │
│         │  OpenClaw Gateway   │             │
│         │  (Docker container) │             │
│         │  POST /tools/invoke │             │
│         │  session per agent  │             │
│         └─────────────────────┘             │
└─────────────────────────────────────────────┘
```

## Implementation Plan

### Task 1: Fix the shared image directory contract

**Files:** `roboclaws/openclaw/bridge.py`, `.github/workflows/ci.yml`, `.gitignore`

1. Change `OpenClawProvider.__init__` to resolve `work_dir` via a 3-way fallback:
   - explicit `work_dir=` arg wins
   - else `os.environ.get("OPENCLAW_WORK_DIR")` (as `Path(...).resolve()`)
   - else `Path("./.openclaw-tmp").resolve()` (creates if missing)
2. Remove the now-unused `import tempfile` at `bridge.py:16` (ruff will flag it anyway).
3. Update `examples/openclaw_demo.py` (Task 2) to rely on the default `work_dir` behavior.
4. Update CI `openclaw-smoke` Docker run command to mount `$PWD/.openclaw-tmp` to the identical absolute path inside the container so host and container see the same absolute frame paths.
5. Add `./.openclaw-tmp/` to `.gitignore`.

### Task 2: Create `examples/openclaw_demo.py`

**New file:** `examples/openclaw_demo.py`

A minimal standalone demo:
- Scene: `FloorPlan201`
- Agents: 2
- Steps: 20 (fast enough for CI, enough to show motion)
- Uses `MultiAgentEngine`, `OpenClawProvider`, `ReplayRecorder`, `GameVisualizer`
- Simple behavior: agents take turns, each step queries the Gateway for an action
- Outputs: `replay.gif`, `report.html` to `output/openclaw-demo/`
- CLI args: `--scene`, `--agents`, `--steps`, `--output-dir`, `--gateway-url`, `--token`

No game logic (territory/coverage) — just pure navigation so the demo is focused on "can OpenClaw control the robot?"

**Session + work_dir hygiene (from review A1 + A2):**
- On startup, `shutil.rmtree(work_dir, ignore_errors=True)` then recreate — prevents stale container-UID-owned files from a prior run blocking host writes.
- Construct `OpenClawProvider(session_prefix=f"roboclaws-demo-{int(time.time())}")` so each invocation gets a unique Gateway session. Avoids MEMORY leaking across runs when iterating against the same long-running local Gateway container.
- Log the resolved `work_dir` and `session_prefix` on start so users can correlate logs ↔ Gateway state.

### Task 3: Local OpenClaw quick-start guide

**New file:** `docs/openclaw-local.md`

Contents:
- Prerequisites: Docker, `KIMI_API_KEY` or Gateway already configured
- One-liner Docker run with the bind mount (pinned image tag — A3 decision):
  ```bash
  # Generate a local Gateway token (dev-only is fine for localhost)
  GATEWAY_TOKEN=$(python -c 'import secrets; print(secrets.token_urlsafe(32))')

  docker run -d --name openclaw-gateway \
    -p 127.0.0.1:18789:18789 \
    -v "$PWD/skills/ai2thor-navigator:/home/node/.openclaw/workspace/skills/ai2thor-navigator:ro" \
    -v "$PWD/.openclaw-tmp:$PWD/.openclaw-tmp" \
    -e OPENCLAW_AUTH_MODE=token \
    -e OPENCLAW_AUTH_TOKEN="$GATEWAY_TOKEN" \
    -e OPENCLAW_ALLOWED_TOOLS=ai2thor-navigator \
    ghcr.io/openclaw/openclaw:2026.4.14
  ```
  > ⚠️ Use `127.0.0.1:18789:18789` (not `:18789`) so the port is localhost-only. If you ever expose the Gateway port externally, rotate `OPENCLAW_AUTH_TOKEN` with a fresh `secrets.token_urlsafe(32)` value.
- How to run the demo: `OPENCLAW_GATEWAY_TOKEN=$GATEWAY_TOKEN python examples/openclaw_demo.py --steps 20`
- How to verify readiness: `curl -H "Authorization: Bearer $GATEWAY_TOKEN" http://localhost:18789/readyz`
- **Troubleshooting:**
  - *"Gateway cannot read frame paths"* → check bind mount uses identical absolute paths on host and container.
  - *"file not found" on macOS Docker Desktop* → `$PWD` must be inside Docker Desktop's File Sharing list (Settings → Resources → File Sharing). If `$PWD` is under `/Users/…`, it's shared by default; if you clone to `/opt/...` or a mapped network drive, add it explicitly.
  - *"permission denied writing to .openclaw-tmp"* → files left by a prior run are owned by the container UID. Either `sudo rm -rf .openclaw-tmp` or let the demo clean it (Task 2 adds `shutil.rmtree` on start).

**Update `CLAUDE.md`:** Add cross-reference under "Cloud vs local development".

### Task 4: Replace `openclaw-smoke` CI job

**File:** `.github/workflows/ci.yml`

1. Replace the existing `openclaw-smoke` job with a simpler one that:
   - Pins the image to `ghcr.io/openclaw/openclaw:2026.4.14` (A3 decision — no more `:latest` drift). Keep the `OPENCLAW_IMAGE` env override for forks.
   - Starts the Gateway with the skill mount + `.openclaw-tmp` bind mount at identical absolute paths.
   - Runs `python examples/openclaw_demo.py --agents 2 --steps 10 --output-dir output/openclaw/demo`
   - Runs `python -m roboclaws.core.reporter output/openclaw/demo`
   - Uploads `report-openclaw` artifact
2. Keep `continue-on-error: true` at the job level (OpenClaw Gateway availability is still optional for forks).
3. Update `publish-pages` to copy `openclaw-src/demo` to `site/openclaw/demo/` and update `scripts/write_pages_index.py` flags/landing copy so the index links to the nav demo (not the old territory/coverage placeholders).

### Task 5: Verify report publishing end-to-end

**Validation (concrete — CQ4 decision):**
1. After local validation succeeds, merge to `main` and wait for `openclaw-smoke` + `publish-pages` to go green.
2. Run these checks (each must return `HTTP/2 200`):
   ```bash
   curl -sI https://miaodx.github.io/roboclaws/openclaw/demo/report.html | head -1
   curl -sI https://miaodx.github.io/roboclaws/openclaw/demo/replay.gif  | head -1
   ```
3. Open `report.html` in a browser and confirm: step slider advances, per-agent FPV switches, overhead map renders, VLM reasoning pane shows non-empty entries.
4. Record the digest of the OpenClaw Gateway image that produced the green run (`docker inspect ghcr.io/openclaw/openclaw:2026.4.14 --format '{{index .RepoDigests 0}}'`) in the merge commit message so the pin is reproducible.

### Task 6: Rewrite README Layer 3 and mark Phase 2 complete

**Update `README.md`:**
- Change `- [ ] **Phase 2**: OpenClaw integration` to `- [x] **Phase 2**: OpenClaw integration`
- **Rewrite** the "OpenClaw + Kimi" Layer 3 section (currently lines ~47-52, promising territory + coverage GIFs). Replace with an honest "OpenClaw navigation demo" block (A5 decision):
  - One heading: *"3. OpenClaw + Kimi — push to `main`"*
  - One paragraph: the demo routes one (or more) AI2-THOR agent(s) through a local OpenClaw Gateway + Kimi VLM. Pure navigation, 20 steps, proves the transport end-to-end.
  - One GIF (embedded from Pages): `https://miaodx.github.io/roboclaws/openclaw/demo/replay.gif`
  - One interactive link: `[▶ Interactive report](https://miaodx.github.io/roboclaws/openclaw/demo/report.html)`
  - Retire issue #39 reference (demo is shipping); leave #40 (Railway) closed/superseded since Task 7 drops the Railway job.
  - Remove the territory/coverage OpenClaw GIF references entirely — they return in a later phase with long-running Gateway instances.

### Task 7: Remove dead `--backend openclaw` paths (from review Step 0)

**Files:** `examples/territory_game.py`, `examples/coverage_game.py`, `.github/workflows/ci.yml`, `tests/test_territory_example.py`, `tests/test_coverage_example.py`

After Task 2 ships the new demo, the old `--backend openclaw` wiring is orphaned:
- `examples/territory_game.py:42` imports `OpenClawProvider`; `:107-110` exposes the `--backend` CLI flag; `:164` instantiates `OpenClawProvider()` with no args. Broken and unused.
- `examples/coverage_game.py:43, 108-110, 211` — identical pattern.
- `.github/workflows/ci.yml:178-272` — the whole `openclaw-railway-smoke` job invokes `territory_game.py --backend openclaw` against Railway. Host/container filesystem sharing doesn't apply to a remote Gateway, so this path can never work with the new contract.

Do:
1. Delete the `--backend` argparse flag and related branching from both example scripts.
2. Remove the `from roboclaws.openclaw.bridge import OpenClawProvider` imports from both examples.
3. Remove the entire `openclaw-railway-smoke` job from `.github/workflows/ci.yml` (plus the `needs:` dependency references).
4. Scrub any lingering test references to `--backend openclaw` in `tests/test_{territory,coverage}_example.py`.
5. Update `docs/issues-roadmap.md` Phase 2 section to reflect the new shipping path.

## Test Plan

| Codepath | Test | Location |
|----------|------|----------|
| Bridge work_dir — env var fallback (NEW Task 1) | `test_provider_uses_env_work_dir` | `tests/test_bridge.py` (new test) |
| Bridge work_dir — default to ./.openclaw-tmp (REGRESSION, IRON RULE) | `test_provider_defaults_to_dot_openclaw_tmp` | `tests/test_bridge.py` (new test) |
| Bridge work_dir — precedence: explicit > env > default (T1) | `test_provider_explicit_work_dir_overrides_env` | `tests/test_bridge.py` (new test) |
| Bridge HTTP errors | `test_bridge_step_*` (existing 9 tests) | `tests/test_bridge.py` |
| Demo script lints | `ruff check examples/openclaw_demo.py` | CI lint-and-mock |
| Demo end-to-end | `openclaw-smoke` CI job runs demo → asserts `report-openclaw` artifact non-empty | `.github/workflows/ci.yml` |
| CI Gateway startup + bind mount | `openclaw-smoke` job `/readyz` probe + demo completion | `.github/workflows/ci.yml` |
| Report publishing reachable | Task 5 curl checks post-merge | Manual + merge-commit record |
| Old `--backend openclaw` paths gone (Task 7) | `grep -r "backend openclaw" --include="*.py" --include="*.yml"` returns nothing | CI lint-and-mock (add a grep guard) |

## Error & Rescue Registry

| Error | Cause | Rescue |
|-------|-------|--------|
| `Gateway did not become ready within 60s` | Docker image path/env drift | Read logs, check bind mounts |
| `OpenClawUnavailable: Gateway unreachable` | Local Gateway not running | Point user to `docs/openclaw-local.md` |
| `Gateway cannot read frame paths` | Missing `.openclaw-tmp` bind mount or path mismatch | Check `-v "$PWD/.openclaw-tmp:$PWD/.openclaw-tmp"` |

## Failure Modes

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| OpenClaw upstream changes image contract | CI breaks | Pinned to `ghcr.io/openclaw/openclaw:2026.4.14` (A3); record digest in Task 5 |
| Session-key collision across local runs | MEMORY leaks between demo invocations | `session_prefix=f"roboclaws-demo-{int(time.time())}"` in Task 2 (A1) |
| Stale `.openclaw-tmp` from prior run blocks host writes | PermissionError on JPEG save | `shutil.rmtree(work_dir, ignore_errors=True)` on demo startup (A2) |
| `$PWD` outside Docker Desktop File Sharing (macOS) | Gateway reports "file not found" | Troubleshooting note in `docs/openclaw-local.md` (A4 / Task 3) |
| Local developer lacks Docker | Can't run Gateway | Document that layers 1-2 work without it |
| `openclaw-smoke` job fails | Pages section omits OpenClaw card | `continue-on-error: true` + best-effort artifact download keeps landing page alive |

## Effort Estimate

| Task | Human | CC+gstack |
|------|-------|-----------|
| Task 1 (transport fix + 3 unit tests) | ~45 min | ~5 min |
| Task 2 (demo script + session/work_dir hygiene) | ~1.5 h | ~15 min |
| Task 3 (docs incl. troubleshooting) | ~30 min | ~5 min |
| Task 4 (CI update, pin image) | ~30 min | ~5 min |
| Task 5 (post-merge verify + record digest) | ~15 min | ~5 min |
| Task 6 (README Layer 3 rewrite) | ~20 min | ~3 min |
| Task 7 (delete orphans + scrub tests) | ~30 min | ~5 min |

**Total: ~4.5 h human / ~45 min CC+gstack** (plus local validation time)

## What already exists (reuse, don't rebuild)

- `OpenClawProvider` + `OpenClawBridge` (`roboclaws/openclaw/bridge.py`) — Task 1 only tweaks the `work_dir` fallback.
- `tests/test_bridge.py` — 27 existing tests cover bridge construction, healthcheck, step errors, provider image writes. Plan adds 3 new unit tests.
- `tests/test_skill.py` — 21 existing tests cover `AI2THORNavigatorSkill`. Not modified.
- `roboclaws/core/{engine,replay,visualizer,reporter}.py` — the demo imports all four; no changes.
- CI `openclaw-smoke` (`.github/workflows/ci.yml:288-446`) — Task 4 keeps the Gateway startup / token / readiness steps; only the game steps change.
- `publish-pages` job — Task 4 copies to `site/openclaw/demo/` reusing existing artifact flow.

## Worktree parallelization strategy

| Step | Modules touched | Depends on |
|------|----------------|------------|
| Task 1 (bridge + gitignore + 3 tests) | `roboclaws/openclaw/`, `tests/` | — |
| Task 2 (new demo script) | `examples/` | Task 1 (wants resolved work_dir) |
| Task 3 (local quickstart docs) | `docs/`, `CLAUDE.md` | — |
| Task 4 (CI update) | `.github/workflows/`, `scripts/write_pages_index.py` | Task 2 (must call real demo) |
| Task 5 (post-merge verification) | — | Tasks 2 + 4 + 6 merged |
| Task 6 (README rewrite) | `README.md` | Task 2 (demo exists) |
| Task 7 (delete orphans) | `examples/`, `.github/workflows/`, `tests/` | — |

**Parallel lanes:**
- **Lane A:** Task 1 → Task 2 → Task 6 (bridge + demo + readme, sequential — shared concern: demo behavior)
- **Lane B:** Task 3 (docs — independent)
- **Lane C:** Task 7 (orphan cleanup — independent, touches `examples/` + `ci.yml` but at different lines than Lane A/D)
- **Lane D:** Task 4 (CI) — depends on Lane A's Task 2 landing first. Touches `ci.yml` like Lane C → merge Lane C before Lane D to avoid conflicts.
- **Lane E:** Task 5 (post-merge) — runs after everything merges.

**Execution order:** Launch A + B + C in parallel worktrees. Merge C. Launch D (waits for A's Task 2). Merge A + B + D. Run E.

**Conflict flag:** Lanes C and D both touch `.github/workflows/ci.yml`. Run C (deletes `openclaw-railway-smoke`) before D (rewrites `openclaw-smoke` in the same file) so D merges cleanly.

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 0 | — | — |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | — | — |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 1 | CLEAR (PLAN) | 7 issues resolved, 0 critical gaps |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | — | — |
| DX Review | `/plan-devex-review` | Developer experience gaps | 0 | — | — |

**Eng Review — 7 resolved decisions + 2 auto-fixes + 1 regression test:**

- **Step 0 scope** — delete orphaned `--backend openclaw` paths + drop `openclaw-railway-smoke` CI job → new **Task 7** added to plan.
- **A1 session-key collision** — `session_prefix=f"roboclaws-demo-{int(time.time())}"` unique per invocation → folded into **Task 2**.
- **A3 image drift** — pin `ghcr.io/openclaw/openclaw:2026.4.14` (user-provided) instead of `:latest` → folded into **Task 4**.
- **A5 README misrepresentation** — rewrite Layer 3 section as "OpenClaw navigation demo" rather than just flipping the checkbox → folded into **Task 6**.
- **T1 precedence test** — add `test_provider_explicit_work_dir_overrides_env` for the 3-way work_dir fallback → added to Test Plan.
- **A2 stale work_dir** (auto-fix) — `shutil.rmtree(work_dir, ignore_errors=True)` on demo startup → folded into Task 2.
- **A4 macOS Docker File Sharing** (auto-fix) — troubleshooting block added to `docs/openclaw-local.md` → folded into Task 3.
- **CQ1 dev-token doc hygiene** (auto-fix) — replace hardcoded `dev-token` with a generated `secrets.token_urlsafe(32)` + localhost-only `-p 127.0.0.1:18789:18789` → folded into Task 3.
- **CQ4 vague verification** (auto-fix) — Task 5 rewritten with concrete `curl -sI` checks + image digest recording.
- **REGRESSION (IRON RULE)** — default `work_dir` fallback changes from `tempfile.mkdtemp` to `./.openclaw-tmp`. Auto-added `test_provider_defaults_to_dot_openclaw_tmp` to Test Plan without AskUserQuestion.

**UNRESOLVED:** 0

**VERDICT:** ENG CLEARED — plan is ready to implement. Two TODOs captured in `TODOS.md` (Phase 2.5 game restore + digest pinning).

---

# Phase 2.1 Amendment: Transport Correction

> ⚠️ Added 2026-04-16 after end-to-end validation of Tasks 1-7 surfaced an architectural error in the original plan. Tasks 1-7 are structurally correct (commits land clean, tests pass, lint passes) but all target the **wrong Gateway endpoint**. This amendment fixes the transport and re-validates live.

## Problem Statement

The original plan built `OpenClawBridge` against `POST /tools/invoke`, expecting a workspace skill (SKILL.md in `skills/ai2thor-navigator/`) to become an HTTP-invocable tool. That expectation is wrong for the pinned `ghcr.io/openclaw/openclaw:2026.4.14` image (and, as far as I can tell from the docs + source read, for every current OpenClaw version):

- `/tools/invoke` dispatches **only plugin-registered tools** (`acpx`, `browser`, `device-pair`, `phone-control`, `talk-voice`) via `api.registerTool()` — confirmed in `/app/dist/tools-invoke-http-*.js:135-145`.
- Workspace skills are **prompt-injection hints** consumed by the Gateway's LLM agent — confirmed in `/app/dist/skills-*.js:640` (`loadWorkspaceSkills`) which injects skills into the system prompt only.
- End-to-end validation with real Kimi + real AI2-THOR against the pinned image returns `404 Tool not available: ai2thor-navigator` on the first `/tools/invoke` call.

The correct transport is `POST /v1/chat/completions` — the Gateway's OpenAI-compatible chat endpoint. The Gateway agent's system prompt already contains every workspace skill (measured: 14,044 prompt tokens on a one-word user message with just our one skill mounted). A short per-turn user message steers the agent to follow `ai2thor-navigator` and reply in the skill's JSON shape.

Sources: [OpenClaw Skills docs](https://docs.openclaw.ai/tools/skills), [OpenAI Chat Completions HTTP API](https://docs.openclaw.ai/gateway/openai-http-api.md), [Tools Invoke HTTP API](https://docs.openclaw.ai/gateway/tools-invoke-http-api.md), Gateway source `/app/dist/` in image `sha256:7ea070b04d1e70811fe8ba15feaad5890b1646021b24e00f4795bd4587a594ed`.

## Revised Architecture — named-agent routing

```
┌────────────────────────────────────────────────────────────────┐
│                      AI2-THOR Engine                           │
│                                                                │
│   sim agent 0 frame ──►  OpenClawProvider                      │
│   sim agent 1 frame ──►     │                                  │
│   sim agent N frame ──►     │  JPEG → base64 data URL          │
│                             ▼                                  │
│         POST /v1/chat/completions                              │
│           ├─► agent 0: model="openclaw/agent-0"                │
│           ├─► agent 1: model="openclaw/agent-1"                │
│           └─► agent N: model="openclaw/agent-N"                │
│                                                                │
│           body: OpenAI messages[] with:                        │
│             • text steer ("follow the ai2thor-navigator        │
│               skill, reply JSON only")                         │
│             • image_url (FPV as data URL)                      │
│             • image_url (overhead as data URL)                 │
│             • structured state (JSON text)                     │
│                                                                │
│ ┌────────────── Gateway: one process, N agents ─────────────┐  │
│ │                                                           │  │
│ │  agent-0               agent-1              agent-N       │  │
│ │   workspace/            workspace/           workspace/   │  │
│ │   SOUL.md               SOUL.md              SOUL.md      │  │
│ │   MEMORY.md             MEMORY.md            MEMORY.md    │  │
│ │   auth-profiles.json    ...                  ...          │  │
│ │   ai2thor-navigator     ai2thor-navigator    ai2thor-nav. │  │
│ │   (skill in prompt)     (skill in prompt)    (skill …)    │  │
│ │                                                           │  │
│ └───────────────────────────────────────────────────────────┘  │
│                                                                │
│  Each agent: independent memory, independent persona slot,     │
│  independent auth — matches SKILL.md's "each simulation agent  │
│  runs as a separate OpenClaw instance" promise.                │
└────────────────────────────────────────────────────────────────┘

No bind mount. No .openclaw-tmp. Images flow inline.
```

## Why named agents, not session headers

Both `model: "openclaw" + x-openclaw-session-key: <key>` and `model: "openclaw/<agentId>"` would work for the transport itself. We pick **named agents** because:

- `skills/ai2thor-navigator/SKILL.md` explicitly promises "each simulation agent runs as a separate OpenClaw instance, each with its own SOUL preset and independent memory". The session-header approach shares memory + persona across all sim agents; the named-agent approach gives each one its own.
- Mirrors the existing per-persona OpenClaw pattern already in use on this machine (`openclaw-alice`, `-bob`, `-carol` containers) — one Gateway process, many logical agents, same mental model.
- Trivially enables Phase 2.2 (per-agent SOUL presets — aggressive/defensive/cooperative): each agent already has its own workspace dir to drop a persona into.
- Model-id parser: `/app/dist/http-utils-*.js:resolveAgentIdFromModel` accepts `openclaw/<agentId>` where `agentId` matches `[a-z0-9][a-z0-9_-]{0,63}`. Missing / unknown agent → 4xx from the chat endpoint, caught fast.

Decision: bootstrap **always pre-creates N agents** (parameterized by `AGENTS` env var, default 2). Demo never calls `openclaw agents add` itself — single source of truth for agent configuration.

## Pure wins from the transport pivot

1. **No bind mount required** → A4 macOS File Sharing trap is gone.
2. **No container-UID-owned stale files** → A2 `shutil.rmtree` hack is gone.
3. **No host/container path identity** → whole class of drift errors gone.
4. **Railway / remote Gateways become viable again** → cloud relay (Issue 13) is no longer fundamentally blocked; just currently deferred.
5. **Per-agent independent memory** → closes the SKILL.md honesty gap.

## What the Gateway needs before it can answer

End-to-end validation uncovered six first-run setup steps the original plan missed. All are one-shot + idempotent — perfect for a bootstrap script:

1. **Chown the config volume + pre-create writable workspace dirs** — Docker creates `/home/node/.openclaw` and `workspace/` as root because of the skill bind mount's parent dirs, blocking the `node` user (uid 1000) from writing state files (`workspace/AGENTS.md`, etc.). Fix: in a one-shot `--user root` container before first startup, `mkdir -p` every dir the Gateway + each agent will need, then `chown -R 1000:1000` the whole tree.
2. **Enable chatCompletions** — `gateway.http.endpoints.chatCompletions.enabled = true` (default: false) in `openclaw.json`.
3. **Register N named agents + seed per-agent state** — for i in 0..N-1:
   - Add agent to `openclaw.json` `agents` map.
   - Create workspace dir `~/.openclaw/workspaces/agent-i/` (with the `ai2thor-navigator` skill bind-mounted).
   - Create agent dir `~/.openclaw/agents/agent-i/agent/` with its own `auth-profiles.json` carrying the Kimi `api_key`.
4. **Pin default model per agent** — each agent entry needs `model.primary = $MODEL` (e.g., `kimi/k2p5`).
5. **SKILL.md frontmatter** — every skill needs YAML frontmatter (`name`, `description`) or the loader drops it. Already fixed in the prep commit (5c812a1).
6. **Use `model: "openclaw/agent-<i>"` in requests** — the `/v1/chat/completions` endpoint rejects raw provider model ids with `"Use openclaw or openclaw/<agentId>"`. agentId regex: `[a-z0-9][a-z0-9_-]{0,63}` (lowercase, alphanumeric + `_-`).

## Scope

### In scope
- `scripts/openclaw-bootstrap.sh` — idempotent first-run setup: volume chown, seed `openclaw.json`, seed `auth-profiles.json`, start container, wait for `/readyz`, print token. Used by both `docs/openclaw-local.md` and CI.
- Rewrite `OpenClawBridge.step()` to `POST /v1/chat/completions` with OpenAI multimodal format.
- Simplify `OpenClawProvider.__init__` — drop `work_dir`, drop JPEG-on-disk flow, drop `.openclaw-tmp` handling.
- Rewrite `examples/openclaw_demo.py` to use the new bridge surface (feeds numpy frames directly, not base64 strings), drop `shutil.rmtree` startup.
- Live re-validation: run the demo end-to-end locally, capture `output/openclaw-demo/report.html` + `replay.gif`, commit the artefacts path in a demo-run log so Task 5 (post-merge verification) can diff against it.
- Rewrite `docs/openclaw-local.md` — `bootstrap.sh` one-liner, no bind mount, drop the stale-files + macOS troubleshooting (they can't happen anymore).
- Rewrite `.github/workflows/ci.yml` `openclaw-smoke` job to call `bootstrap.sh`.
- Retire `OPENCLAW_WORK_DIR` env var and the `./.openclaw-tmp` `.gitignore` entry (or keep `.gitignore` line since it's harmless — decide inside T10).
- Rewrite `tests/test_bridge.py` — existing tests assert the `/tools/invoke` contract and the work_dir fallback. Both are gone.

### Not in scope
- **Per-agent SOUL preset distribution** — bootstrap creates N agents with identical default SOUL. Distributing `skills/ai2thor-navigator/souls/{aggressive,defensive,cooperative}.md` to the corresponding agent workspaces is **Phase 2.2**. The architecture supports it trivially (each agent has its own workspace); we just aren't wiring it up here. File: add `Phase 2.2 — per-agent SOUL presets` to `TODOS.md` at Task 16 close.
- Territory / Coverage game modes over OpenClaw (same as before, later phase).
- Isaac Lab migration (Phase 3).
- Full remote/Railway support — no longer architecturally blocked, but out of this amendment's scope.

## Implementation Plan

### Task 8: `scripts/openclaw-bootstrap.sh`

**New file:** `scripts/openclaw-bootstrap.sh` (already landed in prep commit 5c812a1 — this task extends it to N agents).

Shape (env-driven, all overridable):

```bash
#!/usr/bin/env bash
# Idempotent first-run setup for a local or CI OpenClaw Gateway container.
# Uses: docker, python3 (for token generation + JSON reading).
#
#   AGENTS=2 \
#   AGENT_PREFIX=agent- \
#   CONTAINER=openclaw-gateway \
#   IMAGE=ghcr.io/openclaw/openclaw:2026.4.14 \
#   VOLUME=openclaw-demo-config \
#   PORT=18789 \
#   MODEL=kimi/k2p5 \
#   KIMI_API_KEY=sk-... \
#   ./scripts/openclaw-bootstrap.sh
#
# Outputs (stdout): the Gateway bearer token on a single line so callers can
#   `TOKEN=$(./scripts/openclaw-bootstrap.sh)` in both shells and GitHub Actions.
```

Behaviour:
1. If `CONTAINER` already exists, remove it (clean slate).
2. `docker volume create $VOLUME` (idempotent).
3. One-shot `docker run --rm --user root -v $VOLUME:/home/node/.openclaw $IMAGE` to:
   - `mkdir -p` every dir the Gateway + all N agents will need — including per-agent workspace + per-agent `agents/<id>/agent/` — *before* the real container starts (so Docker doesn't create intermediate dirs as root).
   - Seed `openclaw.json` with:
     - `gateway.auth.mode = "token"`
     - `gateway.http.endpoints.chatCompletions.enabled = true`
     - `agents` map: for each `agent-i` (i=0..AGENTS-1), an entry with `{ workspace: "/home/node/.openclaw/workspaces/agent-i", model: { primary: $MODEL } }`.
     - `agents.defaults.model.primary = $MODEL` so the default agent ("main") also has a valid model — harmless and avoids any "no default" edge case.
   - For each `agent-i`, seed `/home/node/.openclaw/agents/agent-i/agent/auth-profiles.json` with `{"profiles": {"kimi:manual": {"type": "api_key", "provider": "kimi", "key": "$KIMI_API_KEY"}}}` (schema per `/app/dist/store-*.js:parseCredentialEntry`).
   - For each `agent-i`, symlink or copy the `ai2thor-navigator` skill into `/home/node/.openclaw/workspaces/agent-i/skills/ai2thor-navigator`. Actual wiring: one read-only host-side bind mount per agent in step 4 (simpler than copying inside the volume).
   - `chown -R 1000:1000 /home/node/.openclaw`.
4. `docker run -d` the Gateway with:
   - Config volume mount at `/home/node/.openclaw`.
   - One `--mount type=bind,source=$SKILLS_DIR,target=/home/node/.openclaw/workspaces/agent-i/skills/ai2thor-navigator,readonly=true` per agent (a loop builds the `-v` flags). This is more bind mounts but each agent's skill catalog is independent and we can vary skills per agent in Phase 2.2 without rewiring.
5. Wait up to 60s for `/readyz` with bearer auth.
6. `openclaw.json` seeding already pinned the default model per agent; verify via `python3 -c "import json; …"` rather than calling the CLI a second time.
7. Probe `POST /v1/chat/completions` with `model="openclaw/agent-0"` to confirm the full chain (auth profile, skill mount, chat endpoint) works before the demo starts. Fail fast if the first agent doesn't PONG back.
8. Read the live token from `openclaw.json` (the Gateway regenerates the auth token on first boot regardless of env) and echo it on the last stdout line.

Failure modes:
- Missing `KIMI_API_KEY` → exit 1 with a usage hint.
- `AGENTS < 1` or `AGENTS > 8` → exit 1 ("out of supported range; file a Phase 2.x issue if you need more").
- Docker pull fails → exit 2 with the real error.
- `/readyz` never returns 200 within 60s → `docker logs` + exit 3.
- Probe fails → exit 4 with the raw probe body + `docker logs` tail (so the user sees the real Kimi/auth/skill error, not a generic bootstrap failure).

### Task 9: Rewrite `OpenClawBridge.step()` for `/v1/chat/completions` with named agents

**File:** `roboclaws/openclaw/bridge.py`

New `step()` signature (drops `frame_path` / `overhead_path` — they're now inline):

```python
def step(
    self,
    agent_id: int,
    frame: np.ndarray,            # first-person RGB (H, W, 3) uint8
    overhead: np.ndarray,         # overhead map RGB (H, W, 3) uint8
    state: dict[str, Any],
    step_idx: int,
) -> dict[str, Any]:
    """POST a turn to /v1/chat/completions for agent-<agent_id> and parse the action out."""
```

`OpenClawBridge.__init__` gains `agent_prefix: str = "agent-"` so the bridge can compose `model = f"openclaw/{agent_prefix}{agent_id}"` without the caller repeating the prefix. Matches the bootstrap's `AGENT_PREFIX` env var — single source of truth for the naming scheme.

Body:
1. JPEG-encode `frame` and `overhead` → base64 data URLs via `Image.fromarray(...).save(buf, "JPEG", quality=80)`.
2. Build OpenAI messages with a short user steer:
   ```
   user:
     text: "You are RoboClaws {agent_prefix}{agent_id}, step {step_idx}/{max}.
            Follow the ai2thor-navigator skill. Current state (JSON):
            {state_json}. FPV and overhead map attached.
            Reply with ONLY JSON: {\"reasoning\": ..., \"action\": ...}."
     image_url: data:image/jpeg;base64,<fpv>
     image_url: data:image/jpeg;base64,<overhead>
   ```
3. POST with:
   - `Authorization: Bearer <token>`
   - `Content-Type: application/json`
   - body `model = f"openclaw/{agent_prefix}{agent_id}"` (routes to the named agent → its own SOUL / MEMORY / auth)
   - NO `x-openclaw-session-key` header — the named-agent model provides isolation already.
4. Parse `choices[0].message.content`, strip any code fences, `json.loads` → `{"reasoning", "action"}`.
5. Validate `action ∈ NAVIGATION_ACTIONS`, fall back to `MoveAhead` with a visible warning if malformed.
6. Error mapping:
   - `ConnectError` / `ReadTimeout` → `OpenClawUnavailable` (unchanged).
   - HTTP 401 → `OpenClawUnavailable("Gateway rejected bearer token")`.
   - HTTP 400 `"Invalid model"` → `OpenClawUnavailable("agent 'agent-{N}' not registered — run scripts/openclaw-bootstrap.sh with AGENTS>=N+1")`.
   - HTTP 404 → `OpenClawUnavailable("/v1/chat/completions not enabled — re-run scripts/openclaw-bootstrap.sh")`.
   - HTTP 5xx + `{"error": ...}` → `OpenClawUnavailable(body["error"]["message"])`.
   - Non-JSON response from LLM → log full content, return fallback action.

### Task 10: Simplify `OpenClawProvider`

**File:** `roboclaws/openclaw/bridge.py`

- Drop `work_dir` parameter, `OPENCLAW_WORK_DIR` env handling, `_write_image`, the entire `.openclaw-tmp` directory creation.
- Drop `session_prefix` — replaced by `agent_prefix` (see Task 9). One less magic string; matches bootstrap's `AGENT_PREFIX`.
- `get_action(images: list[str], state: dict)` → change signature to accept numpy arrays directly. The base64 intermediate is wasteful given the VLM provider protocol originally used it for the OpenAI providers; for OpenClaw we skip straight to bytes. Grep confirms only caller is `examples/openclaw_demo.py`, updated atomically in Task 11.

### Task 11: Rewrite `examples/openclaw_demo.py`

- Drop `_prepare_work_dir()` (no longer needed — no filesystem exchange).
- Drop `shutil.rmtree` + `OPENCLAW_WORK_DIR` plumbing.
- `OpenClawProvider` gets constructed with just `gateway_url`, `token`, `agent_prefix` — no `work_dir`, no `session_prefix`.
- Add `--agent-prefix` CLI flag (default: `"agent-"`) mirroring the bootstrap's `AGENT_PREFIX`.
- **Precondition check**: before starting the run, demo probes the first agent (`model="openclaw/agent-0"`) with a one-turn PONG to confirm the named agent exists. Fail-fast with an actionable message pointing to `scripts/openclaw-bootstrap.sh AGENTS=$AGENTS` if it doesn't. Avoids the demo running 19 steps before hitting "agent not registered".
- On startup, log: "Gateway URL:", "Agent prefix:", "Agents: agent-0, agent-1, ... (model=openclaw/<agentId>)", "Model (resolved by Gateway):" (read from the probe response).
- Shorter main-flow since no bind-mount prep required.

### Task 12: Rewrite `docs/openclaw-local.md`

Target shape (much shorter than current):

```markdown
## 1. Bootstrap the Gateway (creates N named agents)
export KIMI_API_KEY=sk-...
TOKEN=$(AGENTS=2 ./scripts/openclaw-bootstrap.sh)

## 2. Run the demo
OPENCLAW_GATEWAY_TOKEN=$TOKEN python examples/openclaw_demo.py --agents 2 --steps 20

## 3. Clean up
docker rm -f openclaw-gateway
docker volume rm openclaw-gateway-config
```

Plus a short "What the bootstrap actually did" section:
- Created `agent-0`, `agent-1` (or N) named agents with isolated workspaces.
- Each agent has its own `~/.openclaw/agents/<id>/agent/auth-profiles.json` carrying the Kimi key.
- Each agent's workspace has `skills/ai2thor-navigator/` bind-mounted read-only.
- `/v1/chat/completions` is enabled; requests route per-agent via `model=openclaw/agent-<i>`.

Troubleshooting section shrinks to four items:
- `bootstrap.sh` exits 2 on pre-seed → docker daemon + socket perms.
- Gateway returns 401 → token was regenerated; extract with `docker exec openclaw-gateway cat /home/node/.openclaw/openclaw.json | jq -r .gateway.auth.token` or just re-run `bootstrap.sh` and re-capture.
- Gateway returns 400 "Invalid model" on `openclaw/agent-N` → bootstrap didn't create agent-N; re-run with higher `AGENTS=`.
- Gateway returns 404 on `/v1/chat/completions` → bootstrap didn't enable the endpoint; re-run.

Delete: the macOS File Sharing block, the "container-UID-owned stale files" block, the "host+container path identity" block. Those failure modes are gone.

### Task 13: Rewrite CI `openclaw-smoke`

**File:** `.github/workflows/ci.yml`

Replace the current inline Docker run + token-extraction + preflight + skill-mount step with:

```yaml
- name: Bootstrap OpenClaw Gateway (2 named agents)
  id: gateway
  continue-on-error: true
  env:
    KIMI_API_KEY: ${{ secrets.KIMI_API_KEY }}
    AGENTS: "2"
  run: |
    TOKEN=$(./scripts/openclaw-bootstrap.sh)
    echo "::add-mask::$TOKEN"
    echo "token=$TOKEN" >> "$GITHUB_OUTPUT"

- name: Run OpenClaw navigation demo
  if: steps.gateway.outcome == 'success'
  env:
    OPENCLAW_GATEWAY_TOKEN: ${{ steps.gateway.outputs.token }}
  run: |
    xvfb-run -a python examples/openclaw_demo.py \
      --agents 2 --steps 10 \
      --output-dir output/openclaw/demo
```

Drop: the `mkdir -p $PWD/.openclaw-tmp`, the `-v $PWD/.openclaw-tmp:...` bind mount, the `Resolve Gateway token from container config` step (bootstrap does it), the `Wait for Gateway readiness` step (bootstrap does it), the `KIMI_API_KEY` secret from the demo step env (the Gateway has it; the demo doesn't need it).

Keep: `continue-on-error: true` at the job level, image pin (now lives inside bootstrap's `IMAGE` default — job exports it explicitly to keep the pin visible in the workflow), artifact upload, publish-pages wiring (still `site/openclaw/demo/`).

Add: `::add-mask::` on the captured token so it doesn't leak into log output if a later step prints it.

### Task 14: Rewrite `tests/test_bridge.py`

Tests that must be deleted (they test the old contract):
- `test_provider_uses_env_work_dir`
- `test_provider_defaults_to_dot_openclaw_tmp`
- `test_provider_explicit_work_dir_overrides_env`
- Any `test_bridge_step_*` that asserts `/tools/invoke` path, `sessionKey` body field, or the provider-writes-JPEG behaviour.

New tests:
- `test_bridge_step_posts_to_chat_completions` — mock `httpx.Client`, assert URL path `/v1/chat/completions`, `model = "openclaw/agent-0"`, presence of FPV + overhead image_url parts in the OpenAI payload.
- `test_bridge_step_parses_action_from_response` — feed a canned `chat.completion` response with JSON in `choices[0].message.content`, assert the bridge extracts the action.
- `test_bridge_step_fallback_on_malformed_json` — canned non-JSON content → bridge returns `MoveAhead` + logs warning.
- `test_bridge_step_raises_on_400_invalid_model` — canned 400 with "Invalid model" body → `OpenClawUnavailable` with the "run bootstrap with higher AGENTS=" hint.
- `test_bridge_step_raises_on_404` — canned 404 response → `OpenClawUnavailable` pointing at re-running bootstrap.
- `test_bridge_uses_agent_prefix_in_model_id` — construct bridge with `agent_prefix="bot-"`, call `step(agent_id=3, …)`, assert outbound `model == "openclaw/bot-3"`.
- `test_bridge_validates_action_in_navigation_actions` — LLM returns `{"reasoning": "...", "action": "Teleport"}` (valid) → passes through; returns `"WalkIntoWall"` (invalid) → coerced to `MoveAhead`.

Target test count: ~7 bridge tests (replaces the existing 27+3).

### Task 15: Live re-validation + artefact capture

**Validation (concrete — supersedes Task 5):**
1. `set -a && source .env && set +a && TOKEN=$(AGENTS=2 ./scripts/openclaw-bootstrap.sh)` completes cleanly; log shows "agents registered: agent-0, agent-1" + "probe ok".
2. `openclaw/agent-0` and `openclaw/agent-1` both answer a PONG probe independently (sanity-check each named agent exists and has its own auth):
   ```bash
   for a in agent-0 agent-1; do
     curl -s -X POST http://127.0.0.1:18789/v1/chat/completions \
       -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
       -d "{\"model\":\"openclaw/$a\",\"messages\":[{\"role\":\"user\",\"content\":\"PONG\"}]}"
   done
   ```
3. `OPENCLAW_GATEWAY_TOKEN=$TOKEN python examples/openclaw_demo.py --agents 2 --steps 8` runs to completion.
4. `output/openclaw-demo/report.html` is >10 KB and opens in a browser.
5. `output/openclaw-demo/replay.gif` renders per-step transitions with visible agent movement.
6. `output/openclaw-demo/replay.json` — spot-check 3 random steps: each has a non-empty `vlm_response.reasoning` and a valid `action`.
7. Spot-check per-agent isolation: after the run, compare `docker exec openclaw-gateway ls /home/node/.openclaw/workspaces/agent-0/` vs `.../agent-1/`. Either both have independent `MEMORY.md` changes or both are empty (with identical content = failure, because they'd be sharing state).
8. Commit-message capture: image digest (`docker inspect ghcr.io/openclaw/openclaw:2026.4.14 --format '{{index .RepoDigests 0}}'`) + Kimi model actually used (read from `replay.json` `summary.provider_status.model`).
9. After merge + `openclaw-smoke` green, `curl -sI https://miaodx.github.io/roboclaws/openclaw/demo/report.html | head -1` → `HTTP/2 200`.

### Task 16: Retro-update the README / PLAN.md

- No README changes (still points at `/openclaw/demo/` — the URL is unchanged).
- Add a short `## Phase 2.1 retrospective` section at the bottom of PLAN.md naming the bug + the fix (so future plans catch this upstream-contract class of error earlier).
- Update `docs/issues-roadmap.md` Phase 2 section — unblock Issue 13 (cloud relay) since transport is no longer bind-mount-bound.

## Test Plan (additions / replacements)

| Codepath | Test | Location | Status vs original plan |
|----------|------|----------|-------------------------|
| Bridge work_dir fallback (3 tests) | — | — | **DELETED** (T14) — no longer applicable |
| Bridge POSTs to `/v1/chat/completions` | `test_bridge_step_posts_to_chat_completions` | `tests/test_bridge.py` | NEW (T14) |
| Bridge parses action from OpenAI content | `test_bridge_step_parses_action_from_response` | `tests/test_bridge.py` | NEW (T14) |
| Bridge handles malformed LLM JSON | `test_bridge_step_fallback_on_malformed_json` | `tests/test_bridge.py` | NEW (T14) |
| Bridge maps 404 to readable error | `test_bridge_step_raises_on_404` | `tests/test_bridge.py` | NEW (T14) |
| Bridge per-agent session header | `test_bridge_session_header_per_agent` | `tests/test_bridge.py` | NEW (T14) |
| `scripts/openclaw-bootstrap.sh` shellcheck | `shellcheck scripts/openclaw-bootstrap.sh` | `lint-and-mock` CI | NEW (T8) |
| Demo end-to-end (live Gateway) | `openclaw-smoke` job runs `bootstrap.sh` + demo → non-empty `report-openclaw` artifact | `.github/workflows/ci.yml` | CHANGED (T13) |

## Effort Estimate (amendment)

| Task | Human | CC+gstack |
|------|-------|-----------|
| Task 8 (bootstrap script) | ~1 h | ~15 min |
| Task 9 (bridge rewrite) | ~1 h | ~15 min |
| Task 10 (provider cleanup) | ~20 min | ~5 min |
| Task 11 (demo rewrite) | ~20 min | ~5 min |
| Task 12 (docs rewrite) | ~30 min | ~5 min |
| Task 13 (CI rewrite) | ~20 min | ~5 min |
| Task 14 (test rewrite) | ~1 h | ~15 min |
| Task 15 (live re-validation) | ~30 min (one real demo run) | ~10 min |
| Task 16 (retro + roadmap) | ~15 min | ~5 min |

**Total for amendment: ~5 h human / ~1 h 20 min CC+gstack.**

## Failure Modes (amendment)

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Kimi API key invalid | Gateway 500s on every `/v1/chat/completions` | `bootstrap.sh` runs a one-shot `curl` preflight after seed to catch this before demo starts |
| Gateway regenerates auth token on restart | Stale token in CI → 401 | `bootstrap.sh` always reads the live token from `openclaw.json` post-start |
| Kimi changes prompt-token limit and our 14k skill prompt overflows | `chat.completions` 400 | `openclaw models list` reports ctx; fall back to a smaller skill set or a compact skill variant |
| LLM returns "action" outside `NAVIGATION_ACTIONS` | Non-crashing but useless step | Bridge validates + defaults to `MoveAhead`, logs the raw response for debugging (existing behaviour retained) |
| `/v1/chat/completions` rate-limited | Burst of 429s | Reuse `roboclaws/core/provider_retry.py` retry machinery (the VLM providers already use it — wire it into the bridge) |

## Phase 2.1 retrospective

Filed 2026-04-16 after Tasks 8-14 landed and the transport rewrite passed
mock tests + lint. Live re-validation (Task 15) is the separate next
gate — this retrospective captures the *bug class* that motivated the
amendment, so future plans catch it upstream.

- **Bug class**: "we built against an API surface we hadn't proven end-to-end". The 7 original tasks all passed lint + tests + plan review, because the test suite mocked `httpx` responses shaped like the endpoint we *expected* (`/tools/invoke` with `ok/result` body), not the endpoint that actually exists in the pinned image (`/v1/chat/completions` with OpenAI-style `choices[]`). CI's `openclaw-smoke` job was guarded with `continue-on-error: true` and thus silently absorbed the 404 any real run would have produced.
- **Lesson**: any new external HTTP contract must have at least one integration test that hits the *actual* upstream before the plan clears eng-review. Post-merge verification is too late — it should be a pre-merge gate (live-Gateway demo run with a captured `report.html`).
- **Fix for future plans**: when a plan's eng-review marks an external integration as critical, add an "AUTH → READYZ → ONE-REAL-REQUEST" Nyquist gate ahead of any task that builds on it. Bootstrap scripts go *in* the plan, not retrofitted. The Phase 2.1 `scripts/openclaw-bootstrap.sh` ended up doing this — it both sets up the Gateway **and** probes `/v1/chat/completions` on `agent-0` with a PONG turn, failing fast with a pointer to the actual error. That probe belongs in the plan from day 1 for any similar "new upstream HTTP surface" phase.
- **Side benefit**: the named-agent routing (`model=openclaw/<agentId>` instead of a session header) that this amendment adopts also unblocks Issue 13 (remote/Railway Gateway) — the transport is no longer bind-mount-bound. Closing the honesty gap with `SKILL.md`'s per-agent-instance promise and unblocking cloud relay fell out of the same fix.
