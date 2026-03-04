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

### ~~C1. Observer Pattern vs. Polling — Unresolved UI Update Strategy~~ ✅ Resolved

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

**Resolution:** Adopted the tick-and-read pattern. Removed `ISimulationEvents` from Architecture 6.3, replaced with tick-and-read flow diagram. Updated Section 3.1 to reference the new update strategy. See commit `40afa78`.

---

### ~~C2. ILedger Interface Does Not Support the Two Money Circuits~~ ✅ Resolved

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-SIM-002), ARCHITECTURE (2.1, 3.4) |
| Requirements | FR-SIM-002 |

**The problem:** The PRD requires two separate money layers (reserves and deposits). The Architecture describes a "two-circuit ledger: reserves ledger + deposits ledger" in its component narrative (2.1). But the `ILedger` interface (3.4) defines a single `RecordTransaction(string from, string to, decimal amount, string category, string description)` method with no way to specify which circuit a transaction belongs to.

**Why it matters:** FR-SIM-002 is the core MMT money-circuit requirement. If the interface treats all transactions identically, the two-circuit separation cannot be enforced or verified at the interface level. Tests cannot check circuit isolation. The SFC checker cannot validate circuit-specific invariants.

**Suggested resolution:** Either add a `circuit` parameter to `RecordTransaction()`, or split the interface into `IReservesLedger` and `IDepositsLedger`, or use account naming conventions with a documented schema (e.g., accounts prefixed with `reserve:` or `deposit:`) and add circuit-aware query methods.

**Resolution:** Added `MoneyCircuit` enum (Reserves, Deposits) to Section 3.4. `ITransaction` now carries a `Circuit` property. `ILedger.RecordTransaction` takes a `MoneyCircuit` parameter. Added `CheckCircuitIsolation()` and `GetCircuitTotal()` to `ILedger`. Updated Section 2.1 narrative to describe circuit enforcement rules (non-bank private agents → Deposits only, Treasury/CB → Reserves only, banks → both). Updated Section 4.1 tick data flow to show circuit tags on transactions and circuit isolation check in the Accounting Phase. Also addresses M6 (banks bridging both circuits).

---

### ~~C3. ISimulationFactory.Create() Returns Read-Only Interface Only~~ ✅ Resolved

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

**Resolution:** Introduced `ISimulation` composite interface extending both `ISimulationState` and `ISimulationCommands`. Updated `ISimulationFactory.Create()` return type to `ISimulation` in Section 3.7. Updated Section 2.2 (Game Controller) to clarify it holds `ISimulation` and exposes `ISimulationState` to UI. Updated Section 9.5 (test harness) to note it wraps `ISimulation`.

---

### ~~C4. Policy Lag System Has No Architecture Component~~ ✅ Resolved

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

**Resolution:** Added `IPolicyPipeline` and `IPendingPolicy` interfaces to Architecture Section 3.8, with `PolicyChangeKind` and `PolicyChangeStatus` enums. Added `PolicyPipeline` property to `ISimulationState` (Section 3.1). Updated Tick Engine description (Section 2.1) to flush pipeline at Government Phase start. Updated Government Phase in tick data flow (Section 4.1). Replaced vague player input narrative (Section 4.2) with specific pipeline flow. Added `PolicyPipeline.cs` and `PendingPolicy.cs` to project structure (Section 5). Added `IPolicyPipeline` to injectable dependencies (Section 6.5). Updated Implementation Plan Phase 7 to reference Architecture Section 3.8. Also addresses L5 (overlapping policy changes).

---

### ~~C5. Investment and Depreciation Absent from Architecture~~ ✅ Resolved

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

