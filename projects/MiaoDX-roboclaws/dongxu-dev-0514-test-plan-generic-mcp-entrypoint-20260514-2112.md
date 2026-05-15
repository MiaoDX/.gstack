# Test Plan: Generic MCP Entrypoint And Semantic Capabilities

Created: 2026-05-14
Source plan: docs/plans/generic-mcp-entrypoint-semantic-capabilities.md

## Test Diagram

| New path | Unit | Contract | Regression risk |
| --- | --- | --- | --- |
| Profile declaration parser validates profile id, families, tools, provenance, exposure classification, privacy exclusions | yes | no | malformed profile loads silently |
| AI2-THOR navigation profile represents observe/observe_archived/move/done as canonical/profile tools and scene_objects/goto as accelerators | yes | yes | simulator shortcuts leak into canonical contract |
| MolmoSpaces cleanup profile represents metric_map/fixture_hints/observe/navigate/pick/place tools with ADR-0003 exclusions | yes | yes | private evaluator fields leak into public profile |
| Generic router loads a mock profile and registers only declared public tools | yes | yes | entrypoint registers stale or hidden tools |
| Accelerator opt-in path requires explicit flag and records accelerator provenance | yes | yes | debug helpers look like real robot capability |
| Existing demo launchers keep current commands and behavior | no | yes | router refactor breaks thin demo workflows |

## Required Fast Gates

- Targeted pytest: tests/contract/mcp/test_mcp_server.py tests/contract/molmo_cleanup/test_molmo_realworld_contract.py tests/contract/molmo_cleanup/test_molmo_realworld_mcp_server.py
- New focused tests for profile declarations, router registration, accelerator exclusion, and private-field leak checks.
- Repo wrapper for machine-local pytest isolation: ./scripts/dev/run_pytest_standalone.sh <targeted tests>
- Existing style gates if code changes: ruff check . and ruff format --check .

## Local-Dev Gates Not Required For This Phase

- Real OpenClaw Gateway, real VLM, Docker, GPU, AI2-THOR Unity, ROS/Nav2, and real robot validation are out of scope for the metadata/router prototype.
