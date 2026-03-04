# Implementation Plan

This document breaks the MVP into concrete, ordered implementation phases. Each phase has specific deliverables, dependencies, and a definition of done.

## Development Methodology

### Test-Driven Development (TDD)

All simulation engine code (Phases 0–8) follows the **red-green-refactor** cycle:

1. **RED:** Write a failing test that defines the desired behavior
2. **GREEN:** Write the minimum code to make the test pass
3. **REFACTOR:** Clean up the code while keeping all tests green

Tests are written before implementation. Every phase lists its tests first, then the implementation that makes them pass.

### No Hardcoded Defaults

Per FR-MOD-001 and MODDING.md: **no economic parameters may be hardcoded — not even as temporary defaults to be replaced later.** The data loading system is built in Phase 1 so that all subsequent phases load their configuration from `IDataProvider`. Tests use `InMemoryDataProvider` to supply data without needing files on disk.

## Phase Overview

```
Phase 0:  Project Setup & Test Infrastructure
    │
    ▼
Phase 1:  Data Layer & JSON Loading
    │
    ▼
Phase 2:  Accounting Foundation
    │
    ▼
Phase 3:  Economic Agents (Core)
    │
    ▼
Phase 4:  Markets & Pricing
    │
    ▼
Phase 5:  Banking & Credit
    │
    ▼
Phase 6:  Government Bonds
    │
    ▼
Phase 7:  Investment & Lags
    │
    ▼
Phase 8:  Tick Engine Integration
    │
    ▼
Phase 9:  Godot Project & Game Controller
    │
    ▼
Phase 10: UI — Policy Panel & Time Controls
    │
    ▼
Phase 11: UI — Charts & Data Display
    │
    ▼
Phase 12: Console
    │
    ▼
Phase 13: Game Modes (Sandbox & Scenario)
    │
    ▼
Phase 14: Map View
    │
    ▼
Phase 15: Balancing & Polish
```

---

## Phase 0: Project Setup & Test Infrastructure

**Goal:** Repository structure, build system, test infrastructure, and tooling ready.

**Tasks:**
1. Create .NET solution file (`game.sln`)
2. Create Simulation class library project (`src/Simulation/Simulation.csproj`)
3. Create Godot project (`src/Game/` with `project.godot` and `Game.csproj`)
4. Configure `Game.csproj` to reference `Simulation.csproj`
5. Create test project (`tests/Simulation.Tests/Simulation.Tests.csproj`) with xUnit and a property-based testing library (FsCheck or similar)
6. Create `InMemoryDataProvider` skeleton (implements `IDataProvider`, stores data in dictionaries)
7. Create `SimulationTestHarness` skeleton (convenience methods for test setup)
8. Create test data directories: `tests/Simulation.Tests/TestData/{Minimal,FullBase,InvalidData,Scenarios}`
9. Create `data/base/` directory structure with placeholder JSON files
10. Create `mods/.gitkeep`
11. Add `.gitignore` for C#, Godot, and .NET artifacts
12. Write one smoke test: `SmokeTest_ProjectBuilds_TestFrameworkRuns`
13. Verify: solution builds, tests run (smoke test passes), Godot project opens

**Dependencies:** None

**Definition of done:** `dotnet build` succeeds, `dotnet test` runs with smoke test passing, test infrastructure classes exist, Godot opens the project without errors.

---

## Phase 1: Data Layer & JSON Loading

**Goal:** The data loading system works end-to-end. All data model classes exist. Base game JSON files are created and validated. Test fixtures are in place.

**Test fixtures needed:** Minimal valid dataset (1 sector, 1 household class), invalid data files (missing fields, bad types, negative values, empty files).

**TDD Cycle:**

