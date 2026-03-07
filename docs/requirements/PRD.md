# Macrostate — Product Requirements Document (PRD)

## 1. Overview

### 1.1 Product

Macrostate is an open source, single-player economic simulation game built on Modern Monetary Theory (MMT). The player governs a sovereign currency-issuing nation, setting fiscal policy and observing how money flows through a stock-flow consistent economy.

### 1.2 Goals

1. Provide an engaging economic simulation where MMT dynamics emerge naturally from gameplay
2. Let players experience sovereign currency mechanics firsthand
3. Be detailed enough for university-level educational use
4. Support modding from day one
5. Be fully open source

### 1.3 Target Audience

| Persona | Description | Needs |
|---|---|---|
| **The Curious Gamer** | Enjoys simulation/strategy games (Victoria 3, Democracy 4, Factorio). Interested in economics but not an expert. | Fun gameplay loop, clear feedback, gradual learning curve |
| **The Economics Student** | Studying economics at university. Wants to see MMT concepts in action, not just in textbooks. | Accurate model, inspectable internals, ability to test theories |
| **The Modder** | Wants to extend the game — new scenarios, new mechanics, total conversions. | Clean data format, extensible architecture, documentation |
| **The MMT Enthusiast** | Already understands MMT. Wants a tool to demonstrate concepts to others. | Correct implementation of MMT, visible accounting identities, scenario creation |

## 2. Functional Requirements

### 2.1 Simulation Engine

#### FR-SIM-001: Stock-Flow Consistent Accounting
- The simulation must use double-entry bookkeeping for all financial transactions
- Every monetary flow must have an explicit source and destination
- The accounting identity (Government balance + Private balance = 0) must hold every tick. The central bank is consolidated with the government sector for SFC purposes (Treasury + CB = government sector); the private sector includes households, firms, and commercial banks. This is standard SFC convention per Godley & Lavoie (2012).
- An SFC consistency check must be runnable at any time and must pass

#### FR-SIM-002: Two Money Circuits
- The simulation must track central bank money (reserves) and deposit money (bank deposits) as separate layers
- Government spending must flow: Treasury → bank reserves → recipient deposits
- Taxation must flow: deposits → reserves → Treasury (destruction)
- Non-bank private agents (households, firms) must only interact with deposit money
- The central bank and Treasury must only interact with reserve money
- Commercial banks must operate in both circuits, holding reserve accounts at the central bank and maintaining deposit accounts for customers

#### FR-SIM-003: Monthly Tick Processing
- Each simulation tick represents one month
- Ticks process in defined order: Government → Production → Market → Financial → Accounting
- All balance sheets must be updated and consistent after each tick

#### FR-SIM-004: Economic Indicators
- The simulation must calculate and expose at minimum:
  - Employment rate, unemployment rate
  - Inflation rate (price level change)
  - GDP (total output value)
  - Government fiscal balance (deficit/surplus)
  - Private sector net savings
  - Capacity utilization per sector
  - Unit labor costs per sector
  - Public sector employment share
  - Private debt level
  - Bank reserves
  - Bond yields (weighted average coupon rate across outstanding bonds)
  - Savings rate
  - Wage growth

### 2.2 Economic Agents

#### FR-AGT-001: Government (Treasury)
- Must spend currency into existence (creating reserves and deposits)
- Must collect taxes (destroying deposits and reserves)
- Must issue bonds via auction
- Must track cumulative deficit/surplus
- Spending must be allocatable to: infrastructure, public services, direct transfers
- Must employ workers directly (public sector employment) competing in the labor market alongside firms
- Must procure goods and services from private sectors based on spending allocation (infrastructure -> construction/manufacturing demand, public services -> services/manufacturing demand)
- Must track public sector employment separately from private employment
- Government wage rates must be set by a data-driven pay scale that adjusts more slowly than private sector wages (civil service stickiness)
- Government procurement must create demand in private sectors: infrastructure spending generates construction and manufacturing demand; public services spending generates services and manufacturing demand
- Direct transfers must create no direct resource demand — money flows to households who spend via AIDS demand system
- Government spending must never be rejected or constrained based on Treasury account balance (the real constraint is inflation, not money)
- Government must always pay bond interest and principal when due — sovereign default on domestic-currency debt must not be modeled

