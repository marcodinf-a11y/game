# Product Requirements Document (PRD)

## 1. Overview

### 1.1 Product

An open source, single-player economic simulation game built on Modern Monetary Theory (MMT). The player governs a sovereign currency-issuing nation, setting fiscal policy and observing how money flows through a stock-flow consistent economy.

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
- The accounting identity (Government balance + Private balance = 0) must hold every tick
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
  - Bond yields (weighted average coupon rate across outstanding bonds; no secondary market in MVP)
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
- Direct transfers must create no direct resource demand — money flows to households who spend via AIDS demand system

#### FR-AGT-002: Central Bank
- Must maintain reserve accounts for commercial banks
- Must maintain the Treasury account
- Must have a policy interest rate (fixed at 0 for MVP)
- Must act as buyer of last resort for government bonds

#### FR-AGT-003: Commercial Banks
- Must hold deposit accounts for households and firms
- Must hold a reserve account at the central bank
- Must create money through lending (endogenous money creation)
- Must assess creditworthiness before lending (income, debt-to-income, collateral)
- Must set lending rate as: CB policy rate + spread + risk premium
- Must buy government bonds at auction
- Must track: reserves, loans outstanding, deposits, bonds held, equity

#### FR-AGT-004: Households (3 Classes)
- Must exist in three classes: low income, middle income, high income
- Each class must have different: consumption patterns, saving rates, reservation wages, debt capacity
- Must consume according to the AIDS (Almost Ideal Demand System) — budget shares across sectors determined by income level and sector prices
- Each class must have distinct AIDS parameters (alpha, beta, gamma) reflecting empirically different consumption patterns
- Price elasticity and income elasticity must emerge from the AIDS parameters, not be hardcoded separately
- Budget shares must sum to 1 for each household class (adding-up constraint)
- Must supply labor to firms and government
- Must accept/reject job offers based on reservation wage
- Must be able to take on debt (consumer loans, mortgages)
- Must pay taxes on income

#### FR-AGT-005: Firms (4 Sectors)
- Must exist in four sectors: agriculture & primary, manufacturing, construction, services
- Each sector must have different input/output mixes
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
- Markup must adjust based on demand relative to capacity

#### FR-PRC-002: Three Inflation Buffers
- Productivity gains must absorb wage increases (if productivity rises with wages, no price pressure)
- Demand slack must absorb spending increases (if idle capacity exists, more output not higher prices)
- Profit margin compression must absorb cost increases (firms may accept lower markup)
- Inflation must only occur when all three buffers are exhausted

#### FR-PRC-003: Inflation Measurement
- Inflation must be measured as percentage change in average price level between ticks
- Price level must be a weighted average across all goods
- Sector-specific price changes must be trackable separately

### 2.4 Labor Market

#### FR-LBR-001: Wage Posting
- Firms must post job openings with an offered wage
- Wages must be influenced by: sector conditions, labor scarcity, firm profitability
- Wages must be sticky downward (slow to decrease even when labor is abundant)

#### FR-LBR-002: Job Acceptance
- Households must have a reservation wage (minimum acceptable) that varies by class
- Households must prefer higher-paying jobs
- Workers must be able to move between sectors over time

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
- Mortgages: long-term, secured by housing, for shelter needs
- Business loans: for firm capital investment and operating costs

#### FR-BNK-004: Debt Service and Default
- Borrowers must make periodic debt service payments (principal + interest)
- If a borrower cannot pay, they must default
- Defaults must result in a bank loss (write-off)

### 2.6 Government Bonds

#### FR-BND-001: Auction-Based Issuance
- Government must issue bonds via auction
- Commercial banks must be able to bid (households acquire bonds post-MVP via secondary market)
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
- Public capital must depreciate and require maintenance

#### FR-INV-002: Private Investment
- Firms must invest in capital goods to maintain/expand capacity
- Investment must be funded from retained profits and/or bank loans
- Capital goods must be produced by the industry sector
- Capital must depreciate over time

### 2.8 Time and Lags

#### FR-TIM-001: Policy Lags
- Tax rate changes must take effect after 1 tick
- Spending level changes must take effect after 1-2 ticks
- Spending reallocation must take effect after 2-3 ticks
- Infrastructure effects must materialize over 6-12 ticks
- Public services effects must materialize over 12-24 ticks

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
- Must display a simple single-province map (cosmetic for MVP)
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

Every functional requirement group must be covered by automated tests. Tests follow the naming convention:

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
- Adding a new good/resource must not require modifying existing good code
- New agent types must be addable through defined interfaces

### 3.5 Data Integrity

#### NFR-DTA-001: SFC Guarantee
- The simulation must never produce an SFC imbalance during normal operation
- If an imbalance is detected, it must be logged and flagged as a bug

### 3.6 Usability

#### NFR-USA-001: Learning Curve
- A new player must be able to start a sandbox game and understand the basic controls within 5 minutes
- The scenario mode must provide enough context for a player to understand the objective without external documentation

#### NFR-USA-002: Feedback Clarity
- The relationship between policy changes and economic outcomes must be visible through the UI (charts, pipeline indicators)
- The player must never be confused about what their policy changes are doing

## 4. MVP Definition of Done

The MVP is complete when:

1. **SFC accounting works** — all transactions are double-entry; balances always sum to zero
2. **Two money circuits visible** — reserves and deposits tracked and displayable
3. **Policy controls work** — player can adjust spending, allocation, and tax rate
4. **Money circuit flows** — government spending creates currency; taxation destroys it
5. **Production runs** — firms produce based on profit-driven decisions; households consume via AIDS demand system
6. **Labor market functions** — firms post wages, households accept/reject, unemployment emerges naturally
7. **Bank lending works** — banks create money via loans; repayments destroy it
8. **Bond auctions work** — government issues bonds; banks bid; CB backstops
9. **Inflation responds correctly** — driven by unit labor costs and demand pressure, not raw spending
10. **Charts update live** — all key indicators chart in real-time
11. **Scenario playable** — one scenario with win/lose conditions completable
12. **Sandbox works** — free experimentation mode with full dashboard
13. **Pause and speed controls work** — player can control simulation flow

## 5. Out of Scope (MVP)

The following are explicitly not part of the MVP:

| Feature | Reason for deferral |
|---|---|
| Job Guarantee program | Build basic employment dynamics first |
| Foreign sector / trade / sectoral balances (3-sector) | Requires foreign sector |
| Multiple economies / AI nations | Focus on domestic dynamics first |
| Multiple provinces / geographic map gameplay | One province proves the economic model |
| Individual agent simulation (population groups only) | Population groups sufficient for MVP |
| Monetary policy (CB rate changes) | CB rate fixed at 0 in MVP |
| Multiple tax types | Single income tax; progressive/sales/wealth taxes later |
| Bond secondary market | No resale in MVP |
| Multiple competing banks | Single aggregate bank in MVP |
| Bank insolvency / systemic crisis mechanics | Defaults tracked but no systemic crisis modeling |
| Collective bargaining / wage contracts | Simple wage posting only |
| 5+ household classes or continuous income spectrum | 3 classes in MVP; continuous spectrum later |
| Sub-sector hierarchy | 4 top-level sectors sufficient for MVP; sub-sector expansion post-MVP |
| Mod loader / mod management UI | Data-driven JSON modding sufficient for MVP |
| Plugin API for code mods | Data-driven JSON modding sufficient for MVP |
| Multiplayer | Single-player focus for MVP |
| Tutorial / guided onboarding | Sandbox and scenario modes provide sufficient onboarding |
| Sound / music | Gameplay mechanics first; polish later |
| Localization | English only for MVP |
