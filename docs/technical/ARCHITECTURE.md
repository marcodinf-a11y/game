# Architecture Document

## 1. System Overview

The game is structured as three independent layers connected through clean interfaces. The critical architectural decision is that the **simulation engine is a pure C# library with no Godot dependencies**, making it testable, portable, and modular.

```
┌──────────────────────────────────────────────────────────────────┐
│                        GODOT SCENE TREE                          │
│  ┌────────────┐  ┌────────────┐  ┌──────────┐  ┌─────────────┐  │
│  │ Map View   │  │ Charts     │  │ Policy   │  │ Console     │  │
│  │            │  │            │  │ Panel    │  │             │  │
│  └─────┬──────┘  └─────┬──────┘  └────┬─────┘  └──────┬──────┘  │
│        └───────────┬───┴──────────────┴────────────────┘         │
│                    │ reads state / sends commands                 │
│              ┌─────▼─────┐                                       │
│              │ Game       │                                       │
│              │ Controller │ (Godot node, bridges UI ↔ Sim)       │
│              └─────┬──────┘                                       │
└────────────────────┼─────────────────────────────────────────────┘
                     │
        ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─  Godot boundary
                     │
┌────────────────────▼─────────────────────────────────────────────┐
│                    SIMULATION ENGINE (pure C#)                    │
│                                                                  │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────────┐  │
│  │ Tick Engine  │  │ Agent System  │  │ Accounting System     │  │
│  │              │  │               │  │ (SFC double-entry)    │  │
│  │ Orchestrates │  │ Government    │  │                       │  │
│  │ monthly tick │  │ Central Bank  │  │ Balance sheets        │  │
│  │ phases       │  │ Banks         │  │ Transaction log       │  │
│  │              │  │ Households    │  │ Consistency checks    │  │
│  │              │  │ Firms         │  │                       │  │
│  └──────┬───────┘  └───────┬───────┘  └───────────┬───────────┘  │
│         │                  │                      │              │
│         └──────────┬───────┴──────────────────────┘              │
│                    │                                             │
│              ┌─────▼──────┐                                      │
│              │ Data Layer │                                      │
│              │            │                                      │
│              │ JSON loader│                                      │
│              │ Schemas    │                                      │
│              │ Validation │                                      │
│              └────────────┘                                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 2. Layer Breakdown

### 2.1 Simulation Engine (Pure C# — No Godot Dependencies)

This is the heart of the game. It contains all economic logic, agent behavior, and accounting. It is a standalone C# class library that can be:

- Unit tested without Godot
- Run headlessly (for automated testing, benchmarking, or future server use)
- Developed and debugged in any C# IDE

**Key principle:** This layer knows nothing about rendering, UI, or Godot. It exposes state and accepts commands through clean interfaces.

#### Components

**Tick Engine**
- Orchestrates the monthly tick sequence (Government → Production → Market → Financial → Accounting)
- Manages simulation time (current month/year)
- Handles tick scheduling and phase ordering
- Flushes the policy pipeline at the start of each Government Phase (applies changes whose lag has expired)

**Agent System**
- Base interfaces for all economic agents
- Concrete implementations: Government, CentralBank, CommercialBank, Household, Firm
- Agent registry for looking up and iterating agents
- Agent behavior logic (production decisions, consumption, hiring, lending)
- Government participates as direct employer and procurer (see Section 3.14)

**Investment Engine**
- Processes public investment effects (infrastructure → capacity, public services → productivity) with data-driven lag durations via `IPolicyPipeline`
- Evaluates private firm investment decisions based on capacity utilization thresholds
- Applies capital depreciation to all capital stocks (public and private) each tick
- All rates, thresholds, and lags loaded from `IDataProvider`

**Accounting System (SFC)**
- Double-entry transaction recording
- Balance sheet management for every agent/sector
- Two-circuit ledger: every transaction is tagged with its money circuit (Reserves or Deposits). The ledger enforces circuit isolation: non-bank private agents (households, firms) may only transact in the Deposits circuit, Treasury and the central bank may only transact in the Reserves circuit, and commercial banks operate in both circuits (holding reserve accounts at the central bank and deposit accounts for customers).
- Transaction validation (source and destination must exist, amounts must balance, circuit rules enforced)
- Consistency checks (run post-tick, verify all balances sum to zero across both circuits)
- Circuit isolation check (verify no illegal cross-circuit flows)
- Transaction log for debugging and console inspection

**Markets**
- Labor market: wage posting (firms + government), job matching
- Goods market: AIDS-driven demand + government procurement, selling, inventory
- Bond market: auction mechanism, bidding

**Data Layer**
- JSON file loading and parsing
- Schema validation
- Parameter registry (all economic parameters accessible by path, e.g., `firms.agriculture.baseMarkup`)
- Data-driven agent initialization (sectors, classes, goods defined in data files)

### 2.2 Game Controller (Bridge Layer)

A thin Godot node that bridges the simulation engine and the UI. This is the only component that has dependencies on both Godot and the simulation.

**Responsibilities:**
- Initialize the simulation engine with loaded data
- Implement `ITimeControl` — advance ticks, manage pause/resume state and speed (FR-CON-004)
- Hold the `ISimulation` reference; expose `ISimulationState` to UI nodes (read-only)
- Route player commands (policy changes) from UI to simulation
- Expose `ISimulationState`, `ISimulationCommands`, and `ITimeControl` to console node
- Manage game modes (sandbox, scenario) and win/lose detection
- Handle save/load

**Does NOT contain:**
- Economic logic
- Agent behavior
- Rendering or UI layout

### 2.3 Presentation Layer (Godot Nodes)

Godot scenes and nodes that render the game and handle player input. These nodes read from the simulation state (via Game Controller) and send commands back.

**Components:**

**Policy Panel** — Control nodes (sliders, buttons) for player policy inputs

**Chart System** — Line charts for economic indicators, updated per tick

**Balance Sheet View** — Table display of sector balance sheets

**Map View** — 2D map rendering (cosmetic in MVP)

**Console** — Text input/output overlay, command parsing, help system

**HUD** — Time display, speed controls, mode indicator

**Scenario UI** — Objective display, progress tracking, win/lose feedback

## 3. Key Interfaces

### 3.1 Simulation State (Read)

The simulation exposes a read-only state object that the UI reads after each tick (see Section 6.3 for the update flow):

```csharp
public interface ISimulationState
{
    int CurrentMonth { get; }
    int CurrentYear { get; }

    IGovernmentState Government { get; }
    ICentralBankState CentralBank { get; }
    IBankingState Banks { get; }
    IReadOnlyList<IHouseholdClassState> HouseholdClasses { get; }
    IReadOnlyList<IFirmSectorState> FirmSectors { get; }

    IEconomicIndicators Indicators { get; }
    IReadOnlyList<IBalanceSheet> AllBalanceSheets { get; }
    IReadOnlyDictionary<EconomicSector, IBalanceSheet> SectorBalanceSheets { get; }
    IReadOnlyList<IPendingPolicy> PolicyPipeline { get; }

