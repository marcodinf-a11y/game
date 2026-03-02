# MVP Scope

## Overview

The MVP is the smallest playable version that demonstrates MMT's core mechanics. It strips the full vision down to a single closed economy with population groups, two MMT concepts, and enough UI to create a meaningful gameplay loop.

## What's IN the MVP

### Economy

**Single closed economy** — one nation, one province, no foreign sector.

**Population groups** (not individual agents):

| Group | Role |
|---|---|
| Government | Currency issuer. Spends money into existence, collects taxes (destroys money). |
| Households | Earn wages from firms, consume goods/services, save surplus, pay taxes. |
| Firms (Agriculture) | Produce food and raw materials. Employ workers. Labor + land intensive. |
| Firms (Industry) | Produce manufactured and capital goods. Employ workers. Labor + materials + capital intensive. |
| Firms (Services) | Produce services. Employ workers. Labor + capital intensive. |
| Banks | Hold household and firm savings. Facilitate transactions between groups. |

**Resources:**
- Labor (supplied by households)
- Raw materials (produced by agriculture, consumed by industry)
- Capital goods (produced by industry, used by all firm types)
- Consumer goods and services (produced by all firm types, consumed by households)

### MMT Concepts

**1. Currency issuance & taxation**
- Government spending creates new currency in the economy
- Taxation removes currency from circulation
- The government balance shows cumulative deficit/surplus
- Players see that deficit = net money added to private sector

**2. Inflation as resource constraint**
- Each sector has a productive capacity based on available resources
- Spending into slack (unemployed workers, idle capacity) → increased output, no inflation
- Spending beyond capacity → rising prices (inflation)
- Resource utilization is visible per sector

### Player Controls

Three policy levers:

1. **Total government spending** — how much currency to spend per period
2. **Spending allocation** — where the money goes:
   - Infrastructure (increases productive capacity over time)
   - Public services (education, health — improves workforce quality)
   - Direct transfers (money directly to households)
3. **Tax rate** — single income tax rate applied to households and firms

### Gameplay

- **Real-time with pause** — time flows in monthly ticks, adjustable speed (1x, 2x, 5x)
- **Sandbox mode** — no win/lose, free experimentation. Dashboard shows all key metrics.
- **One scenario** — predefined objective, e.g.:
  - *"Grow employment to 95% while keeping inflation below 5% within 10 years"*
  - Fail conditions: hyperinflation (>25%) or economic collapse (employment <50%)

### UI

- **Policy panel** — sliders and inputs for spending level, allocation percentages, tax rate
- **Live charts** — real-time line charts for:
  - Employment rate
  - Inflation rate
  - GDP / total output
  - Government fiscal balance (deficit/surplus)
  - Private sector net savings
  - Output by sector (agriculture, industry, services)
- **Resource utilization bars** — how close each sector is to capacity
- **Simple map view** — single-province cosmetic map establishing the visual direction for later
- **Time controls** — play, pause, speed buttons

## What's NOT in the MVP

These are deferred to later versions:

| Feature | Reason for deferral |
|---|---|
| Job Guarantee | Adds complexity to labor market. Build basic employment dynamics first. |
| Sectoral balances | Requires foreign sector. Single closed economy can't show 3-sector identity. |
| Multiple economies / trade | Major complexity. Focus on domestic dynamics first. |
| Multiple provinces | Geographic complexity deferred. One province is enough to prove the economic model. |
| Individual agents | Population groups are sufficient for MVP. Agent-based simulation comes later. |
| Credit creation by banks | Banks are passive savings holders in MVP. Credit dynamics added later. |
| Interest rate policy | Fiscal policy only in MVP. Monetary policy added later. |
| Multiple tax types | Single income tax rate. Progressive taxes, sales tax, wealth tax come later. |

## Definition of Done

The MVP is complete when:

1. **Policy controls work** — player can adjust spending level, allocation, and tax rate via UI
2. **Money circuit flows** — government spending visibly creates currency; taxation visibly removes it
3. **Production runs** — firms produce goods using labor and resources; households consume
4. **Inflation responds to spending** — overspending relative to capacity causes visible price increases
5. **Unemployment responds to spending** — underspending causes visible unemployment and idle capacity
6. **Charts update live** — all key economic indicators chart in real-time as the simulation runs
7. **Scenario playable** — one scenario with clear win/lose conditions can be started and completed
8. **Sandbox works** — player can run the economy indefinitely in sandbox mode with the score dashboard
9. **Pause and speed controls work** — player can pause, resume, and adjust simulation speed

## Development Phases

A suggested order for building the MVP:

### Phase 1: Core Simulation
- Economic model with population groups
- Money flow: government → firms → households → government (via tax)
- Basic production and consumption
- Inflation and employment mechanics

### Phase 2: Godot Integration
- Godot project setup with C#
- Time system (tick-based with real-time flow)
- Pause and speed controls

### Phase 3: UI
- Policy panel (sliders for spending, allocation, tax)
- Live charts for key indicators
- Resource utilization display
- Basic map view (cosmetic)

### Phase 4: Gameplay
- Sandbox mode with dashboard
- One scenario with objectives and fail conditions
- Win/lose detection and feedback

### Phase 5: Polish
- Balancing the economic model
- UI/UX improvements
- Playtesting and iteration
