# Design Audit: README and SVG Assets

Date: 2026-05-13
Target: `README.md`, `docs/assets/supported-tools.svg`,
`docs/assets/architecture.svg`

## Consultation Direction

Product type: documentation-first workflow kit for AI-agent-developed repos.

Memorable thing: serious operating software for builders, not a generic SaaS
landing page.

Design posture: industrial/utilitarian, grid-disciplined, restrained color,
implementation-specific labels, and short readable diagrams.

## Findings

1. Medium impact: `supported-tools.svg` had clearer content than the old asset,
   but still read like a generic card layout and buried current updater phases
   in small text.

2. Medium impact: `architecture.svg` had a useful structure, but the visual
   treatment used heavy cards and a loose arrow that made the subsystem boundary
   feel less precise.

3. Polish: both SVGs were taller than needed for a README that is intentionally
   short.

## Fixes Applied

- Reworked `supported-tools.svg` into a compact update map: entrypoint, agent
  runtimes, setup phases, workflow sources, and output surfaces.
- Reworked `architecture.svg` into a numbered architecture flow with clearer
  subsystem boundaries.
- Reduced diagram heights from 420px to 340px.
- Reduced shadow/card styling and used tighter radii, thin strokes, and
  implementation labels.

## Evidence

- Before screenshots:
  - `screenshots/supported-tools-before.png`
  - `screenshots/architecture-before.png`
- After screenshots:
  - `screenshots/supported-tools-after-final.png`
  - `screenshots/architecture-after-v2.png`
- Validation:
  - `wc -l README.md` -> 141
  - `xmllint --noout docs/assets/supported-tools.svg docs/assets/architecture.svg`
  - `bun run verify` -> 6 passed, 0 failed

## Score

Design score: B -> A-

AI slop score: B -> A-

Residual risk: SVGs still use local font fallback chains because repo docs need
to render offline and on GitHub without a font pipeline.
