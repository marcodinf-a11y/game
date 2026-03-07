# Macrostate — Game Design Document

## Concept

Macrostate is an open source economic simulation grounded in Modern Monetary Theory (MMT). The player governs a sovereign currency-issuing nation in a modern/contemporary world with multiple AI-driven economies.

The game aims to let players experience MMT dynamics firsthand — learning not through text, but through the consequences of their fiscal decisions.

## Design Pillars

1. **Victoria 3-style gameplay** — real-time with pause, province-based map, deep economic simulation
2. **MMT as the economic model** — not mainstream economics. The simulation is built on MMT's framework from the ground up
3. **Educational but subtle** — players learn MMT by playing, not by reading textbooks. An optional explainer mode may be added later
4. **Agent-based simulation** — emergent economic behavior from interacting populations, not top-down formulas

## Player Role

The player is the **Government / Sovereign** of one nation:

- Issues the sovereign currency
- Sets fiscal policy (spending levels, allocation, taxation)
- Manages employment and public programs
- Navigates trade relationships with AI nations

Other nations are AI-controlled and react to the player's policies, creating geopolitical and economic dynamics.

## MMT Concepts to Model

### 1. Currency Issuance & Taxation

The foundational mechanic. Government spending creates new currency; taxation removes currency from circulation. The government, as currency issuer, cannot "run out" of its own money. The budget deficit is not inherently bad — it represents net money added to the private sector.

**In-game:** The player sees money flow from government → firms/households via spending, and back from households/firms → government (destruction) via taxation. The government balance sheet is always visible.

### 2. Job Guarantee

A public employment program that offers a job to anyone willing and able to work at a fixed wage. This acts as an automatic stabilizer:

- In downturns, more people move to the Job Guarantee pool (government spending automatically increases)
- In booms, private sector hires from the JG pool (government spending automatically decreases)

The JG replaces unemployment as the "buffer stock" — instead of a buffer stock of unemployed, there's a buffer stock of employed.

**In-game:** The player can enable/configure a Job Guarantee program. Employment dynamics shift from boom/bust cycles to stable full employment with price stability.

### 3. Sectoral Balances

The three sectors of the economy (government, domestic private, foreign) must balance to zero:

```
Government Balance + Private Balance + Foreign Balance = 0
```

A government deficit equals a private sector surplus (to the penny). A trade deficit means the foreign sector is in surplus.

**In-game:** A clear sectoral balance visualization shows how the three sectors relate. The player sees that a government surplus necessarily means private sector deficit (draining savings).

### 4. Inflation as Resource Constraint

The real constraint on government spending is not money but available real resources (labor, materials, productive capacity). When spending pushes demand beyond the economy's ability to produce:

- Prices rise (inflation)
- Bottlenecks appear in specific sectors
- Resource competition intensifies

When there is slack in the economy (unemployed workers, idle factories), spending can increase output without causing inflation.

**In-game:** Resource utilization meters show how close each sector is to capacity. Spending into slack = more output. Spending into full capacity = inflation.

### 5. Political Fiscal Constraints

Real-world governments often impose political rules on their own spending — the US debt ceiling, Germany's Schuldenbremse (debt brake), balanced budget amendments, Maastricht criteria. These are not operational necessities — the currency-issuing government can always spend — they are self-imposed political choices with real economic consequences.

A **policy constraint layer** sits between the player's decisions and the simulation engine. The engine itself never blocks spending (see ECONOMIC-MODEL.md, Government section). Political constraints are optional rules that restrict what the player can do, enforced by scenario configuration or player choice:

- **Debt ceiling:** Cap on outstanding bonds. Hitting it triggers a political crisis event; the player must raise the ceiling or cut spending.
- **Balanced budget rule:** Spending cannot exceed tax revenue. Forces pro-cyclical austerity.
- **Deficit limit (e.g., Maastricht 3% of GDP):** Caps the deficit as a share of GDP.

These constraints create gameplay tension and demonstrate a core MMT insight: self-imposed financial rules can cause unnecessary harm (unemployment, recessions) by preventing the government from responding to real economic conditions. The real constraint is always inflation, not an arbitrary number.

