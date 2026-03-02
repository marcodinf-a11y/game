# MVP Scope

## Overview

The MVP is the smallest playable version that demonstrates MMT's core mechanics. It implements a single closed economy with an SFC framework, two money circuits, three household classes, endogenous bank credit, and enough UI to create a meaningful gameplay loop.

## What's IN the MVP

### Economy

**Single closed economy** — one nation, one province, no foreign sector.

**SFC Framework** — full double-entry bookkeeping. Every transaction tracked. All balances sum to zero.

**Two money circuits:**
- Central bank money (reserves): between Treasury, central bank, and commercial banks
- Deposit money: between banks, households, and firms

**Sectors and agents:**

| Agent | Role |
|---|---|
| Government (Treasury) | Currency issuer. Spends money into existence via reserve creation, collects taxes (destroys money). Issues bonds. |
| Central Bank | Maintains reserve accounts. Sets policy interest rate (fixed at 0 for MVP). Buyer of last resort for bonds. |
| Commercial Banks | Hold deposits, create loans (endogenous money), buy government bonds. Single aggregate bank in MVP. |
| Households (Low income) | High consumption share. Spend mostly on survival/shelter. May take on debt. Low savings. |
| Households (Middle income) | Moderate consumption/saving. Comfort-level spending. Mortgages for housing. |
| Households (High income) | Low consumption relative to income. Significant savings. Buy bonds. Luxury spending. |
| Firms (Agriculture) | Produce food and raw materials. Labor + land intensive. |
| Firms (Industry) | Produce manufactured goods and capital goods. Labor + materials + capital intensive. |
| Firms (Services) | Produce services. Labor + capital intensive. |

**Resources:**
- Labor (supplied by households)
- Raw materials (produced by agriculture, consumed by industry)
- Capital goods (produced by industry, used by all firm types)
- Consumer goods: food (agriculture), manufactured goods (industry), services (services)

### MMT Concepts

**1. Currency issuance & taxation**
- Government spending creates new reserves + deposits
- Taxation destroys deposits + reserves
- Two money circuits explicitly visible
- Government balance + Private balance = 0 (enforced by SFC)

**2. Inflation as resource constraint**
- Cost-plus markup pricing based on unit labor costs (not raw wages)
- Three inflation buffers: productivity gains, demand slack, margin compression
- Spending into slack → more output, no inflation
- Spending beyond capacity → unit costs rise → prices rise
- Resource utilization visible per sector

### Player Controls

Three policy levers:

1. **Total government spending** — how much currency to spend per period
2. **Spending allocation** — where the money goes:
   - Infrastructure (increases productive capacity over time)
   - Public services (education, health — improves labor productivity)
   - Direct transfers (money directly to households)
3. **Tax rate** — single income tax rate applied to households and firms

### Simulation Model

**Pricing:** cost-plus markup with demand adjustment
- Price = (unit labor cost + unit material cost) × (1 + markup)
- Unit labor cost = wages / productivity
- Markup adjusts with demand pressure

**Firms:** profit-driven
- Estimate demand, set production targets, post wages, hire workers, produce, set prices

**Households:** hierarchical needs
- Survival (food, basics) → Shelter (housing) → Comfort (goods) → Luxury (services)
- Poor spend most on survival; rich have surplus for savings/bonds

**Labor market:** wage posting
- Firms post jobs with wages; households accept/reject based on reservation wage
- Wages rise in tight markets, sticky downward in loose markets

**Banks:** endogenous money creation
- Creditworthiness-based lending (income, debt-to-income, collateral)
- Lending rate = CB rate (0) + bank spread + risk premium
- Loans create new deposits; repayments destroy them

**Government bonds:** auction-based
- Banks and wealthy households bid
- Central bank as buyer of last resort
- Interest payments create new currency

**Time lags:** short with visual feedback
- Tax changes: 1 month
- Spending changes: 1-2 months
- Infrastructure effects: 6-12 months
- Public services effects: 12-24 months

### Gameplay

