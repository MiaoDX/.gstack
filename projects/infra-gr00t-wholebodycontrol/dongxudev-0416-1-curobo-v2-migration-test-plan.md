# cuRobo v2 generalized grasp migration test plan

Generated: 2026-04-21T05:52:36Z

## Goal

Verify that the v2 migration hardens a generic public contract without reintroducing bottle-only or stage-name-only coupling.

## Required coverage

1. Request schema tests
   - single-arm side-entry bottle request
   - single-arm top-down request
   - placeholder two-hand box request parses without API changes
2. Result contract tests
   - stable success and failure taxonomy
   - no bottle-only fields required by generic callers
3. Compiler and adapter tests
   - preset -> goal candidate compilation
   - goalset and batch capacity guards
   - attachment lifecycle and result normalization
4. Controller integration tests
   - hidden stage mapping still executes grasp, close, and lift
   - caller does not need to reason about pregrasp or approach
5. Benchmark cases
   - bottle side-entry
   - bottle top-down
   - placeholder two-hand box fixture only, no execution required in first wedge