**Resolution:** Added `IInvestmentEngine` and `ICapitalStock` interfaces in Architecture Section 3.9, with methods for public investment processing, private investment processing, and depreciation. Added `CapitalStock`, `DepreciationRate`, and `InvestmentDemand` properties to `IFirm` (Section 3.3). Inserted investment steps into the tick data flow (Section 4.1): `ProcessPublicInvestment()` in Government Phase, `ProcessPrivateInvestment()` in Production Phase, `ApplyDepreciation()` in Accounting Phase. Added `InvestmentEngine.cs` to project structure Economics/ directory and `InvestmentData.cs` to Data/Models/ (Section 5). Added `IInvestmentEngine` to injectable dependencies (Section 6.5). Added Investment Engine component description to Section 2.1. Phase 7 of the Implementation Plan now references Architecture Section 3.9 for both public and private investment implementation.

---

### ~~C6. Three Inflation Buffers Have No Architecture Component~~ ✅ Resolved

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

**Resolution:** Added `IPricingEngine` interface with `SetPrices()` method and full three-buffer narrative to Architecture Section 3.10. Added `CurrentMarkup` and `MinimumMarkup` properties to `IFirm` (Section 3.3). Updated tick data flow (Section 4.1): annotated Production Phase with buffer 2 note, expanded Market Phase pricing step to show all three buffers and residual pressure flow. Added `PricingData.cs` to project structure Data/Models/ (Section 5). Added `IPricingEngine` to injectable dependencies (Section 6.5). Added three buffer tests to Implementation Plan Phase 4 RED section (tests 12-14: margin absorption, residual pass-through, all-three-exhausted). Updated Phase 4 GREEN step 3 to reference Architecture §3.10 three-buffer logic. Added progressive buffer exhaustion integration test to Phase 8 RED section (test 10).

---

## Medium Findings

### ~~M1. Console Time Commands Cannot Reach Game Controller~~ ✅ Resolved

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-CON-004), ARCHITECTURE (2.2, 3.2, 4.3) |
| Requirements | FR-CON-004 |

**The problem:** FR-CON-004 requires the console to support `pause`, `resume`, `speed`, and `tick` commands. Pause/resume/speed are Game Controller responsibilities (Godot layer), but the console routes commands through `ISimulationCommands` (simulation layer). The Architecture provides no path from simulation-layer console commands to Godot-layer time controls.

**Suggested resolution:** Either move time control to the simulation layer (simulation manages its own pause state), or add a separate `ITimeControl` interface that the console can access alongside `ISimulationCommands`.

**Resolution:** Added `ITimeControl` interface (Architecture Section 3.11) with `Tick()`, `Pause()`, `Resume()`, `IsPaused`, `Speed`, and `SetSpeed()`. Defined in the simulation layer; implemented by the Game Controller. Updated Section 2.2 to list `ITimeControl` implementation as a Game Controller responsibility. Updated Section 4.3 console data flow to show three-branch routing: queries → `ISimulationState`, policy → `ISimulationCommands`, time → `ITimeControl`. Added `ITimeControl.cs` to project structure. This also partially addresses M2 by clarifying that the console parses and routes to typed interface methods.

---

### ~~M2. Console Command Parsing Responsibility Is Contradictory~~ ✅ Resolved

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (3.2, 4.3) |
| Requirements | FR-CON-002, FR-CON-003 |

**The problem:** Architecture 4.3 (data flow) says "Console node parses command" — parsing happens in the Godot presentation layer. But Architecture 3.2 defines `ExecuteConsoleCommand(string command)` on `ISimulationCommands`, which accepts a raw string, implying the simulation engine does the parsing.

If the Console node parses first, it should call typed methods (like `SetSpendingLevel`), not pass a raw string. If the simulation parses, then the Console node is not parsing.

**Suggested resolution:** Clarify: the Console node handles input display, history, and tokenization. It then either (a) routes to typed `ISimulationCommands` methods for simulation commands and to the Game Controller for time commands, or (b) passes a raw string to a `CommandInterpreter` in the simulation layer. Remove the ambiguity by picking one and updating the data flow.