    /// Query any simulation value by dot-separated path. See Section 6.6 for the
    /// full path schema, naming conventions, return types, and error behavior.
    object QueryByPath(string path);
}
```

`SectorBalanceSheets` provides aggregated balance sheets keyed by `EconomicSector`. Aggregation rules:
- **Government, CentralBank, Banking:** return the single agent's balance sheet directly (no computation in MVP — each sector has exactly one agent).
- **Households:** sum `Assets` and `Liabilities` dictionaries across all household classes; `OwnerId` = `"Households"`.
- **Firms:** sum `Assets` and `Liabilities` dictionaries across all firm sectors; `OwnerId` = `"Firms"`.

The `*State` interfaces (`IGovernmentState`, `ICentralBankState`, etc.) are read-only projection interfaces exposed to the UI and console via `ISimulationState`. Internal agent interfaces (`IFirm`, `IHouseholdClass`, etc.) may carry additional implementation-facing properties and methods. The `*State` variants expose only what external consumers need. For `IFirmSectorState` and `IHouseholdClassState`, the agent interface extends its state projection (e.g., `IFirm : IFirmSectorState`), so properties are defined once in the state interface and inherited by the agent interface.

### 3.2 Simulation Commands (Write)

The UI sends commands to the simulation through a command interface:

```csharp
public interface ISimulationCommands
{
    void SetSpendingLevel(decimal amount);
    void SetSpendingAllocation(SpendingAllocation allocation);
    void SetTaxRate(decimal rate);

    // Time
    void Tick();
    void Tick(int count);
}
```

### 3.3 Agent Interfaces

```csharp
public interface IAgent
{
    string Id { get; }
    string Type { get; }
    IBalanceSheet BalanceSheet { get; }
}

// --- Agent State Projections ---
// Read-only interfaces exposed through ISimulationState.
// Agent interfaces extend their state projection so properties are defined once.

public interface IGovernmentState
{
    /// Cumulative fiscal balance (stock): positive = surplus, negative = deficit.
    /// Distinct from IEconomicIndicators.GovernmentFiscalBalance which is a per-period flow.
    decimal CumulativeFiscalBalance { get; }
    decimal TaxRate { get; }
    decimal SpendingLevel { get; }
    SpendingAllocation Allocation { get; }
    decimal BondsOutstanding { get; }
    int PublicSectorEmployment { get; }
    IReadOnlyDictionary<string, int> PublicEmploymentByFunction { get; }
    decimal GovernmentWageRate { get; }
    IReadOnlyDictionary<string, decimal> ProcurementBySector { get; }
    IBalanceSheet BalanceSheet { get; }
}

public interface ICentralBankState
{
    /// Fixed at 0 for MVP.
    decimal PolicyRate { get; }
    decimal TreasuryAccountBalance { get; }
    IBalanceSheet BalanceSheet { get; }
}

/// MVP models a single aggregate commercial bank.
/// Plural naming ("Banks", "IBankingState") is forward-compatible
/// for post-MVP multi-bank support (see L11).
public interface IBankingState
{
    decimal Reserves { get; }
    decimal LoansOutstanding { get; }
    decimal TotalDeposits { get; }
    decimal BondsHeld { get; }
    decimal Equity { get; }
    /// CB policy rate + spread + risk premium.
    decimal LendingRate { get; }
    IBalanceSheet BalanceSheet { get; }
}

public interface IFirmSectorState : IAgent
{
    string SectorId { get; }
    decimal PostedWage { get; }
    decimal CurrentPrice { get; }
    decimal CapacityUtilization { get; }
    decimal Productivity { get; }
    decimal UnitLaborCost { get; }
    int EmployeeCount { get; }
    decimal Inventory { get; }
    decimal CapitalStock { get; }
    decimal DepreciationRate { get; }
    decimal InvestmentDemand { get; }
    decimal CurrentMarkup { get; }
    decimal MinimumMarkup { get; }
}

public interface IHouseholdClassState : IAgent
{
    string ClassId { get; }
    int Population { get; }
    int Employed { get; }
    decimal AverageIncome { get; }
    IReadOnlyDictionary<string, decimal> BudgetShares { get; }
    IReadOnlyDictionary<string, decimal> ConsumptionBySector { get; }
    decimal TotalConsumption { get; }
    decimal SavingsBalance { get; }
    decimal DebtBalance { get; }
}

public interface IFirm : IFirmSectorState
{
}

public interface IHouseholdClass : IHouseholdClassState
{
}

/// Dashboard view of aggregate economic indicators (FR-SIM-004).
/// Some values intentionally duplicate agent state for UI convenience.
public interface IEconomicIndicators
{
    decimal EmploymentRate { get; }
    decimal UnemploymentRate { get; }
    decimal InflationRate { get; }
    decimal GDP { get; }
    /// Current-period fiscal balance (flow).
    /// Distinct from IGovernmentState.CumulativeFiscalBalance which is a cumulative stock.
    decimal GovernmentFiscalBalance { get; }
    decimal PrivateSectorNetSavings { get; }
    IReadOnlyDictionary<string, decimal> CapacityUtilizationBySector { get; }
    IReadOnlyDictionary<string, decimal> UnitLaborCostsBySector { get; }
    decimal PrivateDebtLevel { get; }
    decimal BankReserves { get; }
    /// Public sector workers / Total employed workers.
    decimal PublicSectorEmploymentShare { get; }
    /// Weighted average coupon rate across outstanding bonds (no secondary market in MVP).
    /// Formula: Σ(bond.CouponRate × bond.FaceValue) / Σ(bond.FaceValue). Returns 0 when no bonds outstanding.
    /// Pedagogically valuable: demonstrates the MMT insight that bond rates are policy-influenced
    /// (CB buyer-of-last-resort per FR-BND-001 effectively sets a rate ceiling).
    decimal BondYields { get; }
    decimal SavingsRate { get; }
    decimal WageGrowth { get; }
}
```

### 3.4 Accounting Interfaces

```csharp
public interface IBalanceSheet
{
    string OwnerId { get; }
    IReadOnlyDictionary<string, decimal> Assets { get; }
    IReadOnlyDictionary<string, decimal> Liabilities { get; }
    decimal NetPosition { get; } // Assets - Liabilities
}

public enum EconomicSector
{
    Government,
    CentralBank,
    Banking,
    Households,
    Firms
}

public enum MoneyCircuit
{
    Reserves,  // Central bank money — flows between Treasury, CB, bank reserve accounts
    Deposits   // Commercial bank money — flows between bank deposit accounts, households, firms
}

public interface ITransaction
{
    string Id { get; }
    int Tick { get; }
    MoneyCircuit Circuit { get; }
    string FromAccount { get; }
    string ToAccount { get; }
    decimal Amount { get; }
    string Category { get; } // "spending", "taxation", "wages", "purchase", "lending", etc.
    string Description { get; }
}

