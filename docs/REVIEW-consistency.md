# Document Consistency Review

**Date:** 2026-03-02
**Documents reviewed:** PRD.md, ARCHITECTURE.md, IMPLEMENTATION-PLAN.md
**Purpose:** Identify inconsistencies, gaps, and contradictions across and within the three core design documents before implementation begins.

## How to Read This Report

Each finding has a severity:

- **Critical** — Contradictions or gaps that will cause confusion or rework during implementation. Resolve before starting.
- **Medium** — Gaps that affect specific features but won't block early phases. Resolve before the affected phase begins.
- **Low** — Minor naming issues, missing tests, or ambiguities. Can be resolved during implementation.

Findings are grouped by theme. Each finding cites specific requirement IDs (e.g., FR-SIM-002) and document sections.

---

## Critical Findings

### C1. Observer Pattern vs. Polling — Unresolved UI Update Strategy

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (3.1, 6.3), IMPLEMENTATION-PLAN (Phases 9-11) |
| Requirements | FR-UI-002, FR-UI-003, FR-UI-004 |

**The problem:** The Architecture describes two conflicting UI update strategies and the Implementation Plan silently picks one.

- Architecture 6.3 defines `ISimulationEvents` with four events (`OnTickCompleted`, `OnIndicatorChanged`, `OnTransactionRecorded`, `OnPolicyEnacted`) and explicitly states: "This avoids the UI polling the simulation and keeps the coupling one-directional."
- Architecture 3.1 defines `ISimulationState` as a read-only state object "that the UI can query" — a polling interface.
- The Implementation Plan never builds `ISimulationEvents`. Phases 9-11 use a read/poll model where the Game Controller exposes state and UI nodes read it each tick.

**Why it matters:** This is a core architectural pattern that affects every UI component. If the observer pattern is intended, all UI phases need event subscription logic. If polling is intended, `ISimulationEvents` should be removed from the Architecture.

**Suggested resolution:** Pick one approach. Given that the simulation ticks discretely (monthly), a poll-per-tick model where the Game Controller reads state after each tick and pushes it to UI nodes is simpler and sufficient. Remove `ISimulationEvents` from the Architecture, or downgrade it to an optional optimization for the future. Update Architecture 6.3 to describe the actual update strategy.

---

### C2. ILedger Interface Does Not Support the Two Money Circuits

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-SIM-002), ARCHITECTURE (2.1, 3.4) |
| Requirements | FR-SIM-002 |

**The problem:** The PRD requires two separate money layers (reserves and deposits). The Architecture describes a "two-circuit ledger: reserves ledger + deposits ledger" in its component narrative (2.1). But the `ILedger` interface (3.4) defines a single `RecordTransaction(string from, string to, decimal amount, string category, string description)` method with no way to specify which circuit a transaction belongs to.

**Why it matters:** FR-SIM-002 is the core MMT money-circuit requirement. If the interface treats all transactions identically, the two-circuit separation cannot be enforced or verified at the interface level. Tests cannot check circuit isolation. The SFC checker cannot validate circuit-specific invariants.

**Suggested resolution:** Either add a `circuit` parameter to `RecordTransaction()`, or split the interface into `IReservesLedger` and `IDepositsLedger`, or use account naming conventions with a documented schema (e.g., accounts prefixed with `reserve:` or `deposit:`) and add circuit-aware query methods.

---

### C3. ISimulationFactory.Create() Returns Read-Only Interface Only

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (3.2, 3.7, 9.5) |
| Requirements | FR-CTL-001, FR-CTL-002 |

**The problem:** The `ISimulationFactory` interface (Architecture 3.7) is defined as:

```csharp
ISimulationState Create(IDataProvider dataProvider, int? seed = null);
```

This returns `ISimulationState`, which is read-only. But to actually run the simulation, callers also need `ISimulationCommands` to call `Tick()`, `SetSpendingLevel()`, etc. The `SimulationTestHarness` (Architecture 9.5) confirms this by exposing both `State` and `Tick()`.

**Why it matters:** Every consumer of the factory (Game Controller, test harness, console) needs both read and write access. The return type as defined is unusable on its own.