**Resolution:** Removed `ExecuteConsoleCommand(string command)` from `ISimulationCommands` (Section 3.2). The console node in the Godot layer owns all parsing; after parsing, it routes to typed methods on the appropriate interface: queries → `ISimulationState.QueryByPath()`, policy → `ISimulationCommands` typed methods (`SetTaxRate`, `SetSpendingLevel`, `SetSpendingAllocation`), time → `ITimeControl`. Section 4.3 already reflects this three-branch routing (updated in M1 resolution). Raw strings never cross the Godot boundary into the simulation engine, preserving the pure-library principle (Section 6.1). Note: if future requirements need arbitrary parameter setting beyond the three policy levers (e.g., a debug `set` command), the right approach is a typed `SetByPath(string path, object value)` method — the write counterpart to `QueryByPath` — not a raw string command. That decision is deferred and intersects with M10 (path schema unspecified).

---

### ~~M3. Multiple State Interfaces Referenced But Never Defined~~ ✅ Resolved

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

**Resolution:** Defined all six missing state interfaces in Architecture Section 3.3 under "Agent State Projections": `IGovernmentState` (FR-AGT-001), `ICentralBankState` (FR-AGT-002), `IBankingState` (FR-AGT-003), `IFirmSectorState`, `IHouseholdClassState`, and `IEconomicIndicators` (FR-SIM-004, 13 values from 12 bullet points). Added naming convention note to Section 3.1 explaining that `*State` interfaces are read-only projections for the UI/console, while agent interfaces extend them. `IFirm : IFirmSectorState` and `IHouseholdClass : IHouseholdClassState` so properties are defined once. `IBankingState` doc comment addresses L11 (single aggregate bank, plural naming is forward-compatible). `IEconomicIndicators.BondYields` defined as weighted average coupon rate per M7. Added state interface definition tasks and two RED tests to Implementation Plan Phase 3.

---

### ~~M4. IRandom Interface Never Implemented~~ ✅ Resolved

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (6.5), IMPLEMENTATION-PLAN (all phases) |
| Requirements | NFR-CQA-002 (deterministic tests) |

**The problem:** Architecture 6.5 lists `IRandom` as a core injectable dependency for "seeded random number generation for deterministic tests." No implementation phase creates this interface or its implementations. Phase 8's property-based tests depend on deterministic randomness via the seed parameter on `ISimulationFactory`, but the underlying `IRandom` is never built.

**Suggested resolution:** Add `IRandom` creation to Phase 0 (infrastructure) or Phase 2 (first phase that might need randomness). Define it alongside the other core interfaces.

**Resolution:** Added `IRandom` interface definition and `SeededRandom` narrative to Architecture Section 3.12 (Random Number Generation). Added `IRandom.cs` and `SeededRandom.cs` to project structure Section 5 under `src/Simulation/Core/`. Added Implementation Plan Phase 2 GREEN step 8 (define `IRandom` and implement `SeededRandom`). Amended Phase 8 GREEN step 5 to wire seed through `SeededRandom` and inject as `IRandom` into all components.

---

### ~~M5. "Sector" Balance Sheets (PRD) vs. Per-Agent Balance Sheets (Architecture)~~ ✅ Resolved

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-UI-003), ARCHITECTURE (3.1, 3.4) |
| Requirements | FR-UI-003 |

**The problem:** FR-UI-003 requires the UI to "display SFC balance sheets for each sector." In SFC accounting, "sector" means an aggregate: the government sector, the banking sector, the household sector, etc. But the Architecture defines `IBalanceSheet` per agent (each has an `OwnerId`), and `ISimulationState` returns `IReadOnlyList<IBalanceSheet> AllBalanceSheets` — a flat list of per-agent sheets.

There is no aggregation mechanism to produce sector-level balance sheets from per-agent data.

**Suggested resolution:** Add a method to `ISimulationState` like `IReadOnlyDictionary<string, IBalanceSheet> SectorBalanceSheets` that returns aggregated views, or document that the UI is responsible for aggregating per-agent sheets by agent type.

