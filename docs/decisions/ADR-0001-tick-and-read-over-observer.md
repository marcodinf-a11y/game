# ADR-0001: Tick-and-Read UI Updates over Observer Pattern

**Status:** Accepted

## Context

The Architecture described two conflicting UI update strategies:

- Section 6.3 defined `ISimulationEvents` with four events (`OnTickCompleted`, `OnIndicatorChanged`, `OnTransactionRecorded`, `OnPolicyEnacted`) and stated: "This avoids the UI polling the simulation and keeps the coupling one-directional."
- Section 3.1 defined `ISimulationState` as a read-only state object "that the UI can query" — a polling interface.

The Implementation Plan (Phases 9-11) silently used a read/poll model where the Game Controller exposes state and UI nodes read it each tick.

This is a core architectural pattern affecting every UI component.

## Decision

Adopt the tick-and-read pattern. The Game Controller reads `ISimulationState` after each tick and pushes data to UI nodes. `ISimulationEvents` was removed from the Architecture.

**Rationale:** The simulation ticks discretely (monthly). A poll-per-tick model where the Game Controller reads state after each tick and pushes it to UI nodes is simpler and sufficient for this tick rate. The observer pattern adds subscription management complexity with no benefit when all state changes happen at a single, predictable moment (the tick boundary).

## Consequences

- All UI components read from `ISimulationState` after each tick — no event subscription logic needed.
- Coupling remains one-directional: simulation never references UI types.
- If sub-tick UI updates are ever needed (e.g., animations during a tick), an event system could be reintroduced as an optimization. This is unlikely given the monthly tick granularity.