**Suggested resolution:** Return a composite type. Options:
- Return a `Simulation` class that implements both `ISimulationState` and `ISimulationCommands`
- Return a tuple or wrapper: `(ISimulationState State, ISimulationCommands Commands)`
- Define an `ISimulation` interface that extends both

---

### C4. Policy Lag System Has No Architecture Component

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-TIM-001, FR-TIM-002, FR-TIM-003, FR-UI-007), ARCHITECTURE (4.2) |
| Requirements | FR-TIM-001, FR-TIM-002, FR-TIM-003, FR-UI-007 |

**The problem:** Four PRD requirements depend on a policy lag and pipeline system:

- FR-TIM-001: Tax changes take effect after 1 tick, spending after 1-2 ticks, infrastructure effects over 6-12 ticks, etc.
- FR-TIM-002: Wage adjustments over 1-3 ticks, price adjustments over 1-2 ticks, etc.
- FR-TIM-003: Pending policy changes must be visible in the UI with estimated time to effect.
- FR-UI-007: Pipeline view must visually distinguish enacted, in-pipeline, and taking-effect states.

The Architecture mentions lags in the data flow narrative (4.2: "Simulation Engine queues policy change with appropriate lag") but defines no component for them. There is no `PolicyQueue`, `LagScheduler`, or `IPolicyPipeline` in the project structure, interfaces, or component descriptions. The `ISimulationState` interface has no property to expose pending changes. The `ISimulationEvents` interface has `OnPolicyEnacted` but nothing for "policy queued" or "policy taking effect."

**Why it matters:** Without an architecture for the lag system, Phase 7 of the Implementation Plan must invent the design during implementation. The UI pipeline display (FR-TIM-003, FR-UI-007) has no data source.

**Suggested resolution:** Add a `PolicyPipeline` component to the Architecture with:
- An interface for querying pending changes (for the UI)
- A mechanism for the tick engine to process pending changes at the start of each phase
- Data-driven lag durations (loaded from `IDataProvider`)

---

### C5. Investment and Depreciation Absent from Architecture

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-INV-001, FR-INV-002), ARCHITECTURE (2.1, 4.1, 5) |
| Requirements | FR-INV-001, FR-INV-002 |

**The problem:** The PRD defines two requirement groups for investment:

- FR-INV-001: Public investment (infrastructure increases capacity, public services increase productivity, public capital depreciates).
- FR-INV-002: Private investment (firms invest in capital, funded from profits/loans, capital produced by industry sector, capital depreciates).

The Architecture has no investment component. There is no `InvestmentEngine.cs` or similar in the project structure (Section 5). The tick data flow (Section 4.1) has no investment step in any of the five phases. Capital depreciation is not mentioned anywhere in the Architecture. The `ProductionEngine.cs` is listed but covers production, not investment.

**Why it matters:** Investment and depreciation are core economic mechanics that affect capacity, productivity, and inter-sector dependencies. Without architecture, Phase 7 must design this from scratch.

**Suggested resolution:** Add an investment/depreciation step to the tick data flow (likely in the Production Phase or as a new sub-phase). Add an `InvestmentEngine.cs` to the project structure. Define how investment decisions interact with the banking system (loan requests) and the production system (capital goods from industry).

---

### C6. Three Inflation Buffers Have No Architecture Component

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-PRC-002), ARCHITECTURE (2.1, 4.1), IMPLEMENTATION-PLAN (Phase 4, Phase 8) |
| Requirements | FR-PRC-002 |

**The problem:** FR-PRC-002 requires three specific inflation buffers — a key MMT differentiator:

1. Productivity gains absorb wage increases (if productivity rises with wages, no price pressure)
2. Demand slack absorbs spending increases (if idle capacity exists, more output not higher prices)
3. Profit margin compression absorbs cost increases (firms may accept lower markup)

The PRD states: "Inflation must only occur when all three buffers are exhausted."

The Architecture mentions `PricingEngine.cs` and cost-plus markup pricing but provides no component or mechanism for the three-buffer gating logic. The Implementation Plan tests buffers 1 and 2 (Phase 4 tests 10-11) but has no test for buffer 3 (profit margin compression) and no test for the combined "all three exhausted" condition.