**Resolution:** Added `EconomicSector` enum (Government, CentralBank, Banking, Households, Firms) to Architecture Section 3.4. Added `IReadOnlyDictionary<EconomicSector, IBalanceSheet> SectorBalanceSheets` property to `ISimulationState` (Section 3.1) with aggregation rules: single-agent sectors (Government, CentralBank, Banking) return their agent's balance sheet directly; multi-agent sectors (Households, Firms) sum `Assets` and `Liabilities` across all agents in the group. Updated Section 6.3 tick-and-read diagram so `BalanceSheetPanel` reads `SectorBalanceSheets` instead of `AllBalanceSheets`. Updated Implementation Plan Phase 11 `BalanceSheetPanel` task to reference `SectorBalanceSheets`.

---

### ~~M6. PRD FR-SIM-002 Oversimplifies Money Circuit Access~~ ✅ Resolved

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-SIM-002, FR-AGT-003) |
| Requirements | FR-SIM-002 |

**The problem:** FR-SIM-002 states: "The private sector must only interact with deposit money" and "The central bank and Treasury must only interact with reserve money."

But commercial banks are private sector entities that necessarily interact with **both** circuits: they hold reserve accounts at the central bank (reserve money) and maintain deposit accounts for customers (deposit money). FR-AGT-003 explicitly requires banks to "hold a reserve account at the central bank" and "buy government bonds at auction" (which requires reserves).

**Suggested resolution:** Amend FR-SIM-002 to clarify that banks bridge both circuits. Something like: "Non-bank private sector agents (households, firms) must only interact with deposit money. The central bank and Treasury must only interact with reserve money. Commercial banks operate in both circuits, holding reserve accounts at the central bank and deposit accounts for customers."

**Resolution:** Amended FR-SIM-002 in the PRD to replace the two oversimplified circuit-access bullets with three precise rules: non-bank private agents (households, firms) → Deposits only, central bank and Treasury → Reserves only, commercial banks → both circuits. This aligns the PRD with Architecture Section 2.1 (line 85), which already had the correct rules from the C2 resolution.

---

### ~~M7. Bond Yields Indicator Requires Out-of-Scope Mechanics~~ ✅ Resolved

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-SIM-004, Section 4) |
| Requirements | FR-SIM-004 |

**The problem:** FR-SIM-004 lists "bond yields" as a required indicator. But Section 4 (Out of Scope) explicitly excludes "bond secondary market." Without a secondary market, bond yields are identical to the coupon rate set at auction — making the "yield" indicator either redundant or misleading.

**Suggested resolution:** Either define "bond yields" as the weighted average coupon rate across outstanding bonds (which is meaningful without a secondary market), or replace it with "average bond coupon rate" to avoid implying secondary market mechanics.

**Resolution:** Defined "bond yields" as the weighted average coupon rate across outstanding bonds — the government's average cost of debt. Formula: `Σ(bond.CouponRate × bond.FaceValue) / Σ(bond.FaceValue)`, returning 0 when no bonds are outstanding. This is meaningful without a secondary market because auction demand (FR-BND-001) produces varying coupon rates across issuances, and the weighted average shifts as old bonds mature and new ones are issued. The indicator is also pedagogically valuable: it demonstrates the MMT insight that bond rates are policy-influenced, since the CB buyer-of-last-resort effectively sets a rate ceiling. Updated FR-SIM-004 bullet in the PRD to clarify the definition. Expanded `IEconomicIndicators.BondYields` doc comment in Architecture Section 3.3 with formula, zero-bonds edge case, and pedagogical note.

---

### M8. Household Bond Participation Has No Architecture Support

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-BND-001), ARCHITECTURE (3.3), IMPLEMENTATION-PLAN (Phase 6) |
| Requirements | FR-BND-001 |

**The problem:** FR-BND-001 states: "Banks and high-income households must be able to bid" on government bonds. The Architecture provides no mechanism for household bond participation. The `IHouseholdClass` interface (Architecture 3.3) has no bond-related properties. Implementation Plan Phase 6 has no test for household bond bidding.

**Suggested resolution:** Add bond-related properties to `IHouseholdClass` (or its state interface). Add a test to Phase 6: `FrBnd001_HighIncomeHouseholds_CanBidOnBonds`.

---