public interface ILedger
{
    void RecordTransaction(MoneyCircuit circuit, string from, string to, decimal amount, string category, string description);
    IReadOnlyList<ITransaction> GetTransactions(int tick);
    IReadOnlyList<ITransaction> GetTransactions(string accountId);
    IReadOnlyList<ITransaction> GetTransactions(MoneyCircuit circuit, int tick);
    bool CheckConsistency();       // Verify SFC across both circuits
    bool CheckCircuitIsolation();  // Verify no illegal cross-circuit flows
    decimal GetCircuitTotal(MoneyCircuit circuit);
}
```

### 3.5 Data Loading Interface

```csharp
public interface IDataProvider
{
    T LoadData<T>(string path);                    // Load and deserialize a JSON file
    bool TryLoadData<T>(string path, out T data);  // Non-throwing variant
    IEnumerable<string> ListFiles(string directory);
    string ResolvePath(string relativePath);        // Resolve through data/base/ (and later, mod overrides)
}
```

### 3.6 Data Validation Interface

```csharp
public interface IDataValidator
{
    ValidationResult Validate<T>(string path, T data);
    ValidationResult ValidateAll(IDataProvider provider);
}
```

Validates data at load time. Returns structured errors with file path, field name, and human-readable message. Used by both the game (fail-fast on bad data) and tests (verify error messages for malformed fixtures).

### 3.7 Simulation Composite and Factory

The simulation exposes separate read (`ISimulationState`, Section 3.1) and write (`ISimulationCommands`, Section 3.2) interfaces. A composite interface combines both for consumers that need full access:

```csharp
public interface ISimulation : ISimulationState, ISimulationCommands { }
```

The Game Controller and test harness hold an `ISimulation` reference. UI nodes receive only `ISimulationState` (read-only access).

```csharp
public interface ISimulationFactory
{
    ISimulation Create(IDataProvider dataProvider, int? seed = null);
}
```

Accepts an `IDataProvider` (either `JsonDataProvider` for the real game or `InMemoryDataProvider` for tests) and an optional seed for deterministic randomness. This is the single entry point for constructing a runnable simulation.

### 3.8 Policy Pipeline

Player policy changes (tax rate, spending level, spending allocation) do not take effect immediately. They enter a pipeline with data-driven lag durations (FR-TIM-001). The pipeline tracks pending changes so the UI can display them (FR-TIM-003, FR-UI-007).

```csharp
public enum PolicyChangeKind
{
    TaxRate,
    SpendingLevel,
    SpendingAllocation
}

public enum PolicyChangeStatus
{
    Enacted,      // Player submitted it — lag countdown starts
    InPipeline,   // Waiting for lag to expire
    TakingEffect  // Applied this tick
}

public interface IPendingPolicy
{
    PolicyChangeKind Kind { get; }
    PolicyChangeStatus Status { get; }
    int EnactedTick { get; }
    int EffectTick { get; }        // EnactedTick + lag duration
    int TicksRemaining { get; }
    object OldValue { get; }
    object NewValue { get; }
}

public interface IPolicyPipeline
{
    /// Enqueue a policy change. Lag duration is looked up from IDataProvider
    /// using the path "lags.policy.<kind>" (e.g., "lags.policy.taxRate" → 1).
    void Enqueue(PolicyChangeKind kind, object newValue, int currentTick);

    /// Called by TickEngine at the start of Government Phase.
    /// Returns changes whose lag has expired this tick, with Status = TakingEffect.
    IReadOnlyList<IPendingPolicy> FlushReady(int currentTick);

    /// All pending and recently-applied changes (for UI pipeline display).
    IReadOnlyList<IPendingPolicy> PendingChanges { get; }
}
```

`SimulationCommands` delegates to `IPolicyPipeline.Enqueue()` instead of directly mutating state. The `TickEngine` calls `FlushReady()` at the start of the Government Phase and applies the returned changes. `ISimulationState.PolicyPipeline` exposes `PendingChanges` to the UI (read-only).

**Overlapping changes:** If the player changes the same policy twice before the first takes effect, both entries exist in the pipeline. `FlushReady` applies them in enqueue order. The second change overwrites the first when it arrives.

**Note on economic lags (FR-TIM-002):** Wage stickiness, price adjustment speed, hiring delays, etc. are *not* queued commands — they are behavioral parameters governing how fast agents react. These are handled within individual engines (e.g., `PricingEngine` applies gradual price adjustments using a speed parameter from `IDataProvider`). The `IPolicyPipeline` component is only for player-initiated policy changes.

### 3.9 Investment Engine

```csharp
public interface ICapitalStock
{
    string OwnerId { get; }
    decimal PublicInfrastructure { get; }
    decimal PublicServices { get; }
    decimal PrivateCapital { get; }
}

public interface IInvestmentEngine
{
    /// Processes public investment: infrastructure allocation → capacity boost,
    /// services allocation → productivity boost. Effects deferred via IPolicyPipeline
    /// using lag durations from IDataProvider.
    /// Called during Government Phase after spending is executed.
    void ProcessPublicInvestment(ILedger ledger, IPolicyPipeline pipeline, int currentTick);

    /// Evaluates and executes private firm investment decisions.
    /// Firms invest when capacity utilization exceeds a data-driven threshold
    /// and they have retained profits or can obtain bank credit.
    /// Capital goods are purchased from the manufacturing sector (inter-sector demand).
    /// Called during Production Phase before production occurs.
    void ProcessPrivateInvestment(ILedger ledger);