**Why it matters:** This is the central mechanism that distinguishes the MMT inflation model from mainstream models. If it is not architecturally specified and fully tested, the game's core educational value is undermined.

**Suggested resolution:** Extend the `PricingEngine` architecture to explicitly include the three-buffer check. Add tests to the Implementation Plan:
- `FrPrc002_CostRiseWithMarginSlack_FirmAbsorbsViaSmallerMarkup`
- `FrPrc002_AllThreeBuffersExhausted_InflationOccurs`

---

## Medium Findings

### M1. Console Time Commands Cannot Reach Game Controller

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-CON-004), ARCHITECTURE (2.2, 3.2, 4.3) |
| Requirements | FR-CON-004 |

**The problem:** FR-CON-004 requires the console to support `pause`, `resume`, `speed`, and `tick` commands. Pause/resume/speed are Game Controller responsibilities (Godot layer), but the console routes commands through `ISimulationCommands` (simulation layer). The Architecture provides no path from simulation-layer console commands to Godot-layer time controls.

**Suggested resolution:** Either move time control to the simulation layer (simulation manages its own pause state), or add a separate `ITimeControl` interface that the console can access alongside `ISimulationCommands`.

---

### M2. Console Command Parsing Responsibility Is Contradictory

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (3.2, 4.3) |
| Requirements | FR-CON-002, FR-CON-003 |

**The problem:** Architecture 4.3 (data flow) says "Console node parses command" — parsing happens in the Godot presentation layer. But Architecture 3.2 defines `ExecuteConsoleCommand(string command)` on `ISimulationCommands`, which accepts a raw string, implying the simulation engine does the parsing.

If the Console node parses first, it should call typed methods (like `SetSpendingLevel`), not pass a raw string. If the simulation parses, then the Console node is not parsing.

**Suggested resolution:** Clarify: the Console node handles input display, history, and tokenization. It then either (a) routes to typed `ISimulationCommands` methods for simulation commands and to the Game Controller for time commands, or (b) passes a raw string to a `CommandInterpreter` in the simulation layer. Remove the ambiguity by picking one and updating the data flow.

---

### M3. Multiple State Interfaces Referenced But Never Defined

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (3.1) |
| Requirements | FR-AGT-001, FR-AGT-002, FR-AGT-003, FR-SIM-004 |

**The problem:** `ISimulationState` (Architecture 3.1) references four interfaces that are never defined:

- `IGovernmentState` — needed for FR-AGT-001 (deficit/surplus, spending allocation, bond issuance)
- `ICentralBankState` — needed for FR-AGT-002 (reserve accounts, policy rate)
- `IBankingState` — needed for FR-AGT-003 (reserves, loans, deposits, bonds, equity)
- `IEconomicIndicators` — needed for FR-SIM-004 (12 specific indicators)

Without definitions, there is no way to verify that the Architecture covers the PRD requirements for these entities.

**Suggested resolution:** Define all four interfaces with properties mapping to their respective PRD requirements. At minimum, `IEconomicIndicators` should list all 12 indicators from FR-SIM-004.

---

### M4. IRandom Interface Never Implemented

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (6.5), IMPLEMENTATION-PLAN (all phases) |
| Requirements | NFR-CQA-002 (deterministic tests) |

**The problem:** Architecture 6.5 lists `IRandom` as a core injectable dependency for "seeded random number generation for deterministic tests." No implementation phase creates this interface or its implementations. Phase 8's property-based tests depend on deterministic randomness via the seed parameter on `ISimulationFactory`, but the underlying `IRandom` is never built.

**Suggested resolution:** Add `IRandom` creation to Phase 0 (infrastructure) or Phase 2 (first phase that might need randomness). Define it alongside the other core interfaces.

---

### M5. "Sector" Balance Sheets (PRD) vs. Per-Agent Balance Sheets (Architecture)

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-UI-003), ARCHITECTURE (3.1, 3.4) |
| Requirements | FR-UI-003 |