**RED — write failing tests first:**
1. `FrMod001_LoadSectorData_ReturnsDeserializedSectors`
2. `FrMod001_LoadHouseholdData_ReturnsDeserializedClasses`
3. `FrMod001_LoadAllDataTypes_EachReturnsValidObjects` (goods, firms, banks, government, production, parameters, simulation config)
4. `FrMod002_MissingRequiredField_ValidationErrorWithFieldName`
5. `FrMod002_NegativeMarkupValue_ValidationErrorWithPath`
6. `FrMod002_InvalidJsonSyntax_ClearParseError`
7. `FrMod002_EmptyFile_ClearError`
8. `FrMod002_ValidBaseGameData_PassesAllValidation`
9. `FrMod003_BaseGameLoadsViaSamePathAsMods`
10. `FrMod001_InMemoryProvider_LoadsRegisteredData`
11. `FrMod001_InMemoryProvider_MissingDataThrows`
12. `FrMod001_ListFiles_ReturnsAllFilesInDirectory`
13. `FrMod001_ResolvePath_ResolvesToBaseDataDirectory`
14. `DataValidator_ValidateAll_ChecksEveryDataFile`
15. `DataValidator_ValidSectorData_ReturnsSuccess`
16. `DataValidator_InvalidSectorData_ReturnsErrors`
17. `DataValidator_ValidFullDataset_ReturnsSuccess`
18. `DataModels_RoundTrip_SerializeDeserializeMatch`

**GREEN — implement to make tests pass:**
1. Define data model classes: `SectorData`, `GoodsData`, `HouseholdData`, `FirmData`, `BankData`, `GovernmentData`, `ProductionData`, `ParametersData`, `SimulationConfigData`, `ScenarioData`
2. Implement `IDataProvider` interface
3. Implement `JsonDataProvider` — load JSON from disk, deserialize to data model classes, resolve paths through `data/base/`
4. Implement `IDataValidator` interface and `DataValidator` — validate required fields, value ranges, referential integrity between data files
5. Complete `InMemoryDataProvider` — register data objects in memory, return on request
6. Create all `data/base/` JSON files:
   - `economy/sectors.json`, `economy/goods.json`, `economy/production.json`, `economy/parameters.json`
   - `agents/households.json`, `agents/firms.json`, `agents/banks.json`, `agents/government.json`
   - `scenarios/sandbox.json`, `scenarios/full-employment.json`
   - `config/simulation.json`, `config/ui.json`
7. Create test fixtures:
   - `TestData/Minimal/` — 1 sector, 1 household class, minimal valid data
   - `TestData/FullBase/` — copy of `data/base/`
   - `TestData/InvalidData/` — malformed files for each error type

**REFACTOR:** Extract common validation logic. Ensure all error messages include file path and field name.

**Dependencies:** Phase 0

**Definition of done:** All 18 tests pass. Base game data files exist and validate. Test fixtures exist for all categories. `InMemoryDataProvider` works for test use.

---

## Phase 2: Accounting Foundation

**Goal:** The SFC double-entry bookkeeping system works. Transactions can be recorded and verified.

**Test fixtures needed:** Minimal data from `InMemoryDataProvider` with account structure definitions.

**TDD Cycle:**

**RED — write failing tests first:**
1. `FrSim001_RecordTransaction_UpdatesBothAccounts`
2. `FrSim001_BalanceSheet_AssetsMinusLiabilitiesEqualsNetPosition`
3. `FrSim001_AfterValidTransactions_SfcCheckPasses`
4. `FrSim001_AfterManualBalanceEdit_SfcCheckFails`
5. `FrSim002_GovernmentSpending_CreatesReservesAndDeposits`
6. `FrSim002_Taxation_DestroysDepositsAndReserves`
7. `FrSim002_SpendThenTax_NetBalancesCorrect`
8. `FrSim001_TransactionWithZeroAmount_Rejected`
9. `FrSim001_TransactionWithInvalidAccount_Rejected`
10. `FrSim001_GetTransactionsByTick_ReturnsCorrectSubset`
11. `FrSim001_GetTransactionsByAccount_ReturnsCorrectSubset`