    /// Applies depreciation to all capital stocks (public and private).
    /// Rates are sector-specific, loaded from IDataProvider.
    /// Called during Accounting Phase before balance sheet update.
    void ApplyDepreciation(ILedger ledger);
}
```

### 3.10 Pricing Engine

The `PricingEngine` implements FR-PRC-002's three-buffer inflation model — the central mechanism distinguishing MMT's view of inflation from mainstream models. Inflation only occurs when all three buffers are exhausted.

```csharp
public interface IPricingEngine
{
    /// Sets prices for all firms using cost-plus markup with three-buffer inflation logic.
    /// Buffer 1 (productivity): Emergent from ULC formula — if productivity rises with wages,
    ///   ULC stays flat and no price pressure exists. No pricing logic needed; this is a
    ///   production-side outcome.
    /// Buffer 2 (demand slack): Markup only increases when capacity utilization exceeds a
    ///   sector-specific threshold. Below that threshold, increased spending raises output,
    ///   not prices (firms produce more rather than charge more).
    /// Buffer 3 (margin compression): When input costs rise, firms first compress their markup
    ///   toward MinimumMarkup before passing cost increases to prices. This absorbs cost shocks
    ///   up to the margin slack available.
    /// Residual: Only cost increases that survive all three buffers flow through to price changes.
    /// Called during Market Phase before household purchasing.
    void SetPrices(ILedger ledger);
}
```

**Three-buffer mechanism in detail:**

1. **Buffer 1 — Productivity absorbs wage increases:** This buffer is emergent, not coded in the pricing engine. Unit labor cost (ULC) = total wages / total output. If productivity growth matches wage growth, ULC is flat and there is no cost pressure to pass through. The `PricingEngine` simply computes ULC; the buffer is a mathematical consequence of the production system.

2. **Buffer 2 — Demand slack absorbs spending increases:** When capacity utilization is below `CapacityThreshold` (per sector, from `IDataProvider`), increased demand is met by increasing output rather than raising markup. The markup adjustment only activates when utilization exceeds the threshold. Below it, spending increases raise real output, not prices.

3. **Buffer 3 — Margin compression absorbs cost increases:** When input costs (ULC + unit material cost) rise, firms first reduce their `CurrentMarkup` toward `MinimumMarkup` before raising prices. The absorbable amount equals `CurrentMarkup - MinimumMarkup`. Only the residual cost increase — the portion exceeding available margin slack — passes through to a price increase.

**"All three exhausted" condition:** Inflation occurs when (a) productivity has not kept pace with wages (ULC is rising), AND (b) the economy is operating above the capacity threshold (no output slack), AND (c) the firm's markup has already been compressed to `MinimumMarkup` (no margin slack). Only then does remaining cost pressure flow through as a price increase.

**Data-driven parameters** (per sector, loaded from `IDataProvider`):
- `BaseMarkup` — starting markup rate for the sector
- `MinimumMarkup` — floor below which markup cannot be compressed
- `MarkupAdjustmentSpeed` — how quickly markup adjusts per tick (FR-TIM-002 behavioral lag)
- `CapacityThreshold` — utilization level above which demand pressure raises markup

**Phase:** Market Phase (before household purchasing). See Section 4.1 tick data flow.

Investment logic is split across three existing tick phases rather than adding a sixth phase. Public investment reuses `IPolicyPipeline` for lag effects — infrastructure spending enters the pipeline with a 6-12 tick lag (FR-TIM-001) and boosts capacity when it matures; public services spending enters with a 12-24 tick lag and boosts productivity. Private investment creates inter-sector demand: when firms decide to expand, they purchase capital goods from the manufacturing sector, generating demand and transactions in that sector. All private investment transactions use `MoneyCircuit.Deposits`, funded from retained profits or bank credit (Phase 5 lending).

### 3.11 Time Control

The console needs to issue time commands (FR-CON-004: `pause`, `resume`, `speed`, `tick`) but these are Game Controller concerns, not simulation engine concerns. Even `tick` must route through the Game Controller because the tick-and-read pattern (Section 6.3) requires emitting TickCompleted after each tick so UI nodes refresh. Calling `ISimulationCommands.Tick()` directly would bypass this signal.

```csharp
public interface ITimeControl
{
    /// Advance the simulation by one tick and notify UI (emits TickCompleted).
    /// This is the game-level tick — use this from console and game loop.
    /// Distinct from ISimulationCommands.Tick() which is the engine-level
    /// operation (advances state only, no UI notification).
    void Tick();
    void Tick(int count);
    void Pause();
    void Resume();
    bool IsPaused { get; }
    int Speed { get; }
    void SetSpeed(int multiplier);  // 1 = 1x, 2 = 2x, 5 = 5x
}
```

Defined in the simulation layer as a pure interface (no Godot dependency). Implemented by the Game Controller in the Godot layer. The Game Controller's `Tick()` calls `ISimulation.Tick()` on the engine then emits the TickCompleted signal (Section 6.3). `Pause`/`Resume`/`SetSpeed` control the game loop timing.

`ISimulationCommands` retains its own `Tick()` methods for engine-level access (tests, headless runs, Game Controller internals). `ITimeControl.Tick()` wraps them with UI notification. The console routes ALL time commands (FR-CON-004) through `ITimeControl`.

`ITimeControl` is NOT part of the `ISimulation` composite — time control is a Game Controller concern, not a simulation engine concern. `ISimulation` is returned by `ISimulationFactory.Create()` for pure simulation instances that don't manage game loops.

### 3.12 Random Number Generation

Several simulation components need stochastic behavior (labor market matching, firm decision-making, property-based tests). To keep the simulation deterministic and reproducible, all randomness flows through a single injectable abstraction:

```csharp
public interface IRandom
{
    int Next(int minInclusive, int maxExclusive);
    double NextDouble(); // [0.0, 1.0)
}
```

`SeededRandom` wraps `System.Random` with a caller-provided seed. `ISimulationFactory.Create(data, seed)` instantiates `SeededRandom` and injects it into components that need stochastic behavior. In tests, a fixed seed guarantees deterministic replay — Phase 8's property-based tests (e.g., `Property_AnySeedAnyPolicySequence_SfcHoldsEveryTick`) depend on this for reproducibility across runs.

### 3.13 Consumption Engine (AIDS)

The `ConsumptionEngine` implements the Almost Ideal Demand System (Deaton & Muellbauer, 1980) for household consumption. It determines how each household class allocates its budget across production sectors based on current prices and real income.

```csharp
public interface IConsumptionEngine
{
    /// Computes budget shares and nominal demand per sector for each household class
    /// using the AIDS model. Reads current prices from firm sectors, income from
    /// household classes, and AIDS parameters (alpha, beta, gamma) from IDataProvider.
    /// Called during Market Phase after PricingEngine.SetPrices().
    void ComputeDemand();

    /// Budget shares by household class and sector (output of last ComputeDemand call).
    /// Used by GoodsMarket to execute purchases and by UI for visualization.
    IReadOnlyDictionary<string, IReadOnlyDictionary<string, decimal>> BudgetSharesByClass { get; }

    /// Nominal demand by household class and sector (budget share × disposable income).
    IReadOnlyDictionary<string, IReadOnlyDictionary<string, decimal>> NominalDemandByClass { get; }
}
```

**AIDS formula** for household class _c_ and sector _i_:

```
w_ci = alpha_ci + SUM_j(gamma_cij * ln(p_j)) + beta_ci * ln(M_c / P)
```

Where `w_ci` is the budget share, `alpha_ci` is the intercept, `gamma_cij` captures price effects, `beta_ci` captures income effects, `M_c` is disposable income, and `P` is the Stone price index.

**Per-class parameterization:** Each household class has its own alpha, beta, and gamma parameters loaded from `IDataProvider` (`economy/consumption.json`). This reflects empirical evidence that preferences differ structurally across income groups.

**Theoretical constraints** (enforced by data validation at load time):
1. **Adding-up:** `SUM_i(alpha_ci) = 1`, `SUM_i(gamma_cij) = 0`, `SUM_i(beta_ci) = 0`
2. **Homogeneity:** `SUM_j(gamma_cij) = 0` for each _i_
3. **Symmetry:** `gamma_cij = gamma_cji`

**Edge cases:**
- Near-zero income: epsilon floor on real income before `ln(M/P)` to prevent undefined behavior
- Extreme prices: budget shares clamped to `[0, 1]` and renormalized to sum to 1

**Data-driven parameters** (per class, per sector, loaded from `IDataProvider`):
- `alpha` vector — intercept budget shares at reference prices/income
- `beta` vector — income response (how budget shares shift with real income)
- `gamma` matrix — price response (own-price and cross-price effects, symmetric)

**Phase:** Market Phase (after `PricingEngine.SetPrices()`, before household purchasing). See Section 4.1 tick data flow.

### 3.14 Government Demand

The government participates in the real economy through two channels: direct employment and sector procurement. This creates resource competition with the private sector — the core mechanism demonstrating the MMT insight that the real constraint on government spending is available real resources, not financial capacity.

```csharp
public interface IGovernmentDemand
{
    /// Public sector job postings by spending function (infrastructure, publicServices).
    /// Each posting specifies wage rate and labor quantity.
    /// Registered with LaborMarket during Government Phase.
    IReadOnlyList<GovernmentJobPosting> JobPostings { get; }