#### FR-AGT-001a: Sovereign Currency Invariants
The following invariants must hold unconditionally in the simulation engine. They are not policy choices — they are operational properties of a sovereign currency-issuing government per MMT:

- **INV-GOV-001: No financial constraint on spending.** The Treasury account balance never constrains government spending. The government, as currency issuer, can always spend by crediting bank reserve accounts. The real constraint on government spending is inflation (available real resources), not money. (See ECONOMIC-MODEL.md, Government section.)
- **INV-GOV-002: No sovereign default.** The government, as currency issuer, always pays bond interest and principal when due. Sovereign default on domestic-currency-denominated debt is never an operational necessity. The game does not model voluntary default. (See ECONOMIC-MODEL.md, Bonds section.)

Political fiscal constraints (debt ceilings, balanced budget rules) are self-imposed policy choices modeled in the policy constraint layer, not in the engine. See GAME-DESIGN.md Section 5.

#### FR-AGT-002: Central Bank
- Must maintain reserve accounts for commercial banks
- Must maintain the Treasury account
- Must have a policy interest rate
- Must act as buyer of last resort for government bonds

#### FR-AGT-003: Commercial Banks
- Must hold deposit accounts for households and firms
- Must hold a reserve account at the central bank
- Must create money through lending (endogenous money creation)
- Must assess creditworthiness before lending (income, debt-to-income, collateral)
- Must set lending rate as: CB policy rate + spread + risk premium
- Must buy government bonds at auction
- Must track: reserves, loans outstanding, deposits, bonds held, equity

#### FR-AGT-004: Households
- Must exist in multiple income classes (low income, middle income, high income, and potentially more)
- Each class must have different: consumption patterns, saving rates, reservation wages, debt capacity
- Must consume according to the AIDS (Almost Ideal Demand System) — budget shares across sectors determined by income level and sector prices
- Each class must have distinct AIDS parameters (alpha, beta, gamma) reflecting empirically different consumption patterns
- Price elasticity and income elasticity must emerge from the AIDS parameters, not be hardcoded separately
- Budget shares must sum to 1 for each household class (adding-up constraint)
- Disposable income for each household class must be calculated as: post-tax income + available credit - debt service payments
- AIDS parameters must be validated on load: adding-up (`SUM(alpha)=1`, `SUM(gamma)=0`, `SUM(beta)=0`), homogeneity (`SUM_j(gamma_ij)=0`), and symmetry (`gamma_ij=gamma_ji`) constraints must hold
- Must supply labor to firms and government
- Must accept/reject job offers based on reservation wage
- Must be able to take on debt (consumer loans, mortgages)
- Must pay taxes on income

#### FR-AGT-005: Firms (4 Sectors)
- Must exist in four sectors: agriculture & primary, manufacturing, construction, services
- Each sector must have different input/output mixes
- Must use Leontief (fixed-proportion) production functions: each unit of output requires fixed quantities of each input, with no substitution between inputs
- Must use inter-sector input-output coefficients loaded from data files to define supply chain linkages (ADR-0009)
- Must make profit-driven production decisions (estimate demand, consider costs)
- Must post wages and hire workers
- Must set prices using cost-plus markup with demand adjustment
- Must use unit labor costs (wages/productivity) not raw wages in pricing
- Must invest in capital from retained profits and/or bank loans (capital goods produced by manufacturing sector)
- Must hold inventory of unsold goods
- Must pay wages and taxes

### 2.3 Pricing and Inflation

#### FR-PRC-001: Cost-Plus Markup Pricing
- Prices must be calculated as: (unit labor cost + unit material cost) × (1 + markup)
- Unit labor cost must equal: total wages / total output (not raw wage rate)
- Markup must adjust toward a target via partial adjustment (Caiani et al. 2016): `CurrentMarkup += speed × (TargetMarkup - CurrentMarkup)`
- `TargetMarkup = BaseMarkup × DemandAdjustmentFactor × SupplyPressureFactor`
- `DemandAdjustmentFactor = TargetInventoryRatio / ActualInventoryRatio` — greater than 1 when inventories deplete (demand exceeds supply), less than 1 when inventories pile up (supply exceeds demand)
- `speed = markupUpwardSpeed` when target > current, `markupDownwardSpeed` when target < current (asymmetric adjustment per ADR-0010)
- CurrentMarkup must never fall below MinimumMarkup
- Per-sector parameters (`markupUpwardSpeed`, `markupDownwardSpeed`, `baseMarkup`, `minimumMarkup`, `targetInventoryRatio`) must be loaded from sector data