### M10. QueryByPath Interface Completely Unspecified

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (3.1, 6.6), IMPLEMENTATION-PLAN (Phase 8, Phase 12) |
| Requirements | FR-CON-002, FR-UI-002 |

**The problem:** `ISimulationState.QueryByPath(string path)` returns `object`. Architecture 6.6 describes path-based state access ("all simulation state is queryable by a dot-separated path, e.g., `firms.agriculture.price`") and lists three consumers: the console, the chart data binding system, and future mod scripts. But the valid path space is never defined anywhere.

There is no schema or registry of valid paths. There is no specification of what types the returned `object` can be. There is no documented behavior for invalid paths (return null? throw?). The Implementation Plan Phase 8 test 11 (`FrSim003_QueryByPath_ReturnsCorrectValues`) tests this feature but with no specification of what paths should exist, the test cannot be written.

**Why it matters:** Three independent features depend on path queries working correctly. Without a path schema, each feature must guess what paths exist and hope they match the implementation. Console commands like `query firms.agriculture.price` will fail silently or throw if the path format doesn't match the implementation's conventions.

**Suggested resolution:** Add a path schema specification to Architecture 6.6. Define the root paths (e.g., `government.*`, `banks.*`, `firms.<sector>.*`, `households.<class>.*`, `indicators.*`), their leaf properties, and the return types. This can be a table or a tree listing.

---

### M11. No Architecture for Hierarchical Consumption

| Aspect | Detail |
|---|---|
| Documents | PRD (FR-AGT-004), ARCHITECTURE (2.1, 4.1), IMPLEMENTATION-PLAN (Phase 4) |
| Requirements | FR-AGT-004 |

**The problem:** FR-AGT-004 requires households to "consume according to hierarchical needs: survival → shelter → comfort → luxury." This is a core behavioral mechanic — it determines how income flows through the goods market and which sectors receive demand at different income levels.

The Architecture has no specification for this mechanism. There is no description of how goods map to need levels, how the hierarchy is evaluated (does a household fully satisfy survival before spending on shelter, or is there a blending function?), or how it interacts with the `GoodsMarket`. The tick data flow (4.1, Market Phase) says "Households purchase goods (hierarchical needs)" but provides no further detail. The `IHouseholdClass` interface (3.3) has `ConsumptionSpending` but no need-level breakdown.

Implementation Plan Phase 4 tests this (test 12: `FrSim003_HouseholdsBuySurvivalBeforeComfort`, test 13: `FrAgt004_LowIncomeHouseholds_HigherShareOnNecessities`) but the implementation in the GREEN step ("hierarchical needs purchasing, price elasticity, inventory sales") must invent the design.

**Why it matters:** This is comparable to C6 (inflation buffers) — a core mechanism referenced across documents but with no architectural specification. The need hierarchy determines consumption patterns, which determine demand, which determines production, pricing, and employment. Getting this wrong cascades through the entire model.

**Suggested resolution:** Add a hierarchical consumption component to the Architecture. Specify: how goods map to need levels (data-driven via `IDataProvider`), the evaluation order (strict priority vs. blending), how price elasticity varies by need level (FR-AGT-004), and how unmet needs affect household behavior.

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

### L10. SpendingAllocation Type Referenced But Never Defined

| Aspect | Detail |
|---|---|
| Documents | ARCHITECTURE (3.2) |
| Requirements | FR-CTL-001 |

`ISimulationCommands.SetSpendingAllocation(SpendingAllocation allocation)` references a `SpendingAllocation` type that is never defined in the Architecture. Is it a struct, class, or enum? What fields does it have? FR-CTL-001 requires allocation across infrastructure, public services, and direct transfers — so presumably it has three fields that sum to 100%. But this is left implicit.

---

### L11. Single Aggregate Bank vs. Plural Interfaces

| Aspect | Detail |
|---|---|
| Documents | PRD (Section 5), ARCHITECTURE (3.1, 3.3) |
| Requirements | FR-AGT-003 |

