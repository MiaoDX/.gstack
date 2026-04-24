# /autoplan Restore Point
Captured: 2026-04-23T07:10:49Z | Branch: dongxu-dev-0423 | Commit: 84850ff

## Re-run Instructions
1. Restore the phase-bundle files from the sections below
2. Remove `.planning/phases/02.8-split-model-vision/02.8-AUTOPLAN-REVIEW.md` if you want a fully pre-review tree
3. Invoke /autoplan on `.planning/phases/02.8-split-model-vision` again

## Original Phase Bundle State

### FILE: .planning/phases/02.8-split-model-vision/02.8-01-profile-probe-and-bridge-contract-spike-PLAN.md

---
phase: 02.8
plan: 01
slug: profile-probe-and-bridge-contract-spike
type: execute
wave: 1
depends_on: []
files_modified:
  - spike/split_model_probe.py
  - .planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md
autonomous: false
requirements_addressed: [A-07]

must_haves:
  truths:
    - "A reusable local-dev probe helper exists under `spike/` for Phase 02.8."
    - "`02.8-SPIKE-FINDINGS.md` records the exact behavior of `minimal`, `messaging`, and any attempted `alsoAllow`-style tool policy experiment against the pinned Gateway image."
    - "`02.8-SPIKE-FINDINGS.md` records at least one real vision-description probe using the candidate intermediary model on both FPV + map images and recommends a concrete text contract for navigation."
    - "The findings doc locks one architecture verdict: `mcp-intercept-primary` or `gateway-image-tool-primary`."
    - "If the loser is rejected, the findings doc names the concrete reason (`image not exposed`, `forbidden tools re-opened`, `tool framing unusable`, `description quality inadequate`, or another explicit cause)."
    - "No bearer token, API key, or raw Authorization header is pasted into repo-tracked findings."
  artifacts:
    - path: "spike/split_model_probe.py"
      provides: "Reusable helper for Phase 02.8 tool-surface and vision-description probing"
      contains: "split-model probe"
    - path: ".planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md"
      provides: "Live evidence and architecture lock for the split-model bridge"
      contains: "mcp-intercept-primary"
  key_links:
    - from: "spike/split_model_probe.py"
      to: "02.8-SPIKE-FINDINGS.md"
      via: "captured raw probe outputs under output/probes/02.8/"
      pattern: "output/probes/02\\.8"
---

<objective>
De-risk split-model navigation before touching production runtime code. Probe two
unknowns that the planner must not hand-wave:

1. Can the pinned Gateway expose the generic `image` tool without reopening the
   forbidden tool surface?
2. What text contract should the vision intermediary return so a text-only main
   model can still navigate?

This plan leaves behind a hard decision record that every later plan must obey.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md

THIS PLAN IS LOCAL-DEV ONLY. It requires Docker, a real `MIMO_TP_KEY`, and a
working local Gateway. A cloud session must not fabricate the evidence.
</execution_context>

<context>
@.planning/phases/02.8-split-model-vision/02.8-CONTEXT.md
@.planning/phases/02.8-split-model-vision/02.8-RESEARCH.md
@docs/openclaw-gateway-internals.md
@docs/openclaw-local.md
@scripts/openclaw-bootstrap.sh
@spike/mcp_image_probe.py
</context>

<tasks>

<task type="checkpoint:human-verify" gate="blocking">
  <name>Task 1: Pre-flight — confirm a real local split-model probe session exists</name>
  <read_first>
    - AGENTS.md § 1 and § 7
    - docs/openclaw-local.md
  </read_first>
  <behavior>
    - Docker is available.
    - A real `MIMO_TP_KEY` is present.
    - No stale `openclaw-gateway` container is occupying the expected ports unless the operator intends to reuse it.
    - This plan does not proceed in a cloud-only session.
  </behavior>
  <action>
    Run and paste the outputs of:

    1. `docker --version`
    2. `[[ -n "$MIMO_TP_KEY" ]] && echo mimo-set || echo mimo-missing`
    3. `curl -sf http://127.0.0.1:18789/readyz || echo not-ready`
    4. `docker ps -a --format '{{.Names}}' | grep -x openclaw-gateway || echo absent`
    5. `python -c "import httpx; print('httpx ok')"`

    If this is a cloud session, stop here and defer.
  </action>
  <verify>
    <automated>MISSING — checkpoint task; operator confirms local readiness</automated>
  </verify>
  <done>Operator confirms a real local Gateway session is available for Phase 02.8 probing.</done>
</task>

<task type="auto">
  <name>Task 2: Add a reusable `spike/split_model_probe.py` helper</name>
  <read_first>
    - spike/mcp_image_probe.py
    - docs/openclaw-gateway-internals.md
    - scripts/openclaw-bootstrap.sh
  </read_first>
  <behavior>
    - The helper is local-dev oriented and intentionally small.
    - It supports two probe families:
      - `tool-surface` — ask a running Gateway agent to enumerate its tools
      - `vision-describe` — call the candidate intermediary model directly on two images and persist the text description
    - The helper writes request/response artifacts under a caller-supplied output dir.
    - The helper never prints secrets and exits non-zero on HTTP failures.
  </behavior>
  <action>
    Create `spike/split_model_probe.py` with these concrete CLI modes:

    1. `--probe tool-surface`
       - inputs: `--gateway-url`, `--token-env`, `--model`, `--output-dir`
       - sends a compact prompt asking the agent to list every tool name it has
       - writes `tool-surface.request.json`, `tool-surface.response.json`, and
         `tool-surface.extracted.txt`

    2. `--probe vision-describe`
       - inputs: `--vision-model`, `--fpv`, `--map`, `--output-dir`
       - calls the candidate bridge model directly and writes
         `vision.request.json`, `vision.response.json`, and
         `vision.extracted.txt`
       - default prompt asks for a navigation-focused description:
         obstacles, free directions, notable landmarks, and what the overhead
         implies

    3. `--probe both`
       - runs both families in one invocation

    Reuse the current repo assumptions:
    - OpenAI-compatible transport
    - bearer token loaded from env
    - model defaults kept short and explicit

    Keep it throwaway and spike-grade: no tests, but `python -m py_compile`
    must succeed.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && python -m py_compile spike/split_model_probe.py && python spike/split_model_probe.py --help | rg "probe|gateway-url|token-env|vision-model|fpv|map|output-dir"</automated>
  </verify>
  <done>`spike/split_model_probe.py` exists, compiles, and exposes the required CLI flags.</done>
</task>