#### FR-PRC-002: Inflation Buffers and Supply-Side Markup Pressure
- Productivity gains must absorb wage increases (if productivity rises with wages, no price pressure)
- Demand slack must absorb spending increases (if idle capacity exists, more output not higher prices)
- Profit margin compression must absorb cost increases (firms may accept lower markup)
- Demand-pull inflation must only occur when all three buffers above are exhausted
- Supply-side markup pressure (seller's inflation) must operate independently of the demand buffers: when input availability drops, firms must raise markups using a supply pressure factor (normal input availability / actual input availability) (ADR-0010)
- The supply pressure factor must use the same asymmetric adjustment speeds as demand-driven markup changes

#### FR-PRC-003: Rationing
- When aggregate demand for a sector's output exceeds available supply (inventory + current production), supply must be rationed across buyers
- Unmet demand must feed into the next tick's price adjustments via the pricing engine (demand pressure signal)
- Rationing must apply to both household consumption demand and government procurement demand

#### FR-PRC-004: Inflation Measurement
- Inflation must be measured as percentage change in average price level between ticks
- Price level must be a weighted average across all sectors
- Sector-specific price changes must be trackable separately

### 2.4 Labor Market

#### FR-LBR-001: Wage Posting
- Firms must post job openings with an offered wage
- Wage growth must follow the Godley & Lavoie (2012) wage equation: `ΔW/W = Ω₀ + Ω₁ × (ΔPr/Pr) - Ω₂ × UR + Ω₃ × π`, where `ΔPr/Pr` is sector productivity growth, `UR` is sector unemployment rate, and `π` is the inflation rate
- Ω coefficients (Ω₀ autonomous growth, Ω₁ productivity pass-through, Ω₂ unemployment dampening, Ω₃ inflation catch-up) must be loaded from sector data — downward wage rigidity emerges from calibration, not a separate mechanism
- Government wages must use distinct Ω coefficients reflecting civil service stickiness (slower adjustment)

#### FR-LBR-002: Job Acceptance and Inter-Sector Mobility
- Households must have a reservation wage (minimum acceptable) that varies by class
- Households must prefer higher-paying jobs
- Unemployed workers must be available to any sector immediately
- Employed workers must search for better-paying jobs with some probability each tick (on-the-job search; Caiani et al. 2016) and switch if they find a higher-wage offer in another sector
- Inter-sector mobility must be governed by a sector-pair mobility matrix reflecting skills transferability (Artuc et al. 2010): some transitions are easier (manufacturing → construction) than others (agriculture → services). Mobility parameters must be loaded from data files.
- Workers switching sectors must experience a productivity penalty during a transition period of 1-2 ticks (retraining/relocation; Dawid et al. 2019), reflecting sector-specific human capital (Dix-Carneiro 2014)
- Sector-pair mobility parameters, on-the-job search probability, and transition productivity penalty must be loaded from data files

#### FR-LBR-003: Unemployment
- Unemployment must emerge naturally when firms don't post enough jobs or wages are below reservation wages
- Unemployed households must have zero wage income
- Employment/unemployment rates must be trackable

### 2.5 Banking and Credit

#### FR-BNK-001: Endogenous Money Creation
- Bank loans must create new deposits (not lend from existing deposits)
- Loan creation must simultaneously create: a deposit (bank liability) and a loan receivable (bank asset)
- Loan repayment must destroy deposits and extinguish the corresponding loan

#### FR-BNK-002: Creditworthiness Assessment
- Banks must evaluate: borrower income, existing debt-to-income ratio, and collateral (for secured loans)
- Lending rate must be: CB policy rate + bank spread + risk premium
- Higher risk borrowers must face higher rates or rejection

#### FR-BNK-003: Loan Types
- Consumer loans: short-term, unsecured, for household expenses
- Mortgages: long-term, secured by housing, for shelter needs. _MVP simplification: housing is not modeled as an explicit asset class. Mortgage collateral value is a fixed notional amount per household class. A housing market with price dynamics is a post-MVP enhancement._
- Business loans: for firm capital investment and operating costs

#### FR-BNK-004: Debt Service and Default
- Borrowers must make periodic debt service payments (principal + interest)
- If a borrower cannot pay, they must default
- Defaults must result in a bank loss (write-off)

### 2.6 Government Bonds

#### FR-BND-001: Auction-Based Issuance
- Government must issue bonds via auction
- Commercial banks must be able to bid at primary auctions
- Households may acquire bonds via secondary market (full vision — not in MVP)
- Bond interest rate must be determined by auction demand
- Central bank must act as buyer of last resort (can buy unsold bonds)

#### FR-BND-002: Bond Properties
- Bonds must have: face value, coupon rate (set at auction), maturity
- Government must pay interest on outstanding bonds each period
- Interest payments must create new currency (government spending)

### 2.7 Investment

#### FR-INV-001: Public Investment
- Infrastructure spending must increase productive capacity over time
- Public services spending must increase labor productivity over time
- Public capital must depreciate using geometric depreciation: `K(t+1) = K(t) × (1 - δ) + I(t)`, where δ is the per-category depreciation rate from data files and I(t) is new public investment (Godley & Lavoie 2012)

#### FR-INV-002: Private Investment
- Firms must invest in capital goods to maintain/expand capacity
- Investment must be funded from retained profits and/or bank loans
- Capital goods must be produced by the manufacturing sector (enters sector demand via the Leontief I-O matrix; see ADR-0009)
- Private capital must depreciate using geometric depreciation: `K(t+1) = K(t) × (1 - δ) + I(t)`, where δ is the per-sector `capitalDepreciationRate` from sectors.json and I(t) is new private investment (Godley & Lavoie 2012)

### 2.8 Time and Lags

Where a lag is specified as a range (e.g., 1-2 ticks), the actual value for each instance must be drawn uniformly from [min, max] using the seeded `IRandom` interface. Min and max values are stored in data files and are moddable.

#### FR-TIM-001: Policy Lags
- Tax rate changes must take effect after 1 tick
- Spending level changes must take effect after 1-2 ticks
- Spending reallocation must take effect after 2-3 ticks
- Infrastructure effects must materialize over 6-12 ticks
- Public services effects must materialize over 12-24 ticks
- If a policy parameter is changed while a previous change for the same parameter is still pending, the new value replaces the pending value and the lag timer resets (ADR-0002)

#### FR-TIM-002: Economic Lags
- Wage adjustments: 1-3 ticks
- Price adjustments: 1-2 ticks
- Hiring/firing: 1-2 ticks
- Investment to capacity: 3-6 ticks
- Household spending adjustment: 1 tick

#### FR-TIM-003: Visual Pipeline
- Pending policy changes must be visible in the UI with estimated time to effect
- The player must be able to see what changes are "in the pipeline"

### 2.9 Player Controls

#### FR-CTL-001: Policy Levers
- Player must be able to set total government spending level
- Player must be able to allocate spending across: infrastructure, public services, direct transfers
- Spending allocation determines both public sector employment costs (wages for government workers) and procurement demand placed on private sectors. The total spending level is the budget; the allocation determines how it divides between government payroll and sector procurement within each category.
- Player must be able to set a single income tax rate
- All controls must be adjustable at any time (including while paused)

#### FR-CTL-002: Time Controls
- Player must be able to pause the simulation
- Player must be able to resume the simulation
- Player must be able to set speed: 1x, 2x, 5x
- Player must be able to advance one tick while paused

### 2.10 Game Modes

#### FR-GMD-001: Sandbox Mode
- Must allow unlimited free play with no win/lose conditions
- Must display a dashboard with all key economic indicators
- Must allow the player to experiment freely with all policy levers

#### FR-GMD-002: Scenario Mode
- Must support at least one predefined scenario with:
  - A starting economic state
  - Specific objectives (e.g., employment target, inflation ceiling)
  - A time limit
  - Fail conditions (e.g., hyperinflation, economic collapse)
  - Win detection when objectives are met
  - Lose detection when fail conditions are triggered
- Scenario definitions must be loaded from JSON data files

### 2.11 User Interface

#### FR-UI-001: Policy Panel
- Must provide sliders or inputs for all player policy controls
- Must show current values and pending changes
- Must be accessible while the simulation is running or paused

#### FR-UI-002: Live Charts
- Must display real-time line charts for key economic indicators
- Charts must update each tick
- Must include at minimum: employment rate, inflation rate, GDP, government balance, private savings, sector output

#### FR-UI-003: Balance Sheet View
- Must display SFC balance sheets for each sector
- Must show: assets, liabilities, net position
- Must update each tick

#### FR-UI-004: Resource Utilization
- Must display per-sector capacity utilization
- Must clearly indicate when sectors are at or near capacity

#### FR-UI-005: Map View
- Must display a map with provinces
- Must establish the visual direction for future geographic expansion

#### FR-UI-006: Time Controls
- Must display play/pause button
- Must display speed selector
- Must show current date (month/year)

#### FR-UI-007: Policy Pipeline
- Must show pending policy changes with estimated time to effect
- Must visually distinguish between: enacted, in pipeline, taking effect

### 2.12 Console

#### FR-CON-001: Console Access
- Must be togglable with a key (default `~`)
- Must be available in all builds (including release)
- Must not interfere with gameplay when closed

#### FR-CON-002: Query Commands
- Must support querying any simulation variable
- Must support querying balance sheets for all sectors
- Must support SFC consistency checks

#### FR-CON-003: Write Commands
- Must support setting government policy values
- Must support one-time spend and tax operations
- Write operations affecting money stocks must maintain SFC consistency

#### FR-CON-004: Time Commands
- Must support: pause, resume, speed change, single tick advance

#### FR-CON-005: Scripting
- Must support executing script files containing sequences of commands
- Must support comments in script files

#### FR-CON-006: Help System
- Must provide help for all commands
- Must list available commands by category

### 2.13 Data and Modding

#### FR-MOD-001: Data-Driven Design
- All game data must be loaded from external JSON files
- No economic parameters, sector definitions, goods, or scenarios may be hardcoded — not even as temporary defaults to be replaced later
- The base game data must reside in `data/base/`
- The data loading system (`IDataProvider`, `JsonDataProvider`, data model classes) must be among the first components implemented, before any agent or economic logic

#### FR-MOD-002: Data Schema
- All data files must conform to a defined schema
- Data must be validated on load with clear error messages for malformed data

#### FR-MOD-003: Base Game as Mod
- The base game must load its data through the same system that mods will use
- The data file format must be identical for base game and mods

### 2.14 Testability Contract

Every simulation-engine functional requirement group must be covered by automated tests. Presentation-layer FR groups (FR-UI, FR-CON, FR-GMD, FR-CTL-002) use manual verification for Godot-specific behavior, with automated tests where the logic is separable from the engine (e.g., command parsing, scenario win/lose detection). See `docs/technical/UI-TESTING-ROADMAP.md` for the path toward full automated UI testing.

Tests follow the naming convention:

```
[RequirementId]_[Scenario]_[ExpectedResult]
```

Examples:
- `FrSim001_AfterGovernmentSpending_SfcIdentityHolds`
- `FrPrc001_WageRiseWithProductivityGain_NoPriceIncrease`
- `FrMod001_MissingRequiredField_ProducesClearError`

#### Test Coverage Mapping

| FR Group | Unit Tests | Integration Tests | Property-Based Tests |
|---|---|---|---|
| FR-SIM (Simulation) | Ledger, SfcChecker, BalanceSheet | Full tick cycle, multi-tick SFC | SFC identity holds for all seeds |
| FR-AGT (Agents) | Each agent type in isolation | Agent interactions per tick phase | — |
| FR-PRC (Pricing) | PricingEngine, cost calculations | Price response to demand/cost changes | No negative prices for any input |
| FR-LBR (Labor) | LaborMarket matching, wage posting | Hiring/unemployment dynamics | Employment rate in [0, 1] |
| FR-BNK (Banking) | Loan creation, debt service, defaults | Money creation/destruction cycle | Money stock consistency |
| FR-BND (Bonds) | Auction mechanics, interest payments | Bond lifecycle end-to-end | — |
| FR-INV (Investment) | Capital depreciation, investment decisions | Capacity change over time | — |
| FR-TIM (Time/Lags) | Policy queue, lag calculations | Policy effect after correct delay | — |
| FR-MOD (Data/Modding) | JSON loading, validation, error handling | Simulation init from data files | — |

## 3. Non-Functional Requirements

### 3.1 Performance

#### NFR-PRF-001: Tick Performance
- A single simulation tick must complete in under 100ms on a mid-range desktop (to support 5x speed smoothly)
- The simulation must be decoupled from rendering (simulation tick rate independent of frame rate)

#### NFR-PRF-002: UI Responsiveness
- The UI must remain responsive during simulation (no freezing during tick processing)
- Charts must update smoothly without visible lag

### 3.2 Portability

#### NFR-PRT-001: Platform Support
- Must run on Linux, Windows, and macOS
- Must use Godot's native cross-platform export

### 3.3 Code Quality

#### NFR-CQA-001: Separation of Concerns
- Simulation logic must be completely independent of rendering/UI
- The simulation engine must be testable without Godot (pure C# with no Godot dependencies)
- UI must read from simulation state, not duplicate logic

#### NFR-CQA-002: Testability
- Core economic model must have unit tests
- SFC consistency must be verifiable via automated tests
- Each agent type must be independently testable
- Zero Godot dependencies enforced at the project level (`Simulation.csproj` must not reference Godot assemblies)
- All agents receive dependencies via constructor injection (`IDataProvider`, `ILedger`)
- An `InMemoryDataProvider` must exist for tests, allowing data to be registered in memory instead of read from disk
- The simulation must be constructable and runnable headlessly (no UI, no Godot, pure C#)
- All randomness must use a seeded `IRandom` interface for deterministic, reproducible tests
- The Game Controller must record a policy input log — a list of `(tick, command)` pairs — enabling deterministic replay for SFC error recovery (ADR-0004) and post-MVP save/load (ADR-0006)

#### NFR-CQA-003: Test-Driven Development Methodology
- All simulation engine code must be developed using TDD (red-green-refactor cycle)
- Every public method must have a corresponding test written before its implementation
- Each FR-* requirement must be traceable to at least one automated test
- Every bug fix must include a regression test that fails before the fix and passes after

#### NFR-CQA-004: Test Coverage and Categories
- Minimum 80% line coverage for the simulation engine
- Three test categories:
  - **Unit tests:** isolated component tests (single class, mocked dependencies)
  - **Integration tests:** full tick cycles, money flow end-to-end, policy-to-outcome chains
  - **Property-based tests:** SFC invariants hold for arbitrary inputs, no negative prices, employment rate in [0, 1]
- A headless simulation test harness must support multi-year runs (100+ ticks) without Godot

#### NFR-CQA-005: Data Validation Testing
- JSON schema validation must run at load time for all data files
- Tests must verify that all base game data files (`data/base/`) are valid and complete
- Tests must verify that malformed data produces clear, actionable error messages
- Test JSON fixtures must exist for each data type (minimal valid, full base copy, invalid/malformed, specific economic scenarios)

### 3.4 Extensibility

#### NFR-EXT-001: Modular Design
- Adding a new sector must not require modifying existing sector code
- Adding a new household class must not require modifying existing class code
- Adding a new sector/sub-sector must not require modifying existing sector code
- New agent types must be addable through defined interfaces

### 3.5 Data Integrity

#### NFR-DTA-001: SFC Guarantee
- The simulation must never produce an SFC imbalance during normal operation
- If an imbalance is detected, it must be logged and flagged as a bug
- On SFC failure, the simulation must pause and present the player with a choice: continue with the inconsistent state, or restore to the last consistent state via deterministic replay (ADR-0004)

### 3.6 Usability

#### NFR-USA-001: Learning Curve
- A new player must be able to start a sandbox game and understand the basic controls within 5 minutes
- The scenario mode must provide enough context for a player to understand the objective without external documentation

#### NFR-USA-002: Feedback Clarity
- The relationship between policy changes and economic outcomes must be visible through the UI (charts, pipeline indicators)
- The player must never be confused about what their policy changes are doing

For MVP scoping, definition of done, and out-of-scope features, see [MVP-SCOPE.md](MVP-SCOPE.md).
