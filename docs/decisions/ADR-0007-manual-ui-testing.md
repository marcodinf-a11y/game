# ADR-0007: Manual UI Testing with Scoped Testability Contract

**Status:** Accepted

## Context

PRD Section 2.14 stated: "Every functional requirement group must be covered by automated tests." But Implementation Plan Phases 10-14 use manual verification for UI, console, game mode, and time control features (FR-UI, FR-CON, FR-GMD, FR-CTL-002).

Automated Godot UI testing requires significant infrastructure: headless Godot runtime, GdUnit4 framework, scene tree mocking, and CI integration. This is a side project unto itself.

## Decision

Scope the testability contract to simulation-engine FR groups. Presentation-layer FR groups use manual verification, with automated tests where logic is separable:

- **Console command parsing** — pure string-to-command mapping, easily unit testable.
- **Scenario win/lose detection** — testable headlessly via `SimulationTestHarness`.
- **Policy signal wiring** — verifiable without rendering.

A `UI-TESTING-ROADMAP.md` documents the infrastructure needed for full automated UI testing as a future side project.

## Consequences

- No automated UI test infrastructure blocks MVP progress.
- Simulation logic remains fully tested — the pure-library boundary ensures all economic behavior is testable without Godot.
- UI bugs must be caught through manual testing during development.
- The roadmap provides a clear path to automated UI testing post-MVP if desired.