<task type="auto">
  <name>Task 3: Run the probe matrix and write `02.8-SPIKE-FINDINGS.md`</name>
  <read_first>
    - .planning/phases/02.8-split-model-vision/02.8-RESEARCH.md
    - docs/openclaw-gateway-internals.md
    - scripts/openclaw-bootstrap.sh
    - spike/split_model_probe.py
  </read_first>
  <behavior>
    - The findings doc records one real tool-surface probe for `minimal`.
    - The findings doc records one real tool-surface probe for `messaging`.
    - If the operator attempts an `alsoAllow` or custom tools shape, the findings doc records exactly what config was tried and whether the Gateway accepted it.
    - The findings doc records one real vision-description probe using both an FPV frame and a map/support image.
    - The findings doc locks one architecture verdict and explains why the loser lost.
  </behavior>
  <action>
    Run probes against the pinned Gateway image and save raw artifacts under
    `output/probes/02.8/`.

    Minimum matrix:

    1. `profile=minimal` tool-surface probe
    2. `profile=messaging` tool-surface probe
    3. optional `alsoAllow` / custom-tools-shape experiment if the operator can
       stage it safely without shipping the config
    4. one real vision-description probe against the candidate intermediary
       model using two images

    Then write `.planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md`
    with this exact high-level structure:

    1. Frontmatter: phase, date, status, Gateway image, models tested
    2. `## Probe setup`
    3. `## Tool-surface probes`
    4. `## Vision-description probe`
    5. `## Decision`
       - `mcp-intercept-primary`
       - or `gateway-image-tool-primary`
    6. `## Why the loser lost`
    7. `## What Plan 02 must implement`

    Decision rule:

    - Choose `gateway-image-tool-primary` only if the generic `image` tool is
      exposed without any of the forbidden drift tools (`exec`, `read`, `write`,
      `browser`, `web_fetch`, `canvas`, `nodes`, `cron`, `sessions_*`,
      `subagents`) and a live prompt proves the path is usable for two-image
      navigation input.
    - Otherwise choose `mcp-intercept-primary`.
  </action>
  <verify>
    <automated>test -f /home/mi/ws/gogo/roboclaws/.planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md && rg -n "mcp-intercept-primary|gateway-image-tool-primary|Why the loser lost|What Plan 02 must implement" /home/mi/ws/gogo/roboclaws/.planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md</automated>
  </verify>
  <done>`02.8-SPIKE-FINDINGS.md` exists, records the probe matrix, and locks one architecture verdict.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Local probe helper ↔ real Gateway | The helper talks to a bearer-authenticated Gateway and must not leak credentials while capturing evidence. |
| Findings doc ↔ later plans | Plan 02 will treat the spike verdict as authoritative. A vague result would mis-plan the whole phase. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-02.8-01 | Elevation of Privilege | Tool-surface alternative | mitigate | Record the exact tool inventory for each profile/config attempt before choosing any architecture that depends on the generic `image` tool. |
| T-02.8-02 | Information Disclosure | Probe artifacts | mitigate | Persist request/response artifacts without secrets; findings doc refers to env vars literally instead of pasting tokens. |
| T-02.8-03 | Repudiation | Decision record | mitigate | Force a hard verdict plus an explicit loser rationale in `02.8-SPIKE-FINDINGS.md`. |

No `high`-severity threats; plan proceeds.
</threat_model>

<verification>
- `python -m py_compile spike/split_model_probe.py` exits 0.
- `02.8-SPIKE-FINDINGS.md` records exact tool-surface behavior and the bridge decision.
- The chosen architecture is backed by local evidence, not assumptions.
</verification>

<success_criteria>
- The phase no longer depends on an unresolved `image`-tool question.
- Plan 02 receives a locked architecture and a recommended text-description contract.
- The evidence trail is strong enough for a later local session to reproduce the decision.
</success_criteria>


### FILE: .planning/phases/02.8-split-model-vision/02.8-02-mcp-observe-text-bridge-PLAN.md

---
phase: 02.8
plan: 02
slug: mcp-observe-text-bridge
type: execute
wave: 2
depends_on: ["02.8-01"]
files_modified:
  - roboclaws/openclaw/vision_bridge.py
  - roboclaws/openclaw/mcp_server.py
  - skills/ai2thor-navigator/SKILL.md
  - examples/openclaw_nav_autonomous.py
  - tests/test_openclaw_vision_bridge.py
  - tests/test_openclaw_mcp_server.py
  - tests/test_openclaw_nav_autonomous.py
autonomous: true
requirements_addressed: [A-07]

must_haves:
  truths:
    - "When the split-model path is active, `roboclaws__observe` returns text-only content to the agent and does not forward raw image blocks."
    - "Vision-capable paths keep the current raw-image observe contract."
    - "Bridge activation is controllable via explicit config with an `auto` default and an explicit bridge-model override."
    - "Host-side replay/report artifacts keep their existing image capture path; any new trace metadata is additive-only."
    - "The navigator skill and autonomous kickoff prompt are updated in the same plan so the live agent contract matches the tool behavior."
    - "Bridge failures degrade to safe text, not to raw image passthrough."
  artifacts:
    - path: "roboclaws/openclaw/vision_bridge.py"
      provides: "Dedicated image-to-text bridge for split-model observe results"
      contains: "VisionBridgeResult"
    - path: "roboclaws/openclaw/mcp_server.py"
      provides: "Observe delivery-mode switch between raw images and text bridge"
      contains: "observe_delivery"
    - path: "skills/ai2thor-navigator/SKILL.md"
      provides: "Updated tool contract for image-bearing and text-bridge observe modes"
      contains: "observe_delivery"
  key_links:
    - from: "roboclaws/openclaw/vision_bridge.py"
      to: "roboclaws/openclaw/mcp_server.py"
      via: "text description result consumed by `_do_observe()`"
      pattern: "text-bridge"
    - from: "roboclaws/openclaw/mcp_server.py"
      to: "skills/ai2thor-navigator/SKILL.md"
      via: "shared observe metadata naming (`observe_delivery`, `bridge_model`, `image_labels`)"
      pattern: "observe_delivery"
---