**GREEN — implement to make tests pass:**
1. Implement `Transaction` record (from, to, amount, category, tick, description)
2. Implement `BalanceSheet` (assets dictionary, liabilities dictionary, net position)
3. Implement `Account` (id, balance, type: asset/liability)
4. Implement `Ledger` with two sub-ledgers: reserves ledger (central bank money) and deposits ledger (bank deposits)
5. Implement `RecordTransaction()` — validates and records double-entry
6. Implement `SfcChecker` — verifies all balances sum to zero
7. Account structure initialized from `IDataProvider` (account types and names come from data)
8. Define `IRandom` interface and implement `SeededRandom` (wraps `System.Random` with constructor seed, per Architecture §3.12). No dedicated test — validated through first consumer.

**REFACTOR:** Ensure transaction validation covers all edge cases. Extract common patterns.

**Dependencies:** Phase 1 (needs `IDataProvider` for account structure)

**Definition of done:** All tests pass. A sequence of government spending + taxation results in correct balances and passes SFC check. All account structure comes from data, not hardcoded values.

---

## Phase 3: Economic Agents (Core)

**Goal:** All agent types exist with balance sheets and basic state. No behavior yet.

**Test fixtures needed:** Minimal data with agent definitions (1 sector, 1 household class, 1 bank, government, central bank).

**TDD Cycle:**

**RED — write failing tests first:**
1. `FrAgt001_GovernmentCreatedFromData_HasCorrectInitialState`
2. `FrAgt002_CentralBankCreatedFromData_HasCorrectPolicyRate`
3. `FrAgt003_BankCreatedFromData_HasCorrectSpreadAndThresholds`
4. `FrAgt004_HouseholdCreatedFromData_HasCorrectClassProperties`
5. `FrAgt005_FirmCreatedFromData_HasCorrectSectorProperties`
6. `FrAgt001_AgentBalanceSheets_RegisteredInLedger`
7. `FrAgt001_AgentRegistry_LookUpByType_ReturnsCorrectAgents`
8. `FrAgt001_AgentRegistry_LookUpById_ReturnsCorrectAgent`
9. `FrAgt004_MultipleHouseholdClasses_EachHasDifferentProperties`
10. `FrAgt005_MultipleFirmSectors_EachHasDifferentProperties`
11. `FrMod001_ChangedHouseholdDataValue_AgentReflectsChange`
12. `FrAgt001_GovernmentState_ExposesAllRequiredProperties`
13. `FrAgt003_BankingState_ExposesAllFiveTrackedValues`

**GREEN — implement to make tests pass:**
1. Define `IAgent` interface (id, type, balance sheet)
2. Implement `Government` agent (treasury account, spending level, allocation, tax rate — all from `IDataProvider`)
3. Implement `CentralBank` agent (reserve accounts, policy rate — from `IDataProvider`)
4. Implement `CommercialBank` agent (reserves, deposits, loans, bonds, equity — from `IDataProvider`)
5. Implement `Household` agent (class id, population, deposits, debts, employment status, reservation wage — from `IDataProvider`)
6. Implement `Firm` agent (sector id, deposits, capital, employees, inventory, wage, price, productivity — from `IDataProvider`)
7. Implement `AgentRegistry` — stores and retrieves agents by type/id
8. Wire agents to accounting system (each agent owns accounts in the ledger)
9. Define `IGovernmentState`, `ICentralBankState`, `IBankingState` interfaces (Architecture §3.3)
10. Define `IFirmSectorState`, `IHouseholdClassState` as base interfaces; make `IFirm : IFirmSectorState`, `IHouseholdClass : IHouseholdClassState`
11. Define `IEconomicIndicators` interface (implementation deferred to Phase 7/8 when indicator calculation is built)

**REFACTOR:** Extract common agent initialization patterns.

**Dependencies:** Phase 2 (needs Ledger), Phase 1 (needs `IDataProvider`)

**Definition of done:** All tests pass. All agent types instantiate with properties from data provider. Balance sheets wired to the ledger. Registry lookups work. Test 11 proves that changing data changes agent state — no hardcoded defaults.

---

## Phase 4: Markets & Pricing

**Goal:** Firms produce goods, set prices, households buy goods. Labor market matches workers to jobs.

**Test fixtures needed:** Minimal data with sector definitions, goods, household needs hierarchy, markup rates, productivity values.

**TDD Cycle:**

