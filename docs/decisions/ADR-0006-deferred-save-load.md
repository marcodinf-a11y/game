# ADR-0006: Deferred Save/Load with Replay-Log Approach

**Status:** Accepted

## Context

The Architecture listed "save/load" as a Game Controller responsibility, but the PRD had no save/load requirement and it wasn't listed as out of scope either.

Full state serialization is non-trivial for this simulation — it requires serializing the ledger, all balance sheets, the policy pipeline, capital stocks, AIDS parameters, pending lags, and more. This complexity would slow every implementation phase as the model evolves.

## Decision

Defer save/load to post-MVP. Added to PRD out-of-scope list. Removed "Handle save/load" from Architecture Game Controller responsibilities.

**Preferred approach for post-MVP:** A replay-log approach — record policy inputs per tick, then replay from tick 0 using the original seed to reconstruct state. The simulation is deterministic (seeded `IRandom`), so replaying with the same inputs reproduces the exact state at any tick.

## Consequences

- No serialization infrastructure needed during MVP development.
- The model can evolve freely without maintaining save format compatibility.
- Stellaris-style save compatibility is acceptable (saves may break across major releases).
- The replay-log approach shares infrastructure with SFC error recovery (ADR-0004).
- Trade-off: replay-based loading is slower than deserialization for long games, but avoids the versioning problem entirely.