<objective>
Implement the split-model bridge at the MCP-server seam. After this plan, the
shipped `observe` tool can deliver either raw images or a text description,
while the host still preserves image-bearing artifacts for replay and debugging.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/02.8-split-model-vision/02.8-CONTEXT.md
@.planning/phases/02.8-split-model-vision/02.8-RESEARCH.md
@.planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md
@roboclaws/openclaw/mcp_server.py
@skills/ai2thor-navigator/SKILL.md
@examples/openclaw_nav_autonomous.py
@tests/test_openclaw_mcp_server.py
@tests/test_openclaw_nav_autonomous.py
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Add a dedicated vision-bridge helper with mockable tests</name>
  <read_first>
    - roboclaws/core/vlm.py (MiMo/NVIDIA client patterns)
    - roboclaws/openclaw/mcp_server.py
    - .planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md
  </read_first>
  <behavior>
    - The bridge is a small helper dedicated to image-to-text description, not a reuse of `get_action()`.
    - The bridge resolves its model from explicit config first, then the already-seeded image model.
    - The bridge prompt is optimized for navigation: obstacles, free paths, headings, and salient landmarks.
    - Failures return a safe text fallback plus metadata; they do not silently expose raw images.
  </behavior>
  <action>
    Create `roboclaws/openclaw/vision_bridge.py` with a tiny, mock-friendly API.

    Required contract:

    1. Add a result object, for example:

       ```python
       @dataclass
       class VisionBridgeResult:
           delivery: Literal["images", "text-bridge"]
           description: str
           bridge_model: str | None
           latency_s: float | None
           error: str | None = None
       ```

    2. Add config resolution with explicit override first:
       - `ROBOCLAWS_VISION_BRIDGE_MODEL`
       - fallback `IMAGE_MODEL`

    3. Add an observe-mode selector with three values:
       - `auto`
       - `images`
       - `text-bridge`

    4. The helper sends both FPV and map/support images in one request to the
       intermediary model and asks for a compact navigation description.

    5. On bridge failure, return a safe description such as
       "Vision bridge unavailable; use structured state only" plus error
       metadata.

    6. Add `tests/test_openclaw_vision_bridge.py` covering:
       - config resolution
       - two-image request shape
       - success result normalization
       - safe fallback behavior

    Keep live HTTP out of tests; mock the client layer.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_openclaw_vision_bridge.py -q</automated>
  </verify>
  <done>The bridge helper exists, is mock-tested, and exposes the split-model description contract.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Integrate the bridge into `observe()` and update the agent contract</name>
  <read_first>
    - roboclaws/openclaw/mcp_server.py
    - skills/ai2thor-navigator/SKILL.md
    - examples/openclaw_nav_autonomous.py
    - tests/test_openclaw_mcp_server.py
    - tests/test_openclaw_nav_autonomous.py
  </read_first>
  <behavior>
    - `observe()` can switch between raw-image delivery and text-bridge delivery.
    - The host trace keeps the existing image-bearing `frame_capture` path.
    - The text-bridge mode adds only additive metadata to trace/runtime payloads.
    - The skill and kickoff prompt explicitly tell the agent how to interpret the new observe metadata.
  </behavior>
  <action>
    Modify the runtime surface with these concrete changes:

    1. In `roboclaws/openclaw/mcp_server.py`:
       - resolve observe delivery mode from explicit arg/env
       - in text-bridge mode, return text-only content from `_do_observe()`
       - extend the state JSON with:
         - `observe_delivery`
         - `bridge_model`
         - `view_variant`
         - `image_labels`
       - extend `observe` response trace payloads additively with:
         - `observe_delivery`
         - `bridge_model`
         - `bridge_latency_s`
         - `bridge_error`

    2. Keep `frame_capture` and snapshot writing image-bearing exactly as they
       are today so report generation still works.

    3. In `skills/ai2thor-navigator/SKILL.md`, update the tool contract so
       `roboclaws__observe()` is described as returning:
       - structured state
       - either images or a text bridge description
       - explicit metadata that says which delivery mode was used

    4. In `examples/openclaw_nav_autonomous.py::_kickoff_prompt()`, stop
       telling the agent to always interpret observe results as images. Name the
       new metadata instead.

    5. Update tests:
       - `tests/test_openclaw_mcp_server.py`
         - baseline image-bearing path unchanged
         - text-bridge path returns text-only blocks
         - bridge failure path returns safe text fallback
       - `tests/test_openclaw_nav_autonomous.py`
         - kickoff prompt mentions the new observe metadata
         - any additive runtime metadata written by the example is covered

    Do NOT broaden the Gateway tool profile in this plan. If Plan 01 chose the
    generic `image` tool route instead, implement only what that findings doc
    authorizes and keep the forbidden tool surface closed.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_openclaw_vision_bridge.py tests/test_openclaw_mcp_server.py tests/test_openclaw_nav_autonomous.py -q</automated>
  </verify>
  <done>The MCP server can deliver text-bridge observe results and the agent-facing contract is updated to match.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Host-side MCP server ↔ text-only main model | The server decides whether raw image bytes or bridge text reach the main model. |
| Vision intermediary ↔ main model | The bridge description must be truthful enough for navigation but must not become a second decision policy. |
| New observe metadata ↔ old replay/report pipeline | Host-side artifacts still rely on images and frozen top-level trace keys. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-02.8-02 | Information Disclosure | Split-model observe path | mitigate | In text-bridge mode, `_do_observe()` returns text only; raw images stay on the host-side trace path. |
| T-02.8-03 | Tampering | Frozen trace/report contracts | mitigate | Keep frame-capture images and frozen top-level keys intact; add only payload-level metadata. |
| T-02.8-04 | Reliability | Bridge failure path | mitigate | Return safe text fallback and error metadata instead of silently reverting to raw images. |

No `high`-severity threats; plan proceeds.
</threat_model>

<verification>
- `pytest tests/test_openclaw_vision_bridge.py tests/test_openclaw_mcp_server.py tests/test_openclaw_nav_autonomous.py -q` exits 0.
- Text-only observe delivery never exposes raw image blocks to the main model.
- Existing host-side replay/report capture still works.
</verification>

<success_criteria>
- Split-model observe behavior is real code, not prompt magic.
- The live agent contract matches the actual tool result shape.
- Vision-capable models remain on the existing path.
</success_criteria>


### FILE: .planning/phases/02.8-split-model-vision/02.8-03-selector-hardening-and-interactive-proof-PLAN.md

---
phase: 02.8
plan: 03
slug: selector-hardening-and-interactive-proof
type: execute
wave: 3
depends_on: ["02.8-02"]
files_modified:
  - Makefile
  - scripts/openclaw-bootstrap.sh
  - examples/openclaw_interactive.py
  - scripts/tail-openclaw-chat.py
  - tests/test_openclaw_bootstrap.py
  - tests/test_openclaw_interactive.py
  - tests/test_tail_openclaw_chat.py
autonomous: false
requirements_addressed: [A-07]

must_haves:
  truths:
    - "`make mimo-pro chat` and `make mimo chat` make the split-model path explicit instead of relying on operator guesswork."
    - "If Phase 02.8 introduces new bridge env vars, the selectors and bootstrap commentary reflect them accurately."
    - "The stale bootstrap assertion that still expects only `kimi` and `nvidia` is corrected so the focused split-model pytest slice can actually go green."
    - "The interactive banner/logging surfaces the main model, bridge model, and observe delivery mode."
    - "`tail-openclaw-chat.py` makes text-only `roboclaws__observe` results obvious to the operator."
    - "Default autonomous safety stays intact: Phase 02.8 does not casually widen the default tool profile."
  artifacts:
    - path: "Makefile"
      provides: "Operator entrypoints that make the split-model chat path explicit"
      contains: "mimo-pro"
    - path: "examples/openclaw_interactive.py"
      provides: "Interactive banner/runtime surfacing for split-model sessions"
      contains: "observe mode"
    - path: "scripts/tail-openclaw-chat.py"
      provides: "Readable summary of text-only observe tool results"
      contains: "toolResult"
  key_links:
    - from: "Makefile"
      to: "examples/openclaw_interactive.py"
      via: "selector env passed into the interactive entrypoint"
      pattern: "ROBOCLAWS_OBSERVE_MODE"
    - from: "examples/openclaw_interactive.py"
      to: "scripts/tail-openclaw-chat.py"
      via: "operator sees the same split-model delivery mode in the banner and the chat transcript"
      pattern: "text-bridge"
---