**RED — write failing tests first:**
1. `FrLbr001_FirmPostsJobs_LaborMarketTracksOpenings`
2. `FrLbr002_WorkerAboveReservationWage_AcceptsJob`
3. `FrLbr002_WorkerBelowReservationWage_RejectsJob`
4. `FrLbr001_MultipleJobs_WorkersPreferHigherPaying`
5. `FrLbr003_InsufficientJobs_UnemploymentEmerges`
6. `FrPrc001_UnitLaborCost_EqualsWagesDividedByOutput`
7. `FrPrc001_CostPlusMarkup_CalculatedCorrectly`
8. `FrPrc001_DemandAboveCapacity_MarkupIncreases`
9. `FrPrc001_DemandBelowCapacity_MarkupDecreases`
10. `FrPrc002_WageRiseMatchesProductivityGain_PriceUnchanged`
11. `FrPrc002_IdleCapacity_SpendingIncreaseRaisesOutputNotPrices`
12. `FrPrc002_CostRiseWithMarginSlack_FirmAbsorbsViaSmallerMarkup`
13. `FrPrc002_CostRiseExceedsMarginSlack_PriceRisesByResidual`
14. `FrPrc002_AllThreeBuffersExhausted_InflationOccurs`
15. `FrSim003_HouseholdsBuySurvivalBeforeComfort`
16. `FrAgt004_LowIncomeHouseholds_HigherShareOnNecessities`
17. `FrSim001_AllPurchases_RecordedAsLedgerTransactions`
18. `FrSim001_AfterFullMarketCycle_SfcCheckPasses`
19. `FrMod001_ChangedMarkupInData_ProducesHigherPrices`
20. `FrMod001_ChangedNeedsHierarchy_ChangesConsumptionPattern`

**GREEN — implement to make tests pass:**
1. Implement `LaborMarket` — job posting, matching, employment tracking (parameters from `IDataProvider`)
2. Implement `ProductionEngine` — demand estimation, input checking, production (parameters from `IDataProvider`)
3. Implement `PricingEngine` — unit labor cost, unit material cost, markup with demand
   adjustment, three-buffer inflation logic per Architecture §3.10 (markup rates, minimum
   markup, and capacity thresholds from `IDataProvider`)
4. Implement `GoodsMarket` — disposable income, hierarchical needs purchasing, price elasticity, inventory sales (needs hierarchy from `IDataProvider`)
5. Wire all transactions through the Ledger

**REFACTOR:** Extract market interfaces for testability. Simplify pricing calculation chain.

**Dependencies:** Phase 3

**Definition of done:** All tests pass. A full cycle works: firms hire → produce → price → households buy. All transactions in ledger. SFC check passes. Tests 16–17 prove data-driven behavior.

---

## Phase 5: Banking & Credit

**Goal:** Banks create money through lending. Debt service and defaults work.

**Test fixtures needed:** Banking parameters (spreads, risk thresholds, loan terms), creditworthiness criteria from data.

**TDD Cycle:**

**RED — write failing tests first:**
1. `FrBnk001_LoanCreated_DepositsAndLoansIncrease`
2. `FrBnk001_LoanCreated_TotalMoneyStockIncreases`
3. `FrBnk001_LoanRepaid_DepositMoneyDestroyed`
4. `FrBnk001_InterestPayment_StaysInSystemAsBankRevenue`
5. `FrBnk001_AfterLendingCycle_SfcCheckPasses`
6. `FrBnk002_CreditworthyBorrower_GetsLoan`
7. `FrBnk002_HighDebtToIncome_LoanRejected`
8. `FrBnk002_LendingRate_EqualsCbRatePlusSpreadPlusRisk`
9. `FrBnk003_ConsumerLoan_ShortTermUnsecured`
10. `FrBnk003_Mortgage_LongTermSecured`
11. `FrBnk003_BusinessLoan_ForFirmInvestment`
12. `FrBnk004_BorrowerCannotPay_DefaultTriggered`
13. `FrBnk004_Default_LoanWrittenOff`
14. `FrMod001_ChangedBankSpread_AffectsLendingRate`