    /// Procurement demand by sector, computed from current spending allocation.
    /// Added to household demand in the Market Phase.
    IReadOnlyDictionary<string, decimal> ProcurementBySector { get; }

    /// Compute government resource demand from current spending allocation.
    /// Called during Government Phase after spending is executed.
    void ComputeResourceDemand(decimal infrastructureSpending, decimal publicServicesSpending);
}

public class GovernmentJobPosting
{
    public string Function { get; }       // "infrastructure" or "publicServices"
    public decimal WageRate { get; }      // Data-driven pay scale
    public int LaborDemand { get; }       // Number of workers sought
}
```

**Spending-to-resource mapping** (data-driven, loaded from `IDataProvider` via `agents/government.json`):

| Spending category | Public employment | Sector procurement | Notes |
|---|---|---|---|
| Infrastructure | Engineers, planners (share of spending) | Construction, manufacturing (share of spending) | Both channels compete for real resources |
| Public services | Teachers, doctors, admin (share of spending) | Equipment, outsourced services (share of spending) | Labor-intensive |
| Direct transfers | None | None | No direct resource competition; flows to household income → AIDS demand |

**Government wage behavior:**
- Pay scale is data-driven, loaded from `IDataProvider`
- Adjusts more slowly than private sector wages (civil service stickiness parameter)
- Creates a de facto wage floor — private firms must offer competitive wages or lose workers to the public sector

**Phase:** Government Phase (after spending execution, before Production Phase). Job postings enter the labor market pool alongside firm postings. Procurement demand is served during Market Phase alongside household AIDS demand.

## 4. Data Flow

### 4.1 Simulation Tick

```
Game Controller
    │
    ├── calls Tick() on Simulation Engine
    │
    ▼
Tick Engine
    │
    ├── 1. Government Phase
    │   ├── Flush policy pipeline → apply changes whose lag has expired
    │   ├── Collect taxes → Ledger.RecordTransaction (Deposits: taxpayer → bank) then (Reserves: bank → treasury)
    │   ├── Execute spending → Ledger.RecordTransaction (Reserves: treasury → bank) then (Deposits: bank → recipient)
    │   ├── Process public investment → InvestmentEngine.ProcessPublicInvestment()
    │   ├── Compute government resource demand from spending allocation:
    │   │   ├── Register public sector job postings with LaborMarket
    │   │   └── Register procurement demand with sectors
    │   ├── Pay bond interest → Ledger.RecordTransaction (Reserves + Deposits)
    │   └── Bond auction → BondMarket.RunAuction()
    │
    ├── 2. Production Phase
    │   ├── Evaluate firm investment → InvestmentEngine.ProcessPrivateInvestment()
    │   ├── Firms estimate demand
    │   ├── Firms set production targets
    │   │   └── (Production targets increase up to available capacity,
    │   │        absorbing demand as output growth — FR-PRC-002 buffer 2)
    │   ├── Firms post wages → LaborMarket.PostJobs()
    │   ├── Government posts wages → LaborMarket.PostGovernmentJobs()
    │   ├── Households evaluate ALL postings (firm + government) → LaborMarket.MatchJobs()
    │   └── Production occurs → Firms produce output
    │
    ├── 3. Market Phase
    │   ├── Firms set prices → PricingEngine.SetPrices()
    │   │   ├── ULC computed (buffer 1: if productivity matched wages, ULC flat)
    │   │   ├── Markup adjusted for demand (buffer 2: no increase if slack exists)
    │   │   ├── Cost increases absorbed by margin compression (buffer 3: markup → min)
    │   │   └── Residual pressure → price change
    │   ├── ConsumptionEngine.ComputeDemand() → AIDS budget shares × income = demand per sector
    │   ├── Government procurement demand added to sector demand
    │   ├── Households purchase from sectors based on computed demand
    │   ├── Firms sell from inventory to meet combined demand (household + government)
    │   └── Transactions recorded → Ledger.RecordTransaction
    │
    ├── 4. Financial Phase
    │   ├── Banks process loan applications → Credit creation
    │   ├── Borrowers make debt service payments → Money destruction
    │   └── Defaults processed
    │
    └── 5. Accounting Phase
        ├── Apply depreciation → InvestmentEngine.ApplyDepreciation()
        ├── Balance sheets updated
        ├── SFC consistency check → Ledger.CheckConsistency()
        ├── Circuit isolation check → Ledger.CheckCircuitIsolation()
        ├── Economic indicators calculated
        └── State snapshot published → Game Controller reads new state
```

### 4.2 Player Input

```
Player adjusts slider (Godot UI)
    │
    ▼
Policy Panel node emits signal
    │
    ▼
Game Controller receives signal
    │
    ▼
Game Controller calls ISimulationCommands.SetTaxRate(newRate)
    │
    ▼
SimulationCommands delegates to IPolicyPipeline.Enqueue(TaxRate, newRate, currentTick)
    │  (lag duration looked up from IDataProvider, e.g., "lags.policy.taxRate" → 1 tick)
    ▼
IPendingPolicy created: EffectTick = currentTick + lagDuration, Status = InPipeline
    │
    ▼
UI reads ISimulationState.PolicyPipeline → shows change in pipeline display
    │
    ▼
N ticks later: TickEngine calls pipeline.FlushReady(currentTick)
    → returns this change with Status = TakingEffect
    → Government Phase applies the new tax rate
```

### 4.3 Console Command

```
Player types command in console
    │
    ▼
Console node parses and routes command
    │
    ├── Query  → ISimulationState.QueryByPath() → displays result
    │
    ├── Policy → ISimulationCommands (e.g., SetTaxRate, SetSpendingLevel)
    │
    └── Time   → ITimeControl (Pause, Resume, SetSpeed, Tick)
                    │
                    ▼
                Game Controller executes
                    │
                    ├── Tick: calls ISimulation.Tick() → emits TickCompleted
                    └── Pause/Resume/Speed: updates game loop state