**In-game:** Players can observe the consequences of enabling or removing these rules — comparing outcomes with and without political constraints in otherwise identical economies.

## Gameplay

### Flow

**Real-time with pause**, inspired by Paradox grand strategy games. Time flows continuously (representing months/quarters), and the player can:

- Pause to assess the situation and adjust policies
- Adjust game speed (1x, 2x, 5x)
- Make policy changes at any time (some take effect immediately, others with a lag)

### Goals & Scenarios

**Objective/scenario-based** with both guided and freeform play:

- **Sandbox mode:** No win/lose conditions. Free experimentation with a dashboard tracking key metrics. Ideal for learning and exploration.
- **Scenario mode:** Predefined scenarios with specific objectives and fail conditions. Examples:
  - "Achieve 95% employment with inflation below 5% within 10 years"
  - "Recover from a financial crisis without causing hyperinflation"
  - "Manage a commodity supply shock"
  - "Transition the economy from agricultural to industrial"

### Lose Conditions (Scenario Mode)

- **Hyperinflation** — price level spirals out of control
- **Mass unemployment** — employment falls below a critical threshold
- **Economic collapse** — GDP contracts beyond recovery

## Simulation Model

### Agent-Based with Populations

The economy is modeled using **household classes** that behave as collective agents. Each class has aggregate behavior emerging from simple rules:

- **Households** — earn wages, consume goods, save surplus, pay taxes
- **Firms (by sector)** — hire workers, produce goods/services, sell output, pay wages, pay taxes
- **Banks** — hold deposits, facilitate payments, potentially create credit
- **Government** — spend currency into existence, collect taxes (destroy currency), run programs

In later versions, these groups evolve into individual agents with heterogeneous behavior.

### Resource Types

- **Labor** — the primary resource, supplied by households
- **Raw materials** — used in production, finite supply
- **Capital goods** — machinery, infrastructure, built by industry
- **Consumer goods** — food, manufactured products, services

### Production

Each sector uses fixed-proportion inputs (Leontief production function) — inputs cannot substitute for one another. Inter-sector linkages create supply chain dynamics:

| Sector | Primary inputs | Output |
|---|---|---|
| Agriculture & Primary | Labor, land | Food, raw materials |
| Manufacturing | Labor, raw materials, capital goods | Manufactured goods, capital goods |
| Construction | Labor, manufactured materials, capital goods | Built structures, infrastructure |
| Services | Labor, capital goods | Services (retail, transport, healthcare, etc.) |

## Map & Visuals

### Map

- **2D top-down** geographic map with provinces/regions
- Provinces contain resources, populations, and production facilities
- Visual overlays for economic data (employment heat map, price levels, production output)
- Modern/contemporary art style

### UI

The game is UI-heavy, reflecting its simulation nature:

- **Policy panel** — controls for all player decisions
- **Economic dashboard** — live charts and indicators
- **Map overlays** — visual representation of economic data on the map
- **Sectoral views** — detailed breakdowns by economic sector
- **Time controls** — play/pause and speed adjustment

## Setting

**Modern / Contemporary.** The game world reflects present-day economic structures and challenges. This makes the MMT concepts directly relatable to real-world economic debates about government spending, deficits, inflation, and employment.

Nations in the game are fictional but recognizable — players can draw parallels to real economies without the game making specific political claims.

## Technical Considerations

### Engine: Godot with C#

Chosen for:
- Open source (MIT license) — aligns with project values
- Strong 2D support — ideal for the map-based, UI-heavy style
- Built-in UI system (Control nodes) — essential for the data-heavy interface
- C# scripting — leverages the developer's primary language
- Large community — important for finding help and resources
- Cross-platform desktop export — Linux, Windows, macOS out of the box

### Architecture Considerations

- Simulation engine should be decoupled from rendering (testable independently)
- Economic model should be modular (easy to add new sectors, policies, mechanics)
- **Data-driven from day one** — all game data loaded from JSON files, never hardcoded. See [MODDING.md](../technical/MODDING.md).
- **Mod support from day one** — the base game is structured like a mod. Two-tier modding: data overrides (casual) and plugin API (advanced). Multiple mods combinable with dependency resolution.
- Save/load system for game state