**GREEN — implement to make tests pass:**
1. Implement creditworthiness assessment (income, debt-to-income ratio, collateral, risk premium — thresholds from `IDataProvider`)
2. Implement loan creation (deposit creation, loan receivable, ledger recording — terms from `IDataProvider`)
3. Implement loan types: consumer loans, mortgages, business loans (properties from `IDataProvider`)
4. Implement debt service (monthly payment, principal destruction, interest transfer)
5. Implement defaults (detection, write-off, borrower debt removal)

**REFACTOR:** Extract common loan lifecycle logic. Clean up bank balance sheet updates.

**Dependencies:** Phase 3

**Definition of done:** All tests pass. Banks can lend, money is created, debt is serviced, defaults are handled. SFC consistency maintained throughout. All parameters from data.

---

## Phase 6: Government Bonds

**Goal:** Government issues bonds via auction. Interest is paid. Central bank backstops.

**Test fixtures needed:** Bond parameters (maturity, face value ranges), auction rules from data.

**TDD Cycle:**

**RED — write failing tests first:**
1. `FrBnd001_BondAuction_AllocatesToHighestBidders`
2. `FrBnd001_InsufficientDemand_CentralBankBuysRemainder`
3. `FrBnd002_BondInterestPayment_FlowsCorrectly`
4. `FrBnd002_BondMaturity_FaceValueRepaid`
5. `FrBnd001_BondPurchase_SfcConsistent`
6. `FrBnd002_InterestPayment_SfcConsistent`
7. `FrBnd002_FullBondLifecycle_SfcConsistent`
8. `FrMod001_ChangedBondMaturity_AffectsRepaymentTiming`

**GREEN — implement to make tests pass:**
1. Implement bond data structure (face value, coupon rate, maturity, holder — from `IDataProvider`)
2. Implement `BondMarket.RunAuction()` (issuance, bidding, allocation, CB backstop — parameters from `IDataProvider`)
3. Implement bond accounting (purchase, interest, maturity flows through Ledger)
4. Implement central bank bond operations (buyer of last resort, reserve crediting)

**REFACTOR:** Simplify auction mechanics. Extract bond accounting patterns.

**Dependencies:** Phase 3, Phase 5 (banks need to exist)

**Definition of done:** All tests pass. Bond auctions run, interest is paid, CB backstops. All SFC-consistent. All parameters from data.

---

## Phase 7: Investment & Lags

**Goal:** Public and private investment affect capacity. Policy changes have realistic delays.

**Test fixtures needed:** Investment parameters (depreciation rates, capacity multipliers), lag durations from data.

**TDD Cycle:**

**RED — write failing tests first:**
1. `FrInv001_InfrastructureSpending_IncreasesCapacityAfterLag`
2. `FrInv001_PublicServicesSpending_IncreasesProductivityAfterLag`
3. `FrInv001_PublicCapital_DepreciatesOverTime`
4. `FrInv002_FirmInvestment_IncreasesCapacity`
5. `FrInv002_CapitalGoods_DepreciateOverTime`
6. `FrTim001_TaxRateChange_TakesEffectAfterOneTick`
7. `FrTim001_SpendingChange_TakesEffectAfterCorrectDelay`
8. `FrTim001_AllocationChange_TakesEffectAfterCorrectDelay`
9. `FrTim001_PolicyPipeline_TracksAllPendingChanges`
10. `FrTim002_WageAdjustment_GradualOverMultipleTicks`
11. `FrTim002_PriceAdjustment_GradualOverMultipleTicks`
12. `FrMod001_ChangedDepreciationRate_AffectsCapitalDecay`
13. `FrMod001_ChangedLagDuration_AffectsPolicyTiming`

**GREEN — implement to make tests pass:**
1. Implement public investment per Architecture Section 3.9 (infrastructure → capacity, public services → productivity — rates from `IDataProvider`)
2. Implement private investment per Architecture Section 3.9 (demand-driven, funded from profits + loans — thresholds from `IDataProvider`)
3. Implement capital depreciation (rates from `IDataProvider`)
4. Implement `IPolicyPipeline` and `PendingPolicy` per Architecture Section 3.8 (queue, pipeline tracking, flush — durations from `IDataProvider`)
5. Implement economic lags (sticky wages, gradual prices, hiring/firing delays — from `IDataProvider`)