```

## 5. Project Structure

```
game/
├── game.sln                          # C# solution file
│
├── src/
│   ├── Simulation/                   # Pure C# class library (no Godot deps)
│   │   ├── Simulation.csproj
│   │   ├── Core/
│   │   │   ├── TickEngine.cs         # Tick orchestration
│   │   │   ├── SimulationState.cs    # Central state container
│   │   │   ├── SimulationCommands.cs # Command handler
│   │   │   ├── PolicyPipeline.cs     # IPolicyPipeline implementation
│   │   │   ├── PendingPolicy.cs      # IPendingPolicy value type
│   │   │   ├── ITimeControl.cs       # Time control interface (implemented by Game Controller)
│   │   │   ├── IRandom.cs            # Seeded randomness interface
│   │   │   └── SeededRandom.cs       # IRandom implementation wrapping System.Random
│   │   ├── Accounting/
│   │   │   ├── Ledger.cs             # Double-entry ledger
│   │   │   ├── BalanceSheet.cs       # Balance sheet per agent
│   │   │   ├── Transaction.cs        # Transaction record
│   │   │   └── SfcChecker.cs         # Consistency validation
│   │   ├── Agents/
│   │   │   ├── IAgent.cs             # Agent interface
│   │   │   ├── Government.cs
│   │   │   ├── CentralBank.cs
│   │   │   ├── CommercialBank.cs
│   │   │   ├── Household.cs
│   │   │   └── Firm.cs
│   │   ├── Markets/
│   │   │   ├── LaborMarket.cs        # Wage posting (firms + government) and matching
│   │   │   ├── GoodsMarket.cs        # AIDS demand + government procurement, selling, rationing
│   │   │   └── BondMarket.cs         # Bond auction
│   │   ├── Economics/
│   │   │   ├── PricingEngine.cs      # Cost-plus markup calculation
│   │   │   ├── ProductionEngine.cs   # Production function
│   │   │   ├── InvestmentEngine.cs   # Public/private investment and depreciation
│   │   │   ├── ConsumptionEngine.cs  # AIDS demand system
│   │   │   └── IndicatorCalculator.cs # GDP, inflation, etc.
│   │   └── Data/
│   │       ├── IDataProvider.cs      # Data loading interface
│   │       ├── JsonDataProvider.cs   # JSON file loader
│   │       └── Models/              # Data models for JSON deserialization
│   │           ├── SectorData.cs
│   │           ├── HouseholdData.cs
│   │           ├── ConsumptionData.cs # AIDS parameters (alpha, beta, gamma per class)
│   │           ├── GovernmentDemandData.cs # Spending-to-resource mapping
│   │           ├── InvestmentData.cs
│   │           ├── PricingData.cs     # Per-sector markup parameters and thresholds
│   │           └── ScenarioData.cs
│   │
│   └── Game/                         # Godot project
│       ├── project.godot             # Godot project file
│       ├── Game.csproj               # Godot C# project (references Simulation)
│       ├── Scenes/
│       │   ├── Main.tscn             # Main scene
│       │   ├── UI/
│       │   │   ├── PolicyPanel.tscn
│       │   │   ├── Charts.tscn
│       │   │   ├── BalanceSheetView.tscn
│       │   │   ├── Console.tscn
│       │   │   ├── HUD.tscn
│       │   │   └── ScenarioUI.tscn
│       │   └── Map/
│       │       └── MapView.tscn
│       ├── Scripts/
│       │   ├── GameController.cs     # Bridge: Simulation ↔ Godot
│       │   ├── UI/
│       │   │   ├── PolicyPanel.cs
│       │   │   ├── ChartPanel.cs
│       │   │   ├── BalanceSheetPanel.cs
│       │   │   ├── ConsolePanel.cs
│       │   │   ├── HUD.cs
│       │   │   └── ScenarioUI.cs
│       │   └── Map/
│       │       └── MapView.cs
│       └── Assets/
│           ├── Themes/               # Godot UI themes
│           ├── Fonts/
│           └── Textures/
│
├── tests/
│   ├── Simulation.Tests/            # Unit tests for simulation engine
│   │   ├── Simulation.Tests.csproj
│   │   ├── Accounting/
│   │   │   ├── LedgerTests.cs
│   │   │   ├── SfcCheckerTests.cs
│   │   │   └── BalanceSheetTests.cs
│   │   ├── Agents/
│   │   │   ├── GovernmentTests.cs
│   │   │   ├── HouseholdTests.cs
│   │   │   ├── FirmTests.cs
│   │   │   └── BankTests.cs
│   │   ├── Markets/
│   │   │   ├── LaborMarketTests.cs
│   │   │   ├── GoodsMarketTests.cs
│   │   │   └── BondMarketTests.cs
│   │   ├── Data/
│   │   │   ├── JsonDataProviderTests.cs
│   │   │   ├── DataValidatorTests.cs
│   │   │   └── DataModelTests.cs
│   │   ├── Integration/
│   │   │   ├── TickIntegrationTests.cs
│   │   │   └── MoneyFlowTests.cs
│   │   ├── PropertyTests/
│   │   │   ├── SfcInvariantTests.cs
│   │   │   ├── PriceInvariantTests.cs
│   │   │   └── MoneyStockTests.cs
│   │   ├── Helpers/
│   │   │   ├── InMemoryDataProvider.cs
│   │   │   ├── SimulationTestHarness.cs
│   │   │   └── TestDataBuilder.cs
│   │   └── TestData/
│   │       ├── Minimal/              # 1 sector, 1 class — bare minimum valid data
│   │       ├── FullBase/             # Copy of data/base/ for integration tests
│   │       ├── InvalidData/          # Malformed files for error handling tests
│   │       └── Scenarios/            # Specific economic states (high-inflation, etc.)
│
├── data/
│   └── base/                         # Base game data (JSON files)
│       ├── economy/
│       │   ├── sectors.json
│       │   ├── consumption.json    # AIDS parameters per household class
│       │   ├── production.json
│       │   └── parameters.json
│       ├── agents/
│       │   ├── households.json
│       │   ├── firms.json
│       │   ├── banks.json
│       │   └── government.json
│       ├── scenarios/
│       │   ├── sandbox.json
│       │   └── full-employment.json
│       ├── map/
│       │   └── default-map.json
│       └── config/
│           ├── simulation.json
│           └── ui.json
│
├── docs/                             # Project documentation
│   ├── README.md
│   ├── design/
│   │   ├── GAME-DESIGN.md
│   │   └── ECONOMIC-MODEL.md
│   ├── requirements/
│   │   └── PRD.md
│   ├── technical/
│   │   ├── ARCHITECTURE.md
│   │   ├── IMPLEMENTATION-PLAN.md
│   │   ├── CONSOLE.md
│   │   └── MODDING.md
│   └── reviews/
│       ├── consistency.md
│       └── mmt-accuracy.md
│
└── mods/                             # Mod directory (empty in base game)
    └── .gitkeep