**The problem:** FR-UI-003 requires the UI to "display SFC balance sheets for each sector." In SFC accounting, "sector" means an aggregate: the government sector, the banking sector, the household sector, etc. But the Architecture defines `IBalanceSheet` per agent (each has an `OwnerId`), and `ISimulationState` returns `IReadOnlyList<IBalanceSheet> AllBalanceSheets` — a flat list of per-agent sheets.

There is no aggregation mechanism to produce sector-level balance sheets from per-agent data.

**Suggested resolution:** Add a method to `ISimulationState` like `IReadOnlyDictionary<string, IBalanceSheet> SectorBalanceSheets` that returns aggregated views, or document that the UI is responsible for aggregating per-agent sheets by agent type.

---

### M6. PRD FR-SIM-002 Oversimplifies Money Circuit Access

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-SIM-002, FR-AGT-003) |
| Requirements | FR-SIM-002 |

**The problem:** FR-SIM-002 states: "The private sector must only interact with deposit money" and "The central bank and Treasury must only interact with reserve money."

But commercial banks are private sector entities that necessarily interact with **both** circuits: they hold reserve accounts at the central bank (reserve money) and maintain deposit accounts for customers (deposit money). FR-AGT-003 explicitly requires banks to "hold a reserve account at the central bank" and "buy government bonds at auction" (which requires reserves).

**Suggested resolution:** Amend FR-SIM-002 to clarify that banks bridge both circuits. Something like: "Non-bank private sector agents (households, firms) must only interact with deposit money. The central bank and Treasury must only interact with reserve money. Commercial banks operate in both circuits, holding reserve accounts at the central bank and deposit accounts for customers."

---

### M7. Bond Yields Indicator Requires Out-of-Scope Mechanics

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-SIM-004, Section 4) |
| Requirements | FR-SIM-004 |

**The problem:** FR-SIM-004 lists "bond yields" as a required indicator. But Section 4 (Out of Scope) explicitly excludes "bond secondary market." Without a secondary market, bond yields are identical to the coupon rate set at auction — making the "yield" indicator either redundant or misleading.

**Suggested resolution:** Either define "bond yields" as the weighted average coupon rate across outstanding bonds (which is meaningful without a secondary market), or replace it with "average bond coupon rate" to avoid implying secondary market mechanics.

---

### M8. Household Bond Participation Has No Architecture Support

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-BND-001), ARCHITECTURE (3.3), IMPLEMENTATION-PLAN (Phase 6) |
| Requirements | FR-BND-001 |

**The problem:** FR-BND-001 states: "Banks and high-income households must be able to bid" on government bonds. The Architecture provides no mechanism for household bond participation. The `IHouseholdClass` interface (Architecture 3.3) has no bond-related properties. Implementation Plan Phase 6 has no test for household bond bidding.

**Suggested resolution:** Add bond-related properties to `IHouseholdClass` (or its state interface). Add a test to Phase 6: `FrBnd001_HighIncomeHouseholds_CanBidOnBonds`.

---

### M9. Missing Project Structure Files

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (5), IMPLEMENTATION-PLAN (various phases) |

**The problem:** Several files listed in the Architecture's project structure (Section 5) are never created in any implementation phase:

| File | Status |
|---|---|
| `data/base/map/default-map.json` | Listed in Architecture, never created. Phase 14 creates a map asset but never mentions this JSON file. |
| `tests/Game.Tests/Game.Tests.csproj` | An entire test project listed in Architecture, never established in any phase. |
| `tests/Simulation.Tests/Helpers/TestDataBuilder.cs` | Listed in Architecture, never created. `InMemoryDataProvider` and `SimulationTestHarness` are created, but `TestDataBuilder` is distinct (builder pattern for constructing test data objects). |
| `TestData/Scenarios/*.json` | Architecture 9.3 lists `high-inflation.json`, `high-unemployment.json`, `steady-state.json`. No phase creates them. |

**Suggested resolution:** Either add these files to the appropriate implementation phases, or remove them from the Architecture if they are not needed for the MVP.

---

## Low Findings

### L1. Wrong Requirement ID in Test Name

| Aspect | Detail |
|---|---|
| Documents | IMPLEMENTATION-PLAN (Phase 4) |