**REFACTOR:** Extract lag system into reusable pipeline. Ensure investment and depreciation are symmetric.

**Dependencies:** Phase 4 (production system), Phase 5 (firms borrow for investment)

**Definition of done:** All tests pass. Investment changes capacity over time. Policy lags work correctly. Pipeline tracks pending changes. All durations and rates from data.

---

## Phase 8: Tick Engine Integration

**Goal:** All phases run together in a complete monthly tick. The economy simulates end-to-end.

**Test fixtures needed:** Full base game data (`TestData/FullBase/`), scenario-specific fixtures for targeted tests.

**TDD Cycle:**

**RED — write failing tests first:**
1. `FrSim003_TickPhases_ExecuteInCorrectOrder`
2. `FrSim004_AfterTick_AllIndicatorsCalculated`
3. `FrSim004_EmploymentRate_CalculatedCorrectly`
4. `FrSim004_InflationRate_CalculatedAsPercentChangeInPriceLevel`
5. `FrSim001_AfterEveryTick_SfcIdentityHolds`
6. `FrSim001_GovernmentSpending_IncreasesPrivateSectorBalance`
7. `FrSim001_Taxation_DecreasesPrivateSectorBalance`
8. `FrPrc002_SpendingIntoSlack_IncreasesOutputNotPrices`
9. `FrPrc002_SpendingBeyondCapacity_CausesInflation`
10. `FrPrc002_ProgressiveBufferExhaustion_InflationEmergesGradually`
11. `FrLbr003_InsufficientSpending_CausesUnemployment`
12. `FrSim003_QueryByPath_ReturnsCorrectValues`
13. `FrCtl001_SetSpendingLevel_AcceptedAndQueued`
14. `FrCtl001_SetTaxRate_AcceptedAndQueued`

**Property-based tests:**
15. `Property_AnySeedAnyPolicySequence_SfcHoldsEveryTick`
16. `Property_AnyValidConfig_NoNegativePrices`
17. `Property_AnyValidConfig_EmploymentRateInZeroToOne`
18. `Property_AnyValidConfig_MoneyStockConsistent`

**Integration test:**
19. `Integration_120TickSimulation_SfcConsistentEveryTick_SensibleDynamics`

**GREEN — implement to make tests pass:**
1. Implement `TickEngine` — orchestrate phases in order: Government → Production → Market → Financial → Accounting
2. Implement `IndicatorCalculator` — employment, inflation, GDP, government balance, private balance, capacity utilization, ULC, private debt, all other PRD indicators
3. Implement `SimulationState` — read-only state container, path-based query system (`QueryByPath`)
4. Implement `SimulationCommands` — policy change handlers, console command routing
5. Implement `ISimulationFactory` — create simulation from `IDataProvider` + optional seed, wire seed to `SeededRandom` and inject as `IRandom` (Architecture §3.12) into all components

**REFACTOR:** Optimize tick execution. Extract indicator calculation patterns.

**Dependencies:** Phases 2–7

**Definition of done:** All tests pass including property-based tests. A 120-tick (10 year) simulation runs end-to-end with sensible economic dynamics. SFC consistency holds every tick. All 4 property-based invariants pass across random seeds.

---

## Phase 9: Godot Project & Game Controller

**Goal:** Godot project runs and connects to the simulation engine.

**Tasks:**
1. Set up Godot project with C# support
2. Configure project references (Game → Simulation)
3. Implement `GameController` node:
   - Initialize simulation engine via `ISimulationFactory` with `JsonDataProvider`
   - Manage game loop (advance ticks based on speed, handle pause)
   - Expose simulation state to child nodes
   - Route commands from UI to simulation
4. Implement time management:
   - Pause/resume state
   - Speed settings (1x, 2x, 5x)
   - Map game time to real time (e.g., 1x = 1 tick per second)
5. Create main scene with GameController as root
6. Verify simulation runs inside Godot