<objective>
Harden the already-existing MiMo operator path so the split-model configuration
is explicit, greppable, and debuggable. This plan is intentionally narrow:
Phase 02.8 is not rebuilding the Makefile/bootstrap stack from zero, it is
making the new bridge path operable and obvious.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/02.8-split-model-vision/02.8-CONTEXT.md
@.planning/phases/02.8-split-model-vision/02.8-RESEARCH.md
@.planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md
@Makefile
@scripts/openclaw-bootstrap.sh
@examples/openclaw_interactive.py
@scripts/tail-openclaw-chat.py
@tests/test_openclaw_bootstrap.py
@tests/test_openclaw_interactive.py
@tests/test_tail_openclaw_chat.py
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Make the split-model selector contract explicit</name>
  <read_first>
    - Makefile
    - scripts/openclaw-bootstrap.sh
    - .planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md
    - tests/test_openclaw_bootstrap.py
  </read_first>
  <behavior>
    - The text-only MiMo selectors say what they do in Phase 02.8 terms.
    - If the bridge introduced new env vars, the selectors pass them explicitly.
    - Bootstrap commentary/help text reflects the Phase 02.8 bridge contract.
    - The stale bootstrap test debt from the MiMo provider addition is repaired.
    - Default `minimal` safety remains unchanged unless Plan 01's evidence says otherwise.
  </behavior>
  <action>
    Update the operator-facing selector layer:

    1. In `Makefile`:
       - refresh the MiMo comments/help text so `mimo-pro` / `mimo` clearly
         describe the split-model observe bridge rather than a vague "image tool"
         story
       - if Plan 02 introduced new env vars such as
         `ROBOCLAWS_OBSERVE_MODE=text-bridge`, make the selectors pass them
         explicitly

    2. In `scripts/openclaw-bootstrap.sh`:
       - update the MiMo comments/help text so `IMAGE_MODEL` is documented as
         the bridge/intermediary model for Phase 02.8 as well as the Gateway's
         generic image-tool path
       - only change real bootstrap behavior if Plan 01's findings require it;
         otherwise keep behavior stable and limit the diff to truthful contract
         surfacing

    3. Update `tests/test_openclaw_bootstrap.py` so the focused bootstrap slice
       matches the current curated provider set and any Phase 02.8 selector
       contract changes. In particular, remove the stale assumption that the
       provider case block contains only `kimi` and `nvidia`.

    Do NOT broaden the default tool profile in this plan unless Plan 01
    explicitly chose that architecture and showed it is safe.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_openclaw_bootstrap.py -q</automated>
  </verify>
  <done>The selector/bootstrap layer tells the truth about the split-model bridge and focused bootstrap tests remain green.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Surface split-model sessions clearly in interactive tooling</name>
  <read_first>
    - examples/openclaw_interactive.py
    - scripts/tail-openclaw-chat.py
    - tests/test_openclaw_interactive.py
    - tests/test_tail_openclaw_chat.py
  </read_first>
  <behavior>
    - The interactive banner/log output tells the operator which main model,
      bridge model, and observe delivery mode are active.
    - The chat tail makes text-only observe results obvious without dumping full
      tool payloads.
    - The new surfacing works for both `mimo-v2.5-pro` and `mimo-v2.5`.
  </behavior>
  <action>
    Update the operator-facing tooling:

    1. In `examples/openclaw_interactive.py`:
       - surface the active `MODEL`
       - surface the active bridge/image model
       - surface the observe delivery mode in the banner and/or runtime events

    2. In `scripts/tail-openclaw-chat.py`:
       - when `roboclaws__observe` returns text-only content, make that visible
         in the summary line instead of reducing it to an opaque parts list
       - keep the existing terse formatting; do not dump raw JSON blobs

    3. Update tests:
       - `tests/test_openclaw_interactive.py` covers the banner/runtime surfacing
       - `tests/test_tail_openclaw_chat.py` covers a text-only `observe`
         toolResult path

    Keep the tool-output renderer general. The new behavior should help the
    split-model path without becoming a hard-coded MiMo-only parser.
  </action>
  <verify>
    <automated>cd /home/mi/ws/gogo/roboclaws && pytest tests/test_openclaw_interactive.py tests/test_tail_openclaw_chat.py -q</automated>
  </verify>
  <done>The interactive tooling makes split-model sessions obvious and the focused tests exit 0.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Operator-facing entrypoints ↔ actual runtime mode | Docs/help text must not drift away from what the code really does. |
| Chat transcript renderer ↔ live session diagnosis | Operators rely on terse summaries to understand whether the split-model path is active. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-02.8-05 | Repudiation | Operator entrypoints | mitigate | Make the selector and banner surface the active main model, bridge model, and observe mode explicitly. |
| T-02.8-06 | Elevation of Privilege | Bootstrap/tool profile | mitigate | Keep `minimal` as the default and treat any profile broadening as evidence-gated, not as a convenience edit. |

No `high`-severity threats; plan proceeds.
</threat_model>

<verification>
- `pytest tests/test_openclaw_bootstrap.py tests/test_openclaw_interactive.py tests/test_tail_openclaw_chat.py -q` exits 0.
- An operator can tell, without reading source, whether a chat session is running the split-model bridge.
- No accidental tool-surface broadening slips into the default path.
</verification>

<success_criteria>
- The split-model chat path is explicit and debuggable.
- Selector/help text matches the shipped runtime.
- Operators have a short-path way to confirm text-only observe results.
</success_criteria>


### FILE: .planning/phases/02.8-split-model-vision/02.8-04-local-dev-validation-and-doc-update-PLAN.md

---
phase: 02.8
plan: 04
slug: local-dev-validation-and-doc-update
type: execute
wave: 4
depends_on: ["02.8-03"]
files_modified:
  - roboclaws/openclaw/mcp_server.py
  - .planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md
  - .planning/phases/02.8-split-model-vision/02.8-LOCAL-PROBE-RESULTS.md
  - docs/openclaw-local.md
autonomous: false
requirements_addressed: [A-07]

must_haves:
  truths:
    - "A real local autonomous run proves that a text-only MiMo main model can navigate using bridged `roboclaws__observe` text."
    - "Real interactive runs prove `make mimo-pro chat` and `make mimo chat` expose text-only observe results to the live Gateway session."
    - "`02.8-LOCAL-PROBE-RESULTS.md` records exact commands, artifact paths, and dated verdicts without claiming cloud validation."
    - "`docs/openclaw-local.md` documents only the behavior actually proven locally."
    - "If local validation overturns the earlier spike or reveals a runtime bug, the repo fixes the shipped code and updates the spike record before calling the phase complete."
  artifacts:
    - path: ".planning/phases/02.8-split-model-vision/02.8-LOCAL-PROBE-RESULTS.md"
      provides: "Local evidence for autonomous and interactive split-model behavior"
      contains: "mimo-v2.5-pro"
    - path: "docs/openclaw-local.md"
      provides: "Operator-facing split-model setup and verification instructions"
      contains: "Split-model navigation (Phase 2.8)"
  key_links:
    - from: "docs/openclaw-local.md"
      to: ".planning/phases/02.8-split-model-vision/02.8-LOCAL-PROBE-RESULTS.md"
      via: "link from the split-model operator note to the dated local evidence"
      pattern: "02.8-LOCAL-PROBE-RESULTS"
---

<objective>
Validate the shipped split-model path on a real local workstation, record what
actually happened, and update the operator docs to match the live evidence.

This is the phase gate. Cloud sessions must not claim it happened.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md

THIS PLAN IS LOCAL-DEV ONLY. It requires Docker, AI2-THOR, a real MiMo key,
and the OpenClaw Gateway running locally.
</execution_context>

<context>
@.planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md
@.planning/phases/02.8-split-model-vision/02.8-RESEARCH.md
@docs/openclaw-local.md
@Makefile
@examples/openclaw_nav_autonomous.py
@examples/openclaw_interactive.py
@scripts/tail-openclaw-chat.py
</context>

<tasks>