```

## 6. Key Design Patterns

### 6.1 Simulation as Pure Library

The simulation engine is a pure C# class library with **zero Godot dependencies**. This is enforced at the project level: `Simulation.csproj` does not reference Godot assemblies.

Benefits:
- Unit testable with standard C# test frameworks (xUnit, NUnit)
- Can run headlessly for benchmarking or automated testing
- Could be reused in other frontends (web, different engine) in the future
- Clean separation forces good architecture

### 6.2 Command Pattern for Player Actions

All player actions go through a command interface rather than directly mutating state. This:
- Centralizes all state mutations
- Makes it easy to add undo/redo later
- Makes it easy to record/replay sessions
- Enables the console to use the same commands as the UI

### 6.3 Tick-and-Read UI Update Pattern

After each simulation tick, the Game Controller notifies all UI nodes that new state is available. UI nodes then read the values they need from `ISimulationState`.

```
Game Controller
    │
    ├── calls Tick() on Simulation Engine
    ├── Simulation returns (state is now updated)
    │
    └── emits TickCompleted signal (Godot signal)
            │
            ├── ChartPanel reads Indicators, appends data points
            ├── BalanceSheetPanel reads SectorBalanceSheets, refreshes table
            ├── HUD reads CurrentMonth/CurrentYear, updates label
            ├── PolicyPanel reads pending policy state, updates indicators
            └── ScenarioUI reads scenario progress, updates objectives
```

This works because:
- The simulation is discrete (monthly ticks), not continuous — there is exactly one state change per frame at most.
- `ISimulationState` already provides all the data UI nodes need.
- A single Godot signal is trivial to implement and maintain.
- Coupling is one-directional: UI nodes depend on `ISimulationState`, the simulation knows nothing about the UI.

No `ISimulationEvents` interface is needed in the simulation engine. The notification mechanism lives entirely in the Godot layer as a standard Godot signal on `GameController`.

### 6.4 Data-Driven Agent Configuration

Agents are not hardcoded. Their types, parameters, and behavior thresholds are loaded from JSON data files. The code provides the behavior framework; the data files configure it.

```
Code defines:  "A Firm has inputs, outputs, a markup, and produces goods"
Data defines:  "Agriculture takes labor+land, outputs food, has 15% markup"
```

This means mods can add new sectors, goods, and household classes without touching code (Tier 1 modding).

### 6.5 Dependency Injection for Testability

All simulation components receive their dependencies via constructor injection. No component creates its own dependencies internally. This enables tests to substitute any dependency with a test double.

**Core injectable dependencies:**
- `IDataProvider` — data loading (real: `JsonDataProvider`, test: `InMemoryDataProvider`)
- `ILedger` — transaction recording and SFC checking
- `IAgentRegistry` — agent lookup and iteration
- `IRandom` — seeded random number generation for deterministic tests
- `IPolicyPipeline` — policy change queuing and lag tracking (durations from `IDataProvider`)
- `IInvestmentEngine` — public/private investment processing and capital depreciation (rates and thresholds from `IDataProvider`)
- `IPricingEngine` — cost-plus pricing with three-buffer inflation logic (markup parameters and capacity thresholds from `IDataProvider`)
- `IConsumptionEngine` — AIDS demand computation (parameters per household class from `IDataProvider`)
- `IGovernmentDemand` — government employment and procurement demand (spending-to-resource mapping from `IDataProvider`)

Example: a `Firm` receives `ILedger` and `IDataProvider` in its constructor. In production, these are the real implementations wired through `ISimulationFactory`. In tests, they are in-memory fakes that allow precise control over inputs and verification of outputs.

### 6.6 Path-Based State Access

`ISimulationState.QueryByPath(string path)` provides string-addressed access to any simulation value. It exists primarily for the **console** (`query firms.agriculture.currentPrice`) and **future mod scripts** — consumers where the path is a runtime string. The chart/UI system should use typed access via `ISimulationState` properties directly (see Section 6.3).

#### Path Schema

Paths are dot-separated, lowercase segments. Property names use **camelCase** versions of the C# interface property names.

**Structure:**

```
<root>.<property>                          → scalar value
<root>.<collection>.<id>.<property>        → scalar value from a specific agent
<root>.<collection>.<id>                   → all properties as IReadOnlyDictionary<string, object>
<root>                                     → all properties as IReadOnlyDictionary<string, object>
```

**Root paths and their source interfaces:**

| Root | Source Interface | Collection Key |
|---|---|---|
| `time` | `CurrentMonth`, `CurrentYear` on `ISimulationState` | — |
| `government` | `IGovernmentState` | — |
| `centralbank` | `ICentralBankState` | — |
| `banks` | `IBankingState` | — |
| `firms.<sectorId>` | `IFirmSectorState` | `SectorId` (e.g., `agriculture`, `manufacturing`, `construction`, `services`) |
| `households.<classId>` | `IHouseholdClassState` | `ClassId` (e.g., `low`, `middle`, `high`) |
| `indicators` | `IEconomicIndicators` | — |

**Example paths and return types:**

| Path | Returns | C# Type |
|---|---|---|
| `time.month` | Current month | `int` |
| `time.year` | Current year | `int` |
| `government.taxRate` | Current tax rate | `decimal` |
| `government.bondsOutstanding` | Total bonds outstanding | `decimal` |
| `centralbank.policyRate` | CB policy rate | `decimal` |
| `banks.reserves` | Bank reserve balance | `decimal` |
| `banks.lendingRate` | Current lending rate | `decimal` |
| `firms.agriculture.currentPrice` | Agriculture sector price | `decimal` |
| `firms.manufacturing.capacityUtilization` | Manufacturing capacity utilization | `decimal` |
| `firms.construction` | All construction properties | `IReadOnlyDictionary<string, object>` |
| `government.publicSectorEmployment` | Total public sector employees | `int` |
| `government.governmentWageRate` | Public sector wage rate | `decimal` |
| `government.procurementBySector` | Procurement demand per sector | `IReadOnlyDictionary<string, decimal>` |
| `households.low.employed` | Low-class employed count | `int` |
| `households.low.budgetShares` | Low-class AIDS budget shares per sector | `IReadOnlyDictionary<string, decimal>` |
| `households.low.consumptionBySector` | Low-class nominal spending per sector | `IReadOnlyDictionary<string, decimal>` |
| `indicators.gdp` | Current GDP | `decimal` |
| `indicators.inflationRate` | Current inflation rate | `decimal` |
| `indicators.publicSectorEmploymentShare` | Public / total employment | `decimal` |
| `indicators.capacityUtilizationBySector` | Sector utilization map | `IReadOnlyDictionary<string, decimal>` |

The full set of leaf properties is defined by the corresponding `*State` / `IEconomicIndicators` interfaces in Section 3.3. Any property added to those interfaces automatically becomes a valid path leaf — the schema is **interface-derived**, not independently maintained.

#### Error Behavior

| Condition | Result |
|---|---|
| Valid path to a scalar | Returns the value (typed as `decimal`, `int`, `string`, etc.) |
| Valid path to a collection node | Returns `IReadOnlyDictionary<string, object>` of all child properties |
| Invalid path (typo, nonexistent property) | Returns `null` |
| `null` or empty path | Returns `null` |

No exceptions are thrown across the `QueryByPath` boundary. Console consumers display `"(not found)"` for `null` results.

#### Implementation Strategy

The resolver is built once at simulation creation using **reflection with caching**:

```csharp
class StatePathResolver
{
    // Built at construction time by walking ISimulationState's interface tree.
    // For indexed collections (FirmSectors, HouseholdClasses), paths are registered
    // using the agent's SectorId/ClassId as the key segment.
    // Each entry is a compiled Func<ISimulationState, object> — no per-query reflection cost.
    private readonly Dictionary<string, Func<ISimulationState, object>> _resolvers;