**Testing:** Manual verification — simulation ticks advance in Godot, speed/pause controls respond. The simulation logic is already tested headlessly in Phases 2–8.

**Dependencies:** Phase 8

**Definition of done:** Godot runs, simulation ticks advance, speed and pause controls work.

---

## Phase 10: UI — Policy Panel & Time Controls

**Goal:** Player can control the simulation through the UI.

**Tasks:**
1. Create `PolicyPanel` scene and script:
   - Slider for government spending level
   - Three sliders for spending allocation (infrastructure, services, transfers) — constrained to sum to 100%
   - Slider for tax rate
   - Labels showing current values
   - Signals connected to GameController
2. Create `HUD` scene and script:
   - Current date display (month/year)
   - Play/pause button
   - Speed selector (1x, 2x, 5x)
   - Tick advance button (when paused)
3. Wire policy changes from UI → GameController → SimulationCommands
4. Visual feedback for pending policy changes (in pipeline indicator)

**Testing:** Manual verification + console-based integration scripts that set policy values and verify the simulation state changes. UI code is harder to TDD, but the underlying simulation commands are fully tested.

**Dependencies:** Phase 9

**Definition of done:** Player can adjust all policy levers and time controls via UI. Changes flow to simulation correctly.

---

## Phase 11: UI — Charts & Data Display

**Goal:** Player can see the economy's state through charts and data panels.

**Tasks:**
1. Evaluate/select a chart solution (Godot native, custom drawing, or lightweight library)
2. Implement `ChartPanel`:
   - Line chart component with scrolling time window
   - Support multiple data series per chart
   - Create charts for: employment rate, inflation rate, GDP, government balance, private savings, sector output
3. Implement `BalanceSheetPanel`:
   - Read `ISimulationState.SectorBalanceSheets` (keyed by `EconomicSector`)
   - Table display for each sector's balance sheet
   - Assets, liabilities, net position columns
   - Updated each tick
4. Implement resource utilization bars:
   - Per-sector capacity utilization bar
   - Color coded: green (slack), yellow (moderate), red (near capacity)
5. Layout all panels in the main scene

**Testing:** Manual verification — charts render, update per tick, display correct data from simulation state. Chart data correctness is guaranteed by the tested `IndicatorCalculator` and `SimulationState`.

**Dependencies:** Phase 9

**Definition of done:** All key indicators visible in charts. Balance sheets displayable. Utilization bars update with simulation.

---

## Phase 12: Console

**Goal:** In-game console with query, write, and time commands.

**Tasks:**
1. Create `ConsolePanel` scene:
   - Toggle visibility with `~` key
   - Text input field at bottom
   - Scrollable output history above
   - Semi-transparent overlay
2. Implement command parser:
   - Split command into verb + arguments
   - Route to appropriate handler
3. Implement query commands:
   - `query <path>` — reads from SimulationState.QueryByPath()
   - `balance <sector>` — formats and displays balance sheet
   - `sfc check` — runs consistency check
4. Implement write commands:
   - `set government.spending <amount>`
   - `set government.allocation.<target> <pct>`
   - `set government.tax_rate <rate>`
   - `spend <amount>` / `tax <amount>` — one-time operations
5. Implement time commands:
   - `pause`, `resume`, `speed <n>`, `tick`, `tick <n>`
6. Implement `help` system
7. Implement `run <filename>` script execution
8. Implement command history (up/down arrow)

**Testing:** Manual verification + console script integration tests. The command parser can be unit tested (pure string → command mapping). Query results are verified against the tested `SimulationState`.

**Dependencies:** Phase 9, Phase 8 (needs SimulationState query system)

**Definition of done:** Console opens/closes, all MVP commands work, scripts execute, help displays.

---

## Phase 13: Game Modes (Sandbox & Scenario)

**Goal:** Both sandbox and scenario modes are playable.

**Tasks:**
1. Implement sandbox mode:
   - No win/lose conditions
   - Score dashboard with all key metrics
   - Free play with all controls available