- **Real-time with pause** — monthly ticks, adjustable speed (1x, 2x, 5x)
- **Sandbox mode** — free experimentation with score dashboard
- **One scenario** — e.g., "Achieve 95% employment with inflation below 5% within 10 years"
  - Fail conditions: hyperinflation (>25%) or economic collapse (employment <50%)

### UI

- **Policy panel** — sliders/inputs for spending level, allocation %, tax rate
- **Live charts** — employment, inflation, GDP, government balance, private savings, output by sector, unit labor costs, private debt, bond yields
- **Resource utilization bars** — per-sector capacity utilization
- **Balance sheet view** — SFC balance sheets for each sector
- **Simple map view** — cosmetic single-province map
- **Time controls** — play, pause, speed
- **Policy pipeline** — visual indicator of pending policy effects

## What's NOT in the MVP

| Feature | Reason for deferral |
|---|---|
| Job Guarantee | Build basic employment dynamics first |
| Sectoral balances (3-sector) | Requires foreign sector |
| Multiple economies / trade | Focus on domestic dynamics first |
| Multiple provinces | One province proves the economic model |
| Individual agents | Population groups sufficient for MVP |
| Monetary policy (rate changes) | CB rate fixed at 0 in MVP |
| Multiple tax types | Single income tax; progressive/sales/wealth taxes later |
| Bond secondary market | No resale in MVP |
| Multiple competing banks | Single aggregate bank in MVP |
| Bank insolvency mechanics | Defaults tracked but no systemic crisis modeling |
| Collective bargaining | Simple wage posting only |
| 5+ household classes | 3 classes in MVP; continuous spectrum later |

## Definition of Done

The MVP is complete when:

1. **SFC accounting works** — all transactions are double-entry; balances always sum to zero
2. **Two money circuits visible** — reserves and deposits tracked and displayable
3. **Policy controls work** — player can adjust spending, allocation, and tax rate
4. **Money circuit flows** — government spending creates currency; taxation destroys it
5. **Production runs** — firms produce based on profit-driven decisions; households consume by needs hierarchy
6. **Labor market functions** — firms post wages, households accept/reject, unemployment emerges naturally
7. **Bank lending works** — banks create money via loans; repayments destroy it
8. **Bond auctions work** — government issues bonds; banks/households bid; CB backstops
9. **Inflation responds correctly** — driven by unit labor costs and demand pressure, not raw spending
10. **Charts update live** — all key indicators chart in real-time
11. **Scenario playable** — one scenario with win/lose conditions completable
12. **Sandbox works** — free experimentation mode with full dashboard
13. **Pause and speed controls work** — player can control simulation flow

## Development Phases

### Phase 1: Core Simulation Engine
- SFC accounting system (double-entry bookkeeping)
- Two money circuits (reserves + deposits)
- Balance sheets for all sectors
- Monthly tick processing loop

### Phase 2: Economic Agents
- Government spending and taxation
- Firm production (3 sectors, profit-driven)
- Household consumption (hierarchical needs, 3 classes)
- Labor market (wage posting)
- Bank lending (endogenous money creation)
- Government bond auctions

### Phase 3: Pricing and Dynamics
- Cost-plus markup pricing with unit labor costs
- Demand adjustment and three inflation buffers
- Investment (public infrastructure + private capital)
- Time lags for policy effects

### Phase 4: Godot Integration
- Godot project setup with C#
- Time system (tick-based with real-time flow, pause, speed)
- Connect simulation engine to Godot scene tree

### Phase 5: UI
- Policy panel (sliders for spending, allocation, tax)
- Live charts for all key indicators
- Balance sheet view
- Resource utilization display
- Policy pipeline visualization
- Basic map view (cosmetic)
- Time controls

### Phase 6: Gameplay
- Sandbox mode with dashboard
- One scenario with objectives and fail conditions
- Win/lose detection and feedback

### Phase 7: Polish
- Balancing the economic model
- UI/UX improvements
- Playtesting and iteration