    public object Resolve(ISimulationState state, string path);
}
```

The resolver must be rebuilt when the agent list changes (e.g., loading a different scenario with different sectors). In practice this only happens at simulation creation.

#### Chart / UI Binding

The chart and UI systems do **not** use `QueryByPath`. They read typed properties directly from `ISimulationState` (see Section 6.3). This keeps the UI compile-time safe and avoids string-path fragility. `QueryByPath` is reserved for the console and future mod/script consumers where paths are inherently runtime strings.

## 7. Technology Stack

| Component | Technology |
|---|---|
| Game engine | Godot 4.x (.NET variant) |
| Simulation language | C# (.NET 8+) |
| UI scripting | C# |
| Data format | JSON |
| Unit testing | xUnit (or NUnit) |
| Build system | .NET SDK / MSBuild |
| Version control | Git |

## 8. Constraints and Decisions

### 8.1 Why Simulation is a Separate Library

The most important architectural decision. Alternatives considered:

| Approach | Rejected because |
|---|---|
| All code in Godot project | Can't unit test without Godot. Tight coupling. Hard to refactor. |
| Simulation as Godot autoload | Still coupled to Godot. Can't run headlessly. |
| **Separate C# library** (chosen) | Clean separation. Testable. Portable. Slightly more project setup. |

### 8.2 Why Not ECS (Entity Component System)

Godot and some game engines push ECS. We use a more traditional OOP agent model because:
- Economic agents have complex behavior that maps well to objects
- The SFC accounting system is inherently relational (balance sheets, transactions)
- Population groups are not entities with interchangeable components
- ECS adds complexity without clear benefit for this simulation type

### 8.3 Single-Threaded Simulation

The simulation runs on a single thread. Multi-threading adds complexity and is unnecessary for the MVP:
- Monthly ticks are sequential by nature (phases depend on previous phases)
- Population groups (not individual agents) keep the computation manageable
- Godot's main thread handles rendering; simulation can run on a background thread if needed later

## 9. Testing Architecture

### 9.1 Testing Philosophy

The simulation engine is developed using Test-Driven Development (TDD). Tests are written before implementation code, following the red-green-refactor cycle:

1. **Red:** Write a failing test that defines the desired behavior
2. **Green:** Write the minimum code to make the test pass
3. **Refactor:** Clean up the code while keeping all tests green

Tests are the executable specification of the economic model. If a behavior isn't tested, it isn't guaranteed.

### 9.2 Test Layers

```
┌─────────────────────────────────────────────────────────┐
│              Property-Based Tests                        │
│  SFC identity holds for all seeds and policy sequences  │
│  No negative prices for any valid input                 │
│  Employment rate always in [0, 1]                       │
│  Money stock = govt spending - taxation + bank lending   │
├─────────────────────────────────────────────────────────┤
│              Integration Tests                           │
│  Full tick cycles with all phases                       │
│  Money flow end-to-end (govt → bank → household → firm) │
│  Policy-to-outcome chains (spending → employment)       │
│  Multi-tick stability (120-tick runs)                   │
├─────────────────────────────────────────────────────────┤
│              Unit Tests                                  │
│  Ledger, BalanceSheet, SfcChecker                       │
│  Each agent type (Government, Household, Firm, Bank)    │
│  PricingEngine, ProductionEngine, LaborMarket           │
│  JsonDataProvider, DataValidator, data model loading    │
└─────────────────────────────────────────────────────────┘
```

### 9.3 Test Data Fixtures

```
tests/Simulation.Tests/
├── TestData/
│   ├── Minimal/          # 1 sector, 1 household class — bare minimum valid data
│   │   ├── sectors.json
│   │   ├── consumption.json
│   │   ├── households.json
│   │   ├── firms.json
│   │   ├── banks.json
│   │   ├── government.json
│   │   └── parameters.json
│   ├── FullBase/         # Copy of data/base/ for integration tests
│   ├── InvalidData/      # Malformed files for error handling tests
│   │   ├── missing-fields.json
│   │   ├── negative-values.json
│   │   ├── invalid-types.json
│   │   └── empty-file.json
│   └── Scenarios/        # Specific economic states for targeted tests
│       ├── high-inflation.json
│       ├── high-unemployment.json
│       └── steady-state.json
```

- **Minimal:** The smallest valid dataset. Used by most unit tests to keep tests fast and focused.
- **FullBase:** A copy of the real `data/base/` directory. Used by integration tests and property-based tests. CI checks that this stays in sync with `data/base/`.
- **InvalidData:** Intentionally malformed files. Used to verify that validation produces clear error messages.
- **Scenarios:** Pre-configured economic states (e.g., economy already in high inflation). Used for targeted behavior tests.

### 9.4 InMemoryDataProvider

Implements `IDataProvider` for tests. Instead of reading JSON files from disk, tests register data objects directly in memory:

```csharp
var data = new InMemoryDataProvider();
data.Register("economy/sectors", new SectorData { ... });
data.Register("agents/households", new HouseholdData { ... });

var sim = factory.Create(data, seed: 42);
```

This enables:
- **Minimal data** for focused unit tests (only register what the test needs)
- **Extreme values** for boundary tests (e.g., markup of 0, population of 1)
- **Invalid data** for error handling tests
- **No file system dependency** — tests run anywhere without data files on disk

### 9.5 SimulationTestHarness

A convenience class that wraps common test setup and assertion patterns:

```csharp
public class SimulationTestHarness
{
    // Construction
    static SimulationTestHarness CreateMinimal();               // 1 sector, 1 class
    static SimulationTestHarness CreateFromFixture(string name); // Load from TestData/
    static SimulationTestHarness CreateFromData(IDataProvider provider, int? seed = null);

    // Execution
    void Tick(int count = 1);

    // Assertions
    void AssertSfcConsistent();
    void AssertIndicatorInRange(string indicator, decimal min, decimal max);
    void AssertNoNegativePrices();
    void AssertEmploymentInBounds();

    // Access
    ISimulationState State { get; }
    ILedger Ledger { get; }
}
```

The harness wraps an `ISimulation` instance internally. `State` delegates to `ISimulationState`; `Tick()` delegates to `ISimulationCommands`.

### 9.6 Test Naming Convention

Tests follow the pattern: `[RequirementId]_[Scenario]_[ExpectedResult]`

Examples:
- `FrSim001_AfterGovernmentSpending_SfcIdentityHolds`
- `FrPrc001_WageRiseMatchesProductivityGain_PriceUnchanged`
- `FrBnk001_LoanCreated_DepositsAndLoansIncrease`
- `FrMod001_MissingSectorField_ValidationErrorWithFieldName`
