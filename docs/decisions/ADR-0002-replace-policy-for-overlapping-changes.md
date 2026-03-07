# ADR-0002: Replace Model for Overlapping Policy Changes

**Status:** Accepted

## Context

The player can adjust policy controls at any time, including while paused (FR-CTL-001). Policy changes have lag timers before taking effect (FR-TIM-001). If the player changes the tax rate while paused, then changes it again before unpausing, the behavior for the first pending change is unspecified. Three options:

1. **Replace** — the latest value overwrites any pending change for the same parameter, lag timer resets.
2. **Queue** — changes queue behind each other, each with its own lag.
3. **Reject** — the second change is blocked until the first takes effect.

## Decision

Adopt the **replace** model. The latest value overwrites any pending change for the same parameter, and the lag timer resets.

**Rationale from game precedents:**

- **Democracy 3/4:** Policies are sliders with delayed effects. Moving a slider again before the previous change lands replaces the target value. Players intuitively understand they are setting a destination, not queuing commands.
- **Victoria 3:** Structural changes (law enactments) allow only one in-progress change per category. Changing direction means canceling and restarting — effectively replace with an explicit cancel step.
- **EU4 / HOI4:** Numerical adjustments (tax sliders, production) are immediate or near-immediate. Structural changes use cooldown timers that prevent re-adjustment for N months.

This game's three policy levers (tax rate, spending level, spending allocation) are numerical/slider-style controls, making the Democracy model the closest fit.

## Consequences

- Simple to implement: the `IPolicyPipeline` entry updates its target value.
- Matches the player mental model — instant correction of intent, not instant effect.
- Avoids surprising queue buildup.
- Fully respects FR-CTL-001 (the player can always adjust).
- If post-MVP structural changes are added (e.g., introducing a Job Guarantee program), those could use a different mechanism (Victoria 3-style one-at-a-time enactment) since the process of enacting matters, not just the target value.
