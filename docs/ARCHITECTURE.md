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

**Agent System**
- Base interfaces for all economic agents
- Concrete implementations: Government, CentralBank, CommercialBank, Household, Firm
- Agent registry for looking up and iterating agents
- Agent behavior logic (production decisions, consumption, hiring, lending)

**Accounting System (SFC)**
- Double-entry transaction recording
- Balance sheet management for every agent/sector
- Two-circuit ledger: every transaction is tagged with its money circuit (Reserves or Deposits). The ledger enforces circuit isolation: non-bank private agents (households, firms) may only transact in the Deposits circuit, Treasury and the central bank may only transact in the Reserves circuit, and commercial banks operate in both circuits (holding reserve accounts at the central bank and deposit accounts for customers).
- Transaction validation (source and destination must exist, amounts must balance, circuit rules enforced)
- Consistency checks (run post-tick, verify all balances sum to zero across both circuits)
- Circuit isolation check (verify no illegal cross-circuit flows)
- Transaction log for debugging and console inspection

**Markets**
- Labor market: wage posting, job matching
- Goods market: pricing, buying, selling, inventory
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
- Advance the simulation (call tick) based on game speed and pause state
- Expose simulation state to UI nodes (read-only)
- Route player commands (policy changes) from UI to simulation
- Route console commands to simulation
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

    // For console: query any value by path
    object QueryByPath(string path);
}
```

### 3.2 Simulation Commands (Write)

The UI sends commands to the simulation through a command interface:

```csharp
public interface ISimulationCommands
{
    void SetSpendingLevel(decimal amount);
    void SetSpendingAllocation(SpendingAllocation allocation);
    void SetTaxRate(decimal rate);

    // Console commands
    void ExecuteConsoleCommand(string command);

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

public interface IFirm : IAgent
{
    string SectorId { get; }
    decimal PostedWage { get; }
    decimal CurrentPrice { get; }
    decimal CapacityUtilization { get; }
    decimal Productivity { get; }
    decimal UnitLaborCost { get; }
    int EmployeeCount { get; }
    decimal Inventory { get; }
}

public interface IHouseholdClass : IAgent
{
    string ClassId { get; }
    int Population { get; }
    int Employed { get; }
    decimal AverageIncome { get; }
    decimal ConsumptionSpending { get; }
    decimal SavingsBalance { get; }
    decimal DebtBalance { get; }
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

### 3.7 Simulation Factory Interface

```csharp
public interface ISimulationFactory
{
    ISimulationState Create(IDataProvider dataProvider, int? seed = null);
}
```

Accepts an `IDataProvider` (either `JsonDataProvider` for the real game or `InMemoryDataProvider` for tests) and an optional seed for deterministic randomness. This is the single entry point for constructing a runnable simulation.

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
    │   ├── Collect taxes → Ledger.RecordTransaction (Deposits: taxpayer → bank) then (Reserves: bank → treasury)
    │   ├── Execute spending → Ledger.RecordTransaction (Reserves: treasury → bank) then (Deposits: bank → recipient)
    │   ├── Pay bond interest → Ledger.RecordTransaction (Reserves + Deposits)
    │   └── Bond auction → BondMarket.RunAuction()
    │
    ├── 2. Production Phase
    │   ├── Firms estimate demand
    │   ├── Firms set production targets
    │   ├── Firms post wages → LaborMarket.PostJobs()
    │   ├── Households accept jobs → LaborMarket.MatchJobs()
    │   └── Production occurs → Firms produce output
    │
    ├── 3. Market Phase
    │   ├── Firms set prices (cost-plus markup)
    │   ├── Households purchase goods (hierarchical needs)
    │   └── Transactions recorded → Ledger.RecordTransaction
    │
    ├── 4. Financial Phase
    │   ├── Banks process loan applications → Credit creation
    │   ├── Borrowers make debt service payments → Money destruction
    │   └── Defaults processed
    │
    └── 5. Accounting Phase
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
Game Controller calls ISimulationCommands.SetSpendingLevel(newValue)
    │
    ▼
Simulation Engine queues policy change with appropriate lag
    │
    ▼
N ticks later: policy change takes effect in Government Phase
```

### 4.3 Console Command

```
Player types command in console
    │
    ▼
Console node parses command
    │
    ├── Query command → calls ISimulationState.QueryByPath() → displays result
    │
    └── Write command → calls ISimulationCommands.ExecuteConsoleCommand()
                            │
                            ▼
                        Simulation Engine executes through proper channels
                            │
                            ▼
                        Result returned to Console → displays output
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
│   │   │   └── SimulationCommands.cs # Command handler
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
│   │   │   ├── LaborMarket.cs        # Wage posting and matching
│   │   │   ├── GoodsMarket.cs        # Buying and selling
│   │   │   └── BondMarket.cs         # Bond auction
│   │   ├── Economics/
│   │   │   ├── PricingEngine.cs      # Cost-plus markup calculation
│   │   │   ├── ProductionEngine.cs   # Production function
│   │   │   └── IndicatorCalculator.cs # GDP, inflation, etc.
│   │   └── Data/
│   │       ├── IDataProvider.cs      # Data loading interface
│   │       ├── JsonDataProvider.cs   # JSON file loader
│   │       └── Models/              # Data models for JSON deserialization
│   │           ├── SectorData.cs
│   │           ├── HouseholdData.cs
│   │           ├── GoodsData.cs
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
│   └── Game.Tests/                   # Integration tests that may need Godot
│       └── Game.Tests.csproj
│
├── data/
│   └── base/                         # Base game data (JSON files)
│       ├── economy/
│       │   ├── sectors.json
│       │   ├── goods.json
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
│   ├── DESIGN.md
│   ├── ECONOMIC-MODEL.md
│   ├── MVP.md
│   ├── PRD.md
│   ├── ARCHITECTURE.md
│   ├── MODDING.md
│   └── CONSOLE.md
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
            ├── BalanceSheetPanel reads AllBalanceSheets, refreshes table
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

Example: a `Firm` receives `ILedger` and `IDataProvider` in its constructor. In production, these are the real implementations wired through `ISimulationFactory`. In tests, they are in-memory fakes that allow precise control over inputs and verification of outputs.

### 6.6 Path-Based State Access

All simulation state is queryable by a dot-separated path (e.g., `firms.agriculture.price`). This is used by:
- The console (`query firms.agriculture.price`)
- The data binding system (charts can reference `economy.inflation_rate`)
- Future mod scripts

Implemented via a state tree or registry that maps paths to getters.

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
│   │   ├── goods.json
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

### 9.6 Test Naming Convention

Tests follow the pattern: `[RequirementId]_[Scenario]_[ExpectedResult]`

Examples:
- `FrSim001_AfterGovernmentSpending_SfcIdentityHolds`
- `FrPrc001_WageRiseMatchesProductivityGain_PriceUnchanged`
- `FrBnk001_LoanCreated_DepositsAndLoansIncrease`
- `FrMod001_MissingSectorField_ValidationErrorWithFieldName`