The out-of-scope table says "multiple competing banks" is deferred, implying the MVP has one aggregate bank. But the Architecture uses plural language throughout: the `ISimulationState` property is `IBankingState Banks` (plural), FR-AGT-003 uses "commercial banks must hold deposits," and the Architecture 2.1 narrative says "CommercialBank" (singular agent name) while the interface assumes a plural. L8 flags this as a terminology issue, but the deeper question is whether the MVP models one bank entity or multiple — this affects Phase 3 agent creation (how many bank agents to instantiate) and Phase 5/6 behavior (do banks compete for deposits or loans, or is there one aggregate?).

**Suggested resolution:** Add an explicit statement to the Architecture: "The MVP models a single aggregate commercial bank. The `Banks` property and plural references are forward-compatible naming for post-MVP multi-bank support." Or, if multiple banks are intended for MVP, remove "multiple competing banks" from the out-of-scope list.

---

### L12. No Error Recovery Strategy for SFC Failures

| Aspect | Detail |
|---|---|
| Documents | PRD (NFR-DTA-001), ARCHITECTURE (4.1) |
| Requirements | NFR-DTA-001 |

NFR-DTA-001 states: "If an imbalance is detected, it must be logged and flagged as a bug." The Architecture runs `Ledger.CheckConsistency()` in the Accounting Phase of each tick (Section 4.1). But neither document specifies what happens to the simulation after a failure is detected. Does the simulation halt? Continue with the imbalance? Roll back the tick? Throw an exception that propagates to the Game Controller?

This matters because the Game Controller needs to know how to handle the failure — display an error to the player, crash gracefully, or silently log. The test harness (`AssertSfcConsistent()`) implies it throws, but the production behavior is unspecified.

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

FR-SIM-001 (SFC accounting), FR-SIM-003 (tick processing), FR-AGT-002 (central bank), FR-AGT-005 (firms), FR-PRC-001 (cost-plus pricing), FR-LBR-003 (unemployment), FR-BNK-001 (endogenous money creation), FR-BNK-002 (creditworthiness), FR-BNK-003 (loan types), FR-BNK-004 (debt service/default), FR-BND-002 (bond properties), FR-CTL-002 (time controls), FR-GMD-001 (sandbox), FR-GMD-002 (scenario), FR-MOD-001 (data-driven design), FR-MOD-002 (data schema), FR-MOD-003 (base game as mod), FR-CON-001 (console access), FR-CON-003 through FR-CON-006 (console).

### Requirements With Gaps

| Requirement | Issue(s) |
|---|---|
| FR-SIM-002 | ~~C2 (ILedger interface)~~ ✅, ~~M6 (oversimplified circuit access)~~ ✅ |
| FR-SIM-004 | M3 (IEconomicIndicators undefined), ~~M7 (bond yields)~~ ✅, L2 (missing indicator tests) |
| FR-AGT-001 | M3 (IGovernmentState undefined) |
| FR-AGT-003 | L11 (single vs. plural bank ambiguity) |
| FR-AGT-004 | M11 (no architecture for hierarchical consumption), L2 (missing debt capacity and elasticity tests) |
| FR-PRC-002 | ~~C6 (no architecture, missing tests for buffer 3)~~ ✅ |
| FR-PRC-003 | L2 (sector-specific price tracking untested) |
| FR-LBR-001 | L2 (wage stickiness and determinants untested) |
| FR-LBR-002 | L2 (cross-sector mobility untested) |
| FR-BND-001 | M8 (household bond participation) |
| FR-INV-001 | C5 (absent from architecture) |
| FR-INV-002 | C5 (absent from architecture), L2 (capital goods from industry untested) |
| FR-TIM-001 | C4 (no architecture component), L2 (lag ranges untested) |
| FR-TIM-002 | C4 (no architecture component), L2 (most lag types untested) |
| FR-TIM-003 | C4 (no data source for UI) |
| FR-CTL-001 | L10 (SpendingAllocation type undefined) |
| FR-CON-002 | M10 (QueryByPath path schema unspecified) |
| FR-UI-007 | C4 (no data source for pipeline display) |
| NFR-DTA-001 | L12 (no error recovery strategy for SFC failures) |