<task type="checkpoint:human-verify" gate="blocking">
  <name>Task 1: Pre-flight — confirm local split-model validation is actually possible</name>
  <read_first>
    - AGENTS.md § 1 and § 7
    - docs/openclaw-local.md
  </read_first>
  <behavior>
    - Docker, AI2-THOR, and a real MiMo key are available.
    - No stale `openclaw-gateway` container is blocking the expected ports.
    - The focused pytest slice is green before starting live validation.
    - This plan does not proceed in a cloud-only session.
  </behavior>
  <action>
    Run and paste the outputs of:

    1. `docker --version`
    2. `python -c "import ai2thor; print(ai2thor.__version__)"`
    3. `[[ -n \"$MIMO_TP_KEY\" ]] && echo mimo-key-set || echo mimo-key-missing`
    4. `docker ps -a --format '{{.Names}}' | grep -x openclaw-gateway || echo absent`
    5. `env -i PATH=\".venv/bin:/usr/bin:/bin\" HOME=$HOME MIMO_TP_KEY=\"$MIMO_TP_KEY\" .venv/bin/pytest tests/test_openclaw_mcp_server.py tests/test_openclaw_bootstrap.py tests/test_openclaw_interactive.py tests/test_tail_openclaw_chat.py tests/test_openclaw_nav_autonomous.py -q`

    If this is a cloud session, stop here and defer.
  </action>
  <verify>
    <automated>MISSING — checkpoint task; operator confirms local readiness</automated>
  </verify>
  <done>Operator confirms a real local split-model validation session is available.</done>
</task>

<task type="auto">
  <name>Task 2: Run autonomous + interactive split-model probes and write `02.8-LOCAL-PROBE-RESULTS.md`</name>
  <read_first>
    - .planning/phases/02.8-split-model-vision/02.8-SPIKE-FINDINGS.md
    - examples/openclaw_nav_autonomous.py
    - examples/openclaw_interactive.py
    - scripts/tail-openclaw-chat.py
  </read_first>
  <behavior>
    - At least one autonomous run proves a text-only MiMo main model can complete a real navigation turn sequence using bridged observe text.
    - The interactive path proves both `make mimo-pro chat` and `make mimo chat`.
    - The write-up records what the agent actually saw in each path.
    - If local behavior contradicts Plan 01 or Plan 02, the repo is corrected before the phase is called done.
  </behavior>
  <action>
    Run these local probes and record every artifact path under `output/probes/02.8/`:

    1. **Autonomous proof run**

       ```bash
       cd /home/mi/ws/gogo/roboclaws
       set -a && source .env && set +a
       docker rm -f openclaw-gateway 2>/dev/null || true
       PROVIDER=mimo \
       MODEL=mimo_openai/mimo-v2.5-pro \
       IMAGE_MODEL=mimo_openai/mimo-v2-omni \
       python examples/openclaw_nav_autonomous.py \
         --scene FloorPlan201 \
         --max-moves 30 \
         --wall-budget 180 \
         --output-dir output/probes/02.8-autonomous-mimo-pro
       ```

       Record:
       - `terminated_by`
       - whether `trace.jsonl` contains the bridged observe metadata from Plan 02
       - whether the run made at least one move after a bridged `observe`
       - final `report.html` result

    2. **Interactive `mimo-pro` proof**

       - run `make mimo-pro chat`
       - in a second terminal run `make chat-tail`
       - send one short operator prompt in the Control UI asking the agent to
         describe what it sees and take one careful move
       - record the observed `roboclaws__observe` summary line

    3. **Interactive `mimo` proof**

       - repeat the same procedure with `make mimo chat`
       - record the observed `roboclaws__observe` summary line

    4. Write `.planning/phases/02.8-split-model-vision/02.8-LOCAL-PROBE-RESULTS.md`
       with:
       - date and host context
       - exact commands
       - artifact paths
       - autonomous result summary
       - `mimo-pro` interactive result summary
       - `mimo` interactive result summary
       - final shipped verdict

    If validation overturns the shipped behavior, do these remediation steps
    before marking the plan done:

    1. Fix the runtime code in `roboclaws/openclaw/mcp_server.py`.
    2. Append a dated correction note to `02.8-SPIKE-FINDINGS.md`.
    3. Re-run the focused pytest slice.
    4. Re-run the primary local validation path and record the corrected result.
  </action>
  <verify>
    <automated>test -f /home/mi/ws/gogo/roboclaws/.planning/phases/02.8-split-model-vision/02.8-LOCAL-PROBE-RESULTS.md && rg -n "autonomous-mimo-pro|make mimo-pro chat|make mimo chat|final shipped verdict" /home/mi/ws/gogo/roboclaws/.planning/phases/02.8-split-model-vision/02.8-LOCAL-PROBE-RESULTS.md</automated>
  </verify>
  <done>`02.8-LOCAL-PROBE-RESULTS.md` exists and records autonomous plus both interactive proof paths.</done>
</task>

<task type="auto">
  <name>Task 3: Update `docs/openclaw-local.md` with the validated split-model path</name>
  <read_first>
    - docs/openclaw-local.md
    - .planning/phases/02.8-split-model-vision/02.8-LOCAL-PROBE-RESULTS.md
    - Makefile
  </read_first>
  <behavior>
    - The docs explain the shipped split-model configuration without overstating what was validated.
    - Operators can find both the autonomous proof command and the interactive proof commands.
    - The docs explicitly call out that cloud sessions cannot claim this validation.
  </behavior>
  <action>
    Edit `docs/openclaw-local.md` near the existing MiMo / MCP sections and add:

    `### Split-model navigation (Phase 2.8)`

    It must include:

    - the main-model / vision-model pairing:
      - `mimo-v2.5-pro` or `mimo-v2.5` as `MODEL`
      - `mimo-v2-omni` as `IMAGE_MODEL` / bridge model
    - one autonomous proof command using `examples/openclaw_nav_autonomous.py`
    - the interactive proof commands:
      - `make mimo-pro chat`
      - `make mimo chat`
      - `make chat-tail`
    - the operator expectation that bridged runs expose text-only
      `roboclaws__observe` results in the live session log
    - where to find the dated local evidence:
      `.planning/phases/02.8-split-model-vision/02.8-LOCAL-PROBE-RESULTS.md`
    - one explicit sentence that cloud sessions must not claim this validation

    Keep the broader Phase 2.6 architecture section intact. This is a focused
    additive operator note, not a doc rewrite.
  </action>
  <verify>
    <automated>bash -c 'f=/home/mi/ws/gogo/roboclaws/docs/openclaw-local.md; rg -n "Split-model navigation \\(Phase 2.8\\)|make mimo-pro chat|make mimo chat|chat-tail|02.8-LOCAL-PROBE-RESULTS|cloud sessions" "$f"'</automated>
  </verify>
  <done>`docs/openclaw-local.md` contains a focused split-model subsection with commands, expectations, and the evidence link.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Local validation evidence ↔ repo docs | The docs must reflect only what the local runs actually proved. |
| Local shell ↔ repo-tracked markdown | Real bearer tokens and provider keys are in play and must not leak into tracked files. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-02.8-12 | Information Disclosure | Probe notes / docs | mitigate | Commands in markdown use env-var placeholders where needed; no raw tokens or Authorization headers land in tracked files. |
| T-02.8-13 | Repudiation | Final split-model verdict | mitigate | Record exact commands, artifact paths, and the final shipped verdict in `02.8-LOCAL-PROBE-RESULTS.md`. |
| T-02.8-14 | Tampering | Operator docs | mitigate | Limit the doc edit to a targeted split-model subsection tied to the dated local evidence file. |