Phase 4 test 12: `FrSim003_HouseholdsBuySurvivalBeforeComfort` references FR-SIM-003 (Monthly Tick Processing — tick phase ordering). The test is about hierarchical consumption ordering, which is defined in FR-AGT-004.

**Fix:** Rename to `FrAgt004_HouseholdsBuySurvivalBeforeComfort`.

---

### L2. Missing Test Coverage for Specific PRD Sub-Requirements

| Aspect | Detail |
|---|---|
| Documents | PRD (various), IMPLEMENTATION-PLAN (various phases) |

The following PRD sub-requirements have no corresponding test in the Implementation Plan:

| PRD Requirement | Missing Test |
|---|---|
| FR-LBR-001 | Wage downward stickiness |
| FR-LBR-001 | Wage influenced by sector conditions, labor scarcity, firm profitability |
| FR-LBR-002 | Cross-sector worker mobility over time |
| FR-PRC-002 | Profit margin compression buffer (buffer 3 of 3) |
| FR-PRC-002 | All three buffers exhausted -> inflation occurs |
| FR-PRC-003 | Sector-specific price tracking |
| FR-AGT-004 | Debt capacity varies by household class |
| FR-AGT-004 | Price elasticity varies by need level |
| FR-BND-001 | High-income households can bid on bonds |
| FR-INV-002 | Capital goods produced by industry sector |
| FR-TIM-002 | Hiring/firing lag (1-2 ticks) |
| FR-TIM-002 | Investment-to-capacity lag (3-6 ticks) |
| FR-TIM-002 | Household spending adjustment lag (1 tick) |
| FR-SIM-004 | Individual tests for: unemployment rate, bond yields, savings rate, wage growth, bank reserves |

---

### L3. Testability Contract Contradicts Manual-Only Testing for UI Phases

| Aspect | Detail |
|---|---|
| Documents | PRD (2.14), IMPLEMENTATION-PLAN (Phases 10-14) |

PRD Section 2.14 states: "Every functional requirement group must be covered by automated tests." But Phases 10-14 use "manual verification" as their testing strategy for FR-UI, FR-CON, FR-GMD, and FR-CTL-002.

The PRD's own test coverage mapping table (Section 2.14) also omits these groups, which is consistent with the Implementation Plan but contradicts the blanket statement.

**Suggested resolution:** Amend the PRD's testability contract to scope it to simulation-engine FR groups, or add automated tests for at least: console command parsing (easily unit testable), scenario win/lose detection (testable headlessly), and policy change signal wiring.

---

### L4. IAgentRegistry Interface Not Defined

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (6.5), IMPLEMENTATION-PLAN (Phase 3) |

Architecture 6.5 lists `IAgentRegistry` as a core injectable dependency. Phase 3 builds a concrete `AgentRegistry` class but never defines the `IAgentRegistry` interface. The dependency injection pattern requires the interface form for test substitution.

---

### L5. Overlapping Policy Changes Underspecified

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-CTL-001, FR-TIM-001) |

FR-CTL-001 says all controls are adjustable at any time, including while paused. FR-TIM-001 defines policy lags. But if the player changes the tax rate while paused, then changes it again before unpausing, the PRD does not specify behavior: does the second change replace the first, queue behind it, or get rejected?

---

### L6. Save/Load in Architecture with No PRD Requirement

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (2.2), PRD (all sections) |

Architecture 2.2 says the Game Controller handles "save/load." The PRD has no save/load requirement, and it is not listed as out of scope either.

**Suggested resolution:** Either add save/load to the PRD as a requirement or to the out-of-scope list, or remove it from the Architecture.

---

### L7. Architecture Data Model List Incomplete

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (5), IMPLEMENTATION-PLAN (Phase 1) |

Architecture Section 5 lists 4 data model files under `src/Simulation/Data/Models/`: `SectorData.cs`, `HouseholdData.cs`, `GoodsData.cs`, `ScenarioData.cs`.

Implementation Plan Phase 1 needs 10 model classes, adding: `FirmData`, `BankData`, `GovernmentData`, `ProductionData`, `ParametersData`, `SimulationConfigData`. These correspond to JSON files already listed in the Architecture's `data/base/` tree — the model file list is simply incomplete.