2. Scenario data format already defined in Phase 1 (`ScenarioData` model, `scenarios/*.json` files)
3. Implement scenario loader:
   - Read scenario JSON via `IDataProvider`
   - Initialize simulation to scenario starting state
   - Track objective progress
4. Implement win/lose detection:
   - Check objectives each tick
   - Check fail conditions each tick
   - Trigger appropriate UI feedback
5. Create `ScenarioUI`:
   - Objective display and progress
   - Time remaining
   - Win/lose screen
6. Create one complete scenario:
   - "Full Employment Challenge": achieve 95% employment, keep inflation below 5%, 10 year time limit
   - Fail: hyperinflation (>25%) or employment collapse (<50%)

**Testing:** Manual verification for UI. Scenario loading and win/lose logic can be tested headlessly using `SimulationTestHarness` with scenario fixtures.

**Dependencies:** Phase 10, Phase 11 (UI needs to exist)

**Definition of done:** Player can start sandbox or scenario mode. Scenario can be won or lost. Score dashboard works in sandbox.

---

## Phase 14: Map View

**Goal:** Cosmetic single-province map establishing visual direction.

**Tasks:**
1. Create simple 2D map asset (single province, modern/contemporary style)
2. Implement `MapView` scene:
   - Display map texture
   - Province boundary rendering
   - Basic zoom/pan
3. Add data overlays (optional MVP stretch):
   - Employment heat map coloring
   - Economic activity indicator
4. Integrate into main scene layout

**Testing:** Manual verification — map renders, zoom/pan works.

**Dependencies:** Phase 9

**Definition of done:** Map displays in game. Basic visual representation of the single province.

---

## Phase 15: Balancing & Polish

**Goal:** The economic model produces realistic, playable dynamics.

**Tasks:**
1. Tune economic parameters in `data/base/` JSON files:
   - Starting values for all sectors, classes, resources
   - Markup rates, productivity values, depreciation rates
   - Consumption propensities, reservation wages
   - Investment thresholds, lending criteria
2. Run extended simulations (100+ years) and check for:
   - Stability (does the economy reach a steady state without intervention?)
   - Responsiveness (do policy changes have visible effects?)
   - Realism (do dynamics match MMT predictions?)
   - Edge cases (what happens at extreme settings?)
3. Add parameter sweep tests: systematically vary key parameters and verify behavioral differences
4. Add 100+ year (1200+ tick) stability verification test
5. Verify that tuning `data/base/` JSON produces expected behavioral differences without code changes
6. UI/UX improvements:
   - Chart readability
   - Panel layout optimization
   - Clear labeling and tooltips
   - Visual polish
7. Bug fixing from playtesting
8. Write console scripts for common test scenarios

**Dependencies:** All previous phases

**Definition of done:** A new player can start, understand the controls, observe meaningful economic dynamics, and complete the scenario. Extended simulations are stable. Key MMT predictions hold (deficit = private surplus, spending into slack doesn't cause inflation, etc.). Parameter sweep tests pass.

---

## Risk Areas

| Risk | Mitigation |
|---|---|
| Economic model instability (runaway values, crashes) | TDD catches accounting errors immediately. SFC consistency check every tick. Property-based tests verify invariants across random seeds. |
| Balancing difficulty (unrealistic dynamics) | All parameters in data files — iterate without code changes. Parameter sweep tests catch regressions. 100+ year stability tests. |
| Godot C# integration issues | Keep simulation as pure C# library. Minimize Godot-specific code. Simulation is fully tested headlessly before Godot integration. |
| Scope creep | Strict MVP scope defined. Features deferred to post-MVP are documented. |
| Performance (slow ticks at scale) | Population groups (not individual agents) keep computation manageable. Profile early if tick time exceeds 100ms. |
| Chart rendering in Godot | Evaluate chart solutions early in Phase 11. Custom drawing with Godot's `_Draw()` is always a fallback. |
| TDD slowing early development | Pure C# simulation is ideal for TDD — no UI, no framework dependencies, fast test execution. The overhead is minimal and the safety net is significant. |
| Test fixtures diverging from base data | `TestData/FullBase/` is a copy of `data/base/`. CI check verifies they stay in sync. |