No `high`-severity threats; plan proceeds.
</threat_model>

<verification>
- `02.8-LOCAL-PROBE-RESULTS.md` exists and records autonomous plus both interactive proofs.
- If validation required a runtime correction, `roboclaws/openclaw/mcp_server.py` and `02.8-SPIKE-FINDINGS.md` were updated before phase close-out.
- `docs/openclaw-local.md` contains `Split-model navigation (Phase 2.8)` with commands and the evidence link.
</verification>

<success_criteria>
- The split-model path is proven or corrected with real local evidence.
- Operators have one doc section that tells them how to run and verify the feature.
- The phase ends with a dated evidence-backed verdict instead of a cloud-only claim.
</success_criteria>


### FILE: .planning/phases/02.8-split-model-vision/02.8-CONTEXT.md

# Phase 2.8: Split-model navigation — Context

**Gathered:** 2026-04-23
**Status:** Ready for planning
**Source:** Design session — live conversation 2026-04-23

<domain>
## Phase Boundary

Extend the Phase 2.6 MCP autonomous loop to support text-only reasoning models
(mimo-v2.5-pro, mimo-v2.5) as the main model by adding a transparent vision
intercept layer. The agent's tool surface is unchanged; the intercept converts
image payloads inside `roboclaws__observe` results to text descriptions before
they reach the text-only main model.

Infrastructure already wired (not in scope here):
- `MIMO_TP_KEY` env var, `MimoProvider` class, `make mimo-pro chat` / `make mimo chat`
- `IMAGE_MODEL` bootstrap env var, `imageModel.primary` pinning in openclaw.json
- Bootstrap auto-sets `IMAGE_MODEL=mimo_openai/mimo-v2-omni` when main model is text-only

What is NOT in scope:
- Reopening the push model or the exec/curl Phase 2.5 contract
- Changing the `observe`/`move`/`done` tool names or their schemas as seen by the agent
- Any change to vision-capable paths (mimo-omni, kimi, nvidia continue to work as-is)

</domain>

<decisions>
## Implementation Decisions

### D-01: Why images don't reach text-only models today (LOCKED understanding)
The `roboclaws__observe` MCP tool returns a tool result that includes base64 image
bytes (FPV + overhead map). OpenClaw forwards this result directly to the main model.
Text-only models (mimo-v2.5-pro, mimo-v2.5) receive the message but cannot process the
image parts — they are blind to the visual content. The `IMAGE_MODEL` env var only
powers the Gateway's built-in generic `image` tool, NOT MCP tool results.

### D-02: Two image-processing paths in OpenClaw (LOCKED understanding)
- **Direct vision**: main model is vision-capable (mimo-omni, kimi, nvidia); images
  in tool results processed natively. Tool profile: `minimal`.
- **IMAGE_MODEL delegation**: main model is text-only; OpenClaw's built-in `image`
  tool routes to IMAGE_MODEL. BUT under `profile: minimal` the `image` tool is not
  exposed. Under `profile: coding` it is, but `coding` causes exec/curl drift (Phase 2.5 lesson).

### D-03: Preferred intercept location (LOCKED decision)
Intercept at the MCP server layer (`roboclaws/openclaw/` skill or a thin wrapper around
`roboclaws__observe`). When the caller model is text-only, the observe handler:
1. Captures the FPV + overhead map images as usual
2. Calls the vision model (IMAGE_MODEL / mimo-v2-omni) synchronously to produce a
   text description of the scene
3. Returns the text description (not raw images) as the tool result content
The agent receives text; its reasoning proceeds normally. The tool name/schema unchanged.

### D-04: Tool-profile probe (open question, spike required)
Probe whether `profile: messaging` exposes the generic `image` tool OR whether a
`tools.alsoAllow` mechanism in openclaw.json can add `image` to `profile: minimal`
without the full `coding` surface. If yes, an alternative implementation uses the
Gateway's built-in delegation instead of the MCP-layer intercept. Write-up required
regardless of outcome — this closes the architectural open question from Phase 2.6.

### D-05: Model routing (LOCKED)
The MCP server knows the main model via `MODEL` env var (already available in the
MCP server process environment, inherited from the bootstrap). Route:
- `MODEL` ends with `mimo-v2-omni` (or any vision-capable alias) → return raw images
- `MODEL` ends with `mimo-v2.5-pro` or `mimo-v2.5` → activate vision intercept
- Should be configurable via env var (e.g. `VISION_BRIDGE_MODEL`) rather than
  hard-coded model name matching, for future extensibility.

### D-06: Vision intercept output format (open question)
The text description returned to the agent should be structured enough for navigation:
- Scene description (objects, obstacles, directions)
- Agent position metadata (already in the text part of the observe result)
- Possibly a condensed overhead-map description (grid quadrant summary)
Exact format to be determined during the spike; optimise for navigation decisions not verbosity.

### D-07: `make mimo-pro chat` / `make mimo chat` already wired (LOCKED)
Makefile selector targets set `_MIMO_VARIANT` and `_MIMO_IMAGE_MODEL`. Bootstrap
auto-sets `IMAGE_MODEL=mimo_openai/mimo-v2-omni` when the text-only model is chosen.
Phase 2.8 does NOT need to change the Makefile or bootstrap — those are done.
Phase 2.8 only adds the MCP-layer intercept so the navigation actually works.

### Claude's Discretion
- Exact async/sync structure of the vision API call inside the MCP handler
- Whether to add a `--vision-bridge` CLI flag to openclaw_interactive.py or auto-detect from MODEL
- Test harness for the intercept (unit test with a mock vision client is acceptable)
- Report/trace schema for recording which observe results were text-bridged vs raw

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### MCP tool surface (Phase 2.6)
- `roboclaws/openclaw/` — MCP skill implementation; `observe` handler is the intercept point
- `docs/openclaw-gateway-internals.md` — tool profile docs (minimal/coding/messaging), IMAGE_MODEL flow
- `.planning/phases/02.6-openclaw-mcp-tools-integration/02.6-SPIKE-FINDINGS.md` — exec/curl drift lesson + minimal profile rationale

### Provider + bootstrap wiring (done in Phase 2.8 pre-work)
- `roboclaws/core/vlm.py` — `MimoProvider`, `_MODEL_ALIASES` (mimo-omni only for direct VLM)
- `scripts/openclaw-bootstrap.sh` — IMAGE_MODEL auto-set logic for text-only models
- `Makefile` — `mimo-pro` / `mimo` selector targets with `_MIMO_IMAGE_MODEL`

### Architecture + design
- `docs/openclaw-local.md` — model matrix, IMAGE_MODEL column, probe results
- `CLAUDE.md` — cloud vs local split; local-dev gate rule

</canonical_refs>

<specifics>
## Specific Ideas

- MiMo token-plan endpoint: `https://token-plan-cn.xiaomimimo.com/v1` — already used
  by `MimoProvider`; the vision intercept can reuse the same client
- The intercept can call `mimo-v2-omni` with just the image(s) + a brief "describe this
  scene for navigation" system prompt — no need for the full roboclaws system prompt
- Probe plan: run `make mimo-pro chat`, observe what the agent receives in the first
  `roboclaws__observe` result, confirm it's text not image bytes