Similarly, `IDataValidator.cs` and `DataValidator.cs` are described in Architecture 3.6 but missing from the project structure file listing in Section 5.

---

### L8. Terminology Inconsistencies

| Term Variants | Context |
|---|---|
| "Household classes" / "population groups" | Used interchangeably across all three documents, never explicitly equated. PRD uses "classes" in requirements, "population groups" in out-of-scope. Architecture uses "population groups" in technical justifications. |
| `IBankingState` / `CommercialBank` / "Banks" | The state interface naming (`IBankingState`) does not follow the same pattern as the agent name (`CommercialBank`). Property is named `Banks` (plural) despite "multiple competing banks" being out of scope. |
| `IHouseholdClassState` / `IHouseholdClass` | Two interfaces with no documented relationship — the "State" suffix implies a read-only view, but this is not stated. |
| `IFirmSectorState` / `IFirm` | Same pattern as above. |
| "Dashboard" / "score dashboard" | PRD calls it a "dashboard" for sandbox mode (no win/lose). Implementation Plan adds "score," implying scoring in a mode with no win/lose conditions. |
| `IMPLEMENTATION-PLAN.md` | Listed in the docs directory on disk but omitted from Architecture Section 5's project structure listing. |

---

### L9. Phase 6 Dependency on Phase 5 Overstated

| Aspect | Detail |
|---|---|
| Documents | IMPLEMENTATION-PLAN (Phase 6) |

Phase 6 lists dependencies as "Phase 3, Phase 5 (banks need to exist)." But banks exist after Phase 3 (which creates all agent types). Phase 5 adds lending behavior, which is not required for bond purchasing. The stated rationale "banks need to exist" is satisfied by Phase 3 alone. Phase 6 may not actually depend on Phase 5 unless the bond auction requires lending infrastructure.

This means Phases 5 and 6 could potentially be parallelized or reordered.

---

## Appendix: Coverage Summary

### Requirements Fully Covered Across All Documents

These requirement groups have consistent coverage in the PRD, Architecture, and Implementation Plan with no gaps:

FR-SIM-001 (SFC accounting), FR-SIM-003 (tick processing), FR-AGT-002 (central bank), FR-AGT-003 (commercial banks), FR-AGT-005 (firms), FR-PRC-001 (cost-plus pricing), FR-LBR-003 (unemployment), FR-BNK-001 (endogenous money creation), FR-BNK-002 (creditworthiness), FR-BNK-003 (loan types), FR-BNK-004 (debt service/default), FR-BND-002 (bond properties), FR-CTL-001 (policy levers), FR-CTL-002 (time controls), FR-GMD-001 (sandbox), FR-GMD-002 (scenario), FR-MOD-001 (data-driven design), FR-MOD-002 (data schema), FR-MOD-003 (base game as mod), FR-CON-001 through FR-CON-006 (console).

### Requirements With Gaps

| Requirement | Issue(s) |
|---|---|
| FR-SIM-002 | C2 (ILedger interface), M6 (oversimplified circuit access) |
| FR-SIM-004 | M3 (IEconomicIndicators undefined), M7 (bond yields), L2 (missing indicator tests) |
| FR-AGT-001 | M3 (IGovernmentState undefined) |
| FR-AGT-004 | L2 (missing debt capacity and elasticity tests) |
| FR-PRC-002 | C6 (no architecture, missing tests for buffer 3) |
| FR-PRC-003 | L2 (sector-specific price tracking untested) |
| FR-LBR-001 | L2 (wage stickiness and determinants untested) |
| FR-LBR-002 | L2 (cross-sector mobility untested) |
| FR-BND-001 | M8 (household bond participation) |
| FR-INV-001 | C5 (absent from architecture) |
| FR-INV-002 | C5 (absent from architecture), L2 (capital goods from industry untested) |
| FR-TIM-001 | C4 (no architecture component), L2 (lag ranges untested) |
| FR-TIM-002 | C4 (no architecture component), L2 (most lag types untested) |
| FR-TIM-003 | C4 (no data source for UI) |
| FR-UI-007 | C4 (no data source for pipeline display) |