</specifics>

<deferred>
## Deferred Ideas

- Supporting non-MiMo text-only models in the split-model path (e.g. GPT-4o-mini as
  main + GPT-4o as vision) — the intercept should be provider-agnostic by design but
  only MiMo is validated in Phase 2.8
- Multi-image batching (FPV + map in a single vision call vs two calls) — default to
  one call with both images; optimise later
- Streaming the vision description back to the agent (vs synchronous call) — out of scope

</deferred>

---

*Phase: 02.8-split-model-vision*
*Context gathered: 2026-04-23 from design session*


### FILE: .planning/phases/02.8-split-model-vision/02.8-RESEARCH.md

---
phase: "02.8"
kind: "research"
date: "2026-04-23"
status: "Complete"
---

# Phase 2.8 — Research

## Question

What do we need to know to plan split-model OpenClaw navigation in this repo
without reopening the Phase 2.5 `exec`/`curl` contract or casually widening the
Phase 2.6 `profile: minimal` tool surface?

## Current Runtime Facts

### 1. `roboclaws__observe` still emits raw MCP image blocks

`roboclaws/openclaw/mcp_server.py` currently does exactly this inside
`_do_observe()`:

- builds `state_text` as JSON
- returns `result = [state_text]`
- appends one `MCPImage(...)` per prompt image in `prompt_bundle.prompt_images`

So the current wire contract to the main model is still:

- first text block = structured state
- remaining blocks = raw PNG images (`fpv`, `overhead`, optional `chase`)

That matches the user constraint exactly: today a text-only main model still
receives image-bearing tool results it cannot use.

### 2. The bootstrap/Makefile wiring for text-only MiMo is already present

The repo already has the selector/bootstrap path that Phase 2.8 needs to build
on rather than re-invent:

- `Makefile`
  - `mimo-pro` sets `MODEL=mimo_openai/mimo-v2.5-pro`
  - `mimo` sets `MODEL=mimo_openai/mimo-v2.5`
  - both selectors pin `_MIMO_IMAGE_MODEL=mimo_openai/mimo-v2-omni`
- `scripts/openclaw-bootstrap.sh`
  - when `MODEL` matches `*mimo-v2.5-pro*|*mimo-v2.5`, it auto-sets
    `IMAGE_MODEL="${IMAGE_MODEL:-mimo_openai/mimo-v2-omni}"`
  - seeds `agents.defaults.imageModel.primary = image_model`
  - keeps each agent's `model.primary = model`

Planning implication: Phase 02.8 does not start from zero on `MODEL` /
`IMAGE_MODEL`. The missing piece is not selector wiring; it is how the
text-only main model consumes `roboclaws__observe`.

### 3. The tool-policy contract is intentionally narrow today

The current repo evidence is consistent on this point:

- `docs/openclaw-gateway-internals.md` documents exactly three profile values:
  `minimal`, `coding`, `messaging`
- `scripts/openclaw-bootstrap.sh` seeds each agent's tools block as only
  `{"profile": tool_profile}`
- `tests/test_openclaw_bootstrap.py` enforces that the default seeded tools
  block contains no extra keys: no `alsoAllow`, no deny list, no custom
  siblings

That means any "`alsoAllow` on minimal" path is an explicit deviation from the
shipped contract and requires live proof before it can become a plan default.

### 4. The direct VLM layer is not the right seam for the split-model bridge

`roboclaws/core/vlm.py` now catalogs MiMo text-only ids (`mimo-v2.5-pro`,
`mimo-v2.5`) alongside `mimo-v2-omni`, but the file's own comments still say:

- only `mimo-omni` is intended for direct image-bearing VLM navigation
- text-only MiMo ids should use the OpenClaw-side delegation path

More importantly, `MimoProvider.get_action(images, state)` is shaped for the
game-engine action contract, not for generic scene description:

- it builds the navigation system prompt
- it forces a tool schema named `AgentAction`
- it expects an action-oriented tool-call response

Planning implication: Phase 02.8 should add a narrow OpenClaw-side
image-description helper or bridge client, not try to reuse the direct
navigation provider as-is.

### 5. The existing proof seams already expose what the agent actually sees

Two shipped seams matter for validation:

- `examples/openclaw_interactive.py`
  - boots AI2-THOR + the MCP server + Gateway for live chat
  - is the natural seam for `make mimo-pro chat` / `make mimo chat`
- `scripts/tail-openclaw-chat.py`
  - pretty-prints the Gateway session JSONL
  - currently summarizes tool-result content by inner kind, e.g.
    `parts=['image', 'image', 'text']`

That means the interactive path can prove whether `roboclaws__observe` became
text-only from the agent's perspective without inventing a new UI.

### 6. The shipped agent instructions still assume image-bearing observes

`skills/ai2thor-navigator/SKILL.md` and
`examples/openclaw_nav_autonomous.py::_kickoff_prompt()` both currently say, in
effect, that `observe` returns frames / images and that the agent should use
`view_variant` and `image_labels` to interpret the image bundle.

Planning implication: if Phase 02.8 changes `observe` to return text for
text-only models, the skill/prompt surface must be updated in the same phase.
Otherwise the agent instructions will lag the runtime contract.

## Candidate Implementation Paths

### Option A — MCP-layer observe intercept (preferred)

Shape:

- keep `roboclaws__observe` as the only scene-perception entry point
- when `MODEL` is a text-only MiMo id, the MCP server:
  - captures the prompt bundle as usual
  - synchronously calls a vision model (`VISION_BRIDGE_MODEL` or
    `IMAGE_MODEL`, defaulting to `mimo_openai/mimo-v2-omni`)
  - returns text blocks instead of raw `MCPImage` blocks

Why this fits the repo:

- obeys Context D-03 directly
- keeps `profile: minimal` intact
- does not re-open `exec`, `browser`, `web_fetch`, or the generic `image` tool
- leaves vision-capable main models alone

Costs:

- needs a new bridge helper for scene description
- needs observe-mode metadata and skill/prompt updates
- needs additive trace/result metadata if operators are going to prove the path

### Option B — Gateway built-in `image` tool via `messaging` or `alsoAllow`

Shape:

- expose the generic `image` tool to the agent
- let the text-only main model call it against snapshots or tool-result images

Why it is attractive:

- conceptually uses the Gateway's existing `IMAGE_MODEL` abstraction
- avoids a custom intercept if the profile/tool-policy surface turns out to be
  cleaner than expected

Why it is not the default:

- Phase 2.6 shipped specifically to avoid the broader `coding` surface
- the current repo only documents `minimal` / `coding` / `messaging`
- the current bootstrap/tests explicitly forbid extra keys in the tools block
- no repo evidence yet proves that `messaging` or `alsoAllow` exposes `image`
  without reintroducing drift or extra escape hatches

Planning implication: this path gets a real local spike first. It does not get
to displace the MCP intercept on vibes.

### Option C — Reopen `coding` / `exec` / `curl`

Rejected. This conflicts with:

- the Phase 2.5 lesson in the roadmap and retros
- the Phase 2.6 minimal-profile contract
- the user's explicit scope boundary for Phase 02.8

This path should not appear in any Phase 02.8 implementation task.

## Recommended Runtime Contract

### Observe output

For the text-only split-model path, keep the first content block as structured
state JSON and replace the image blocks with one deterministic bridge text
block:

```text
[state_text_json, vision_bridge_text]
```

Recommended state additions:

- keep existing keys unchanged
- add `observe_content_mode`
  - `raw-images`
  - `vision-bridge-text`
- when bridged, set `image_labels` to `["vision_bridge"]` rather than leaving
  stale image labels that imply binary images still follow

This keeps the agent-facing contract additive while making the actual payload
truthful.

### Vision-bridge text format

The bridge text should be concise and navigation-shaped, not a verbose caption.
Recommended sections:

1. `Immediate view`
   - near obstacles
   - walkable openings
   - salient objects / landmarks
2. `Overhead summary`
   - rough free-space / blocked-space shape
   - nearby branches or dead ends
3. `Navigation cues`
   - short directional guidance in the current heading frame

The bridge should describe both FPV and overhead in one call, not emit two
independent captions.

### Trace / artifact metadata

Any new metadata must stay additive-only. The frozen contracts from Phase 2.6
still apply:

- keep the top-level `trace.jsonl` keys intact
- keep `snapshot_metrics()` key equality intact
- do not rename `sim_server_metrics` in `run_result.json`

Recommended additive fields:

- `observe_output_mode` in the observe response trace (`raw-images` vs
  `vision-bridge-text`)
- `vision_bridge_model` in the observe response trace
- optional `vision_bridge_error` only when the bridge falls back to a safe text
  response

## Validation Implications

### Automated

The phase already has good unit seams:

- `tests/test_openclaw_mcp_server.py`
  - current observe result shape
  - additive trace checks
- `tests/test_openclaw_bootstrap.py`
  - tools block shape
  - `IMAGE_MODEL` propagation
- `tests/test_openclaw_interactive.py`
  - interactive bootstrap/banner path
- `tests/test_tail_openclaw_chat.py`
  - pretty-printing of tool-result content kinds
- `tests/test_openclaw_nav_autonomous.py`
  - kickoff prompt wording
  - autonomous artifact writing

### Local-only

Cloud cannot prove the actual Phase 02.8 claim. Required local evidence:

1. Spike: does `messaging` or a manual `alsoAllow` patch expose a safe `image`
   tool path?
2. Autonomous run: can `mimo-v2.5-pro` or `mimo-v2.5` actually navigate with
   bridged `observe` text?
3. Interactive run: do `make mimo-pro chat` and `make mimo chat` show text-only
   `roboclaws__observe` results in the live session log?

## Concrete Planning Implications

1. Plan 01 should be a local-only capability spike. It must compare
   `minimal`, `messaging`, and an explicit `alsoAllow` candidate if the config
   accepts it, then lock the architecture verdict in a findings doc.
2. Plan 02 should implement the MCP-layer vision bridge as the default path and
   update the skill / kickoff prompt so the agent understands text-bearing
   observes.
3. Plan 03 should assume Makefile/bootstrap wiring already exists and focus on
   hardening/proving it with regression coverage and interactive proof surfaces,
   not on re-creating it from scratch.
4. Plan 04 should be local-dev only and must document both an autonomous proof
   run and the interactive `make mimo-pro chat` / `make mimo chat` proof path.

---

*Phase: 02.8-split-model-vision*
*Research completed: 2026-04-23*


### FILE: .planning/phases/02.8-split-model-vision/02.8-VALIDATION.md

---
phase: 02.8
slug: split-model-vision
status: planned
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-23
---

# Phase 02.8 — Validation Strategy

> Per-phase validation contract for split-model navigation.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest |
| **Config file** | none |
| **Quick run command** | `pytest tests/test_openclaw_mcp_server.py tests/test_openclaw_bootstrap.py tests/test_openclaw_interactive.py tests/test_tail_openclaw_chat.py tests/test_openclaw_nav_autonomous.py -q` |
| **Full suite command** | `env -i PATH=\".venv/bin:/usr/bin:/bin\" HOME=$HOME KIMI_API_KEY=\"$KIMI_API_KEY\" .venv/bin/pytest -q` |
| **Estimated runtime** | ~60 seconds for the focused slice, longer for full suite |

---

## Sampling Rate

- **After every implementation task:** run the smallest relevant pytest slice
- **After every plan wave:** run the full focused split-model slice
- **Before `$gsd-verify-work`:** full suite must be green
- **Max feedback latency:** 60 seconds for unit feedback

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 02.8-01-01 | 01 | 1 | A-07 | T-02.8-01 | Spike records real tool-surface evidence without leaking tokens | manual | `python spike/split_model_gateway_probe.py --help` | `spike/split_model_gateway_probe.py` | ⬜ pending |
| 02.8-02-01 | 02 | 2 | A-07 | T-02.8-05 | MCP `observe` returns text for text-only MiMo while preserving additive contracts | unit | `pytest tests/test_openclaw_mcp_server.py tests/test_openclaw_nav_autonomous.py -q` | `roboclaws/openclaw/mcp_server.py` | ⬜ pending |
| 02.8-03-01 | 03 | 3 | A-07 | T-02.8-08 | Existing `MODEL`/`IMAGE_MODEL` wiring stays narrow and interactive proof surfaces expose the text-only observe path clearly | unit | `pytest tests/test_openclaw_bootstrap.py tests/test_openclaw_interactive.py tests/test_tail_openclaw_chat.py -q` | `examples/openclaw_interactive.py` | ⬜ pending |
| 02.8-04-01 | 04 | 4 | A-07 | T-02.8-10 | Real local autonomous + interactive runs confirm the shipped split-model path and docs record only validated behavior | manual | `PROVIDER=mimo MODEL=mimo_openai/mimo-v2.5-pro IMAGE_MODEL=mimo_openai/mimo-v2-omni python examples/openclaw_nav_autonomous.py --scene FloorPlan201 --max-moves 30 --wall-budget 180` | `02.8-LOCAL-PROBE-RESULTS.md` | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠ flaky*

---

## Wave 0 Requirements

- [x] Existing test infrastructure covers the planned files; no Wave 0 scaffold is required.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Determine whether `messaging` or a patched `alsoAllow` path exposes a safe generic `image` tool | A-07 | Requires a real Gateway image, real model behavior, and live tool inventory evidence | Run Plan 01 locally, record tool lists and image-tool attempts in `02.8-SPIKE-FINDINGS.md`, and reject any path that widens the surface beyond the proven-safe contract |
| Confirm a text-only MiMo main model can complete a real navigation run using bridged observe text | A-07 | Requires Docker, AI2-THOR, and a real MiMo key | Run Plan 04 autonomous validation locally and record `terminated_by`, `trace.jsonl`, `run_result.json`, and `report.html` evidence in `02.8-LOCAL-PROBE-RESULTS.md` |
| Confirm `make mimo-pro chat` and `make mimo chat` expose text-only observe results to the live chat session | A-07 | Requires the Control UI and live session JSONL | Run Plan 04 interactive validation locally, tail the chat session, and record the observed `roboclaws__observe` part kinds in `02.8-LOCAL-PROBE-RESULTS.md` |

---

## Validation Sign-Off

- [x] Every plan has an automated verify command or an explicit manual-only gate
- [x] The manual-only checks are confined to local-dev behavior the cloud cannot prove
- [x] No frozen Phase 2.6 contract is allowed to drift without an additive-only note
- [x] No watch-mode flags
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** planned


