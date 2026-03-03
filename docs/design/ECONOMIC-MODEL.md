# Economic Model

This document defines the simulation's economic model in detail. The model is grounded in Modern Monetary Theory (MMT) and post-Keynesian economics, using a Stock-Flow Consistent (SFC) framework.

## Foundational Framework: Stock-Flow Consistency (SFC)

The simulation uses double-entry bookkeeping for the entire economy. Every financial transaction has an explicit source and destination. Money never appears from nowhere or vanishes — except through the two legitimate channels:

1. **Government spending** — creates new currency (central bank reserves + bank deposits)
2. **Taxation** — destroys currency (removes deposits + reserves from circulation)
3. **Bank lending** — creates new deposit money (and a corresponding debt)
4. **Loan repayment** — destroys deposit money (and extinguishes corresponding debt)

At the end of every simulation tick, the accounting identity must hold:

```
Government balance + Private sector balance = 0
(MVP: closed economy, no foreign sector)

Full game:
Government balance + Private sector balance + Foreign sector balance = 0
```

This is not an assumption — it is enforced by the double-entry bookkeeping. Every dollar of government deficit is mechanically a dollar of private sector surplus.

## Two Money Circuits

The model explicitly tracks two layers of money, reflecting how modern monetary systems actually operate.

### Central Bank Money (Reserves)

```
┌──────────────────────────────────────────────────────────────┐
│                    CENTRAL BANK LEDGER                        │
│                                                              │
│   Treasury Account          Bank Reserve Accounts            │
│   ┌─────────────┐          ┌─────────────┐                  │
│   │ Government   │          │ Bank A      │                  │
│   │ balance      │          │ reserves    │                  │
│   └─────────────┘          ├─────────────┤                  │
│                             │ Bank B      │                  │
│                             │ reserves    │                  │
│                             └─────────────┘                  │
└──────────────────────────────────────────────────────────────┘
```

- The **Treasury** (government) holds an account at the central bank.
- Each **commercial bank** holds a reserve account at the central bank.
- **Only** the central bank, treasury, and commercial banks operate in this layer.
- The private sector (households, firms) never sees or touches reserves.

**Government spending flow:**
1. Treasury account debited (reserves leave treasury)
2. Recipient's bank reserve account credited (reserves arrive at bank)
3. Bank credits recipient's deposit account (see deposit layer below)

**Taxation flow (reverse):**
1. Taxpayer's deposit account debited at their bank
2. Bank's reserve account at central bank debited
3. Treasury account credited (money effectively destroyed)

### Deposit Money (Bank Deposits)

```
┌──────────────────────────────────────────────────────────────┐
│                  COMMERCIAL BANK LEDGERS                      │
│                                                              │
│   Household Accounts         Firm Accounts                   │
│   ┌─────────────┐          ┌─────────────┐                  │
│   │ Low income   │          │ Agri firms  │                  │
│   ├─────────────┤          ├─────────────┤                  │
│   │ Mid income   │          │ Industry    │                  │
│   ├─────────────┤          ├─────────────┤                  │
│   │ High income  │          │ Services    │                  │
│   └─────────────┘          └─────────────┘                  │
│                                                              │
│   This is the only money households and firms see.           │
└──────────────────────────────────────────────────────────────┘
```

- Households and firms hold deposit accounts at commercial banks.
- All private sector transactions occur in this layer.
- Wages, purchases, savings — all are movements of deposit money between accounts.

### Bank Credit Creation (Endogenous Money)

Banks create new deposit money when they issue loans. This is NOT lending out existing deposits — it is the creation of new money:

```
Bank makes a $50,000 mortgage:
  Bank's balance sheet:
    Assets:  +$50,000 (loan receivable)
    Liabilities: +$50,000 (new deposit in borrower's account)

  Borrower's balance sheet:
    Assets:  +$50,000 (new deposit)
    Liabilities: +$50,000 (mortgage debt)
```

When the loan is repaid, the process reverses — deposit money is destroyed.

**Note:** Bank lending does NOT require prior reserves. Banks lend first and seek reserves after (or the central bank provides them). This is a key MMT/post-Keynesian insight about endogenous money.

## Sectors and Agents

### Government

The currency-issuing sovereign. The player controls this sector.

**Balance sheet tracks:**
- Treasury account at central bank (reserves)
- Outstanding bonds (liabilities)
- Accumulated deficit/surplus

**Actions:**
- Spend money (creates currency) allocated to: infrastructure, public services, direct transfers
- Collect taxes (destroys currency)
- Issue bonds (auction-based, see below)
- Set tax rate
- Set spending level and allocation

### Central Bank

Operates the reserve system and sets monetary policy.

**Actions:**
- Maintain reserve accounts for commercial banks
- Set the policy interest rate (base rate)
- Buy/sell government bonds on secondary market (buyer of last resort)
- In MVP: policy rate can default to 0

### Commercial Banks

Intermediate between the reserve and deposit circuits. Create credit.

**Balance sheet tracks:**
- Reserve account at central bank (asset)
- Loans outstanding (assets)
- Customer deposits (liabilities)
- Government bonds held (assets)
- Equity / retained profits

**Behavior:**
- Accept deposits from households and firms
- Create loans based on creditworthiness assessment (see Lending section)
- Buy government bonds at auction
- Earn interest on loans and bonds
- Pay interest on deposits (if modeled)
- Lending rate = central bank policy rate + risk-based spread

### Households (3 Classes in MVP)

| Class | Characteristics |
|---|---|
| **Low income** | High propensity to consume. Spend most income on survival needs (food, housing). Minimal savings. May take on debt for necessities. |
| **Middle income** | Moderate consumption and saving. Can afford comfort goods. Take on debt for housing (mortgages). Some discretionary spending. |
| **High income** | Low propensity to consume (relative to income). Significant savings and investment. Spend on luxury goods/services. May buy government bonds. |

**Balance sheet tracks:**
- Deposit account at bank (asset)
- Debts / loans (liabilities)
- Net wealth

**Behavior:**
- Supply labor to firms (see Labor Market)
- Consume goods according to hierarchical needs (see Consumption)
- Save surplus income
- Borrow from banks when needed/eligible
- Pay taxes on income

### Firms (3 Sectors)

All firms are profit-driven: they estimate demand, consider costs, and make production and hiring decisions to maximize profit.

| Sector | Inputs | Outputs |
|---|---|---|
| **Agriculture** | Labor, land | Food (survival need), raw materials |
| **Industry** | Labor, raw materials, capital goods | Manufactured goods (comfort need), capital goods (investment) |
| **Services** | Labor, capital goods | Services (comfort/luxury need) |

**Balance sheet tracks:**
- Deposit account at bank (asset)
- Capital goods / equipment (assets)
- Loans (liabilities)
- Equity / retained profits
- Inventory (unsold goods)

**Behavior:**
- Estimate demand and set production targets
- Post wages and hire workers (see Labor Market)
- Purchase inputs from other sectors
- Produce goods/services
- Set prices (see Pricing)
- Pay wages and taxes
- Invest in capital from retained profits + bank loans
- Hold inventory of unsold goods

## Pricing Model

### Cost-Plus Markup with Demand Adjustment

Prices are based on **unit costs**, not raw input prices. This is critical: wage increases alone do not cause inflation if productivity keeps pace.

```
Price = (Unit labor cost + Unit material cost) × (1 + markup)

Where:
  Unit labor cost = Total wages paid / Total output produced
  Unit material cost = Total material costs / Total output produced
  Markup = Base markup × Demand adjustment factor
```

**Demand adjustment factor:**
- When demand < capacity (slack): markup tends toward base or below (competitive pressure)
- When demand ≈ capacity: markup at base level
- When demand > capacity (overheating): markup increases (sellers have pricing power)

### Three Buffers Against Inflation

Higher wages do NOT automatically cause inflation. Three mechanisms absorb wage increases before prices must rise:

1. **Productivity gains** — if output per worker rises with wages, unit labor costs are unchanged
2. **Demand slack** — if there is unsold inventory or idle capacity, higher wages increase sales volume without requiring more production (e.g., less food waste, fuller capacity utilization)
3. **Profit margin compression** — firms may accept lower markups under competitive pressure rather than raise prices

**Inflation occurs when:** all three buffers are exhausted — productivity can't keep up, there is no slack, and margins are already compressed. Then unit costs rise and firms pass them through as higher prices.

### Unit Labor Costs (Lohnstückkosten)

The model tracks unit labor costs per sector, not raw wages:

```
Unit labor cost = Wage rate / Labor productivity
Labor productivity = Output / Workers employed
```

- If wages rise 3% and productivity rises 3% → ULC unchanged → no price pressure
- If wages rise 5% and productivity rises 2% → ULC rises 3% → cost-push price pressure
- If wages rise 2% and productivity rises 4% → ULC falls 2% → prices can fall or margins improve

## Consumption Model: Hierarchical Needs

Households fulfill needs in priority order, inspired by Maslow's hierarchy. Lower needs must be met before spending on higher needs.

```
Priority 1: SURVIVAL (food, water, basic clothing)
    → Purchased from: Agriculture sector
    → All classes must fulfill this first
    → Demand is highly inelastic (people buy food regardless of price)

Priority 2: SHELTER (housing, utilities, basic infrastructure)
    → Purchased from: Services + Industry sector
    → Low income may struggle here; may require debt
    → Somewhat inelastic

Priority 3: COMFORT (manufactured goods, transportation, education, healthcare)
    → Purchased from: Industry + Services sector
    → Middle and high income reach this level
    → Moderately elastic (responsive to price)

Priority 4: LUXURY (premium services, leisure, financial assets)
    → Purchased from: Services sector
    → Primarily high income
    → Highly elastic
```

**Spending logic per tick:**
1. Calculate total disposable income (post-tax income + available credit - debt service)
2. Allocate to survival needs at current market prices
3. If income remains, allocate to shelter
4. If income remains, allocate to comfort
5. If income remains, allocate to luxury
6. Remaining income → savings (deposits at bank)

**Class differences:**
- **Low income:** Most/all income goes to survival and shelter. Little discretionary. May need debt for shelter.
- **Middle income:** Fulfills survival and shelter comfortably. Some comfort spending. May take mortgage for housing.
- **High income:** Fulfills all levels easily. Significant surplus goes to savings, bonds, or investment.

## Labor Market: Wage Posting

Firms post job openings with an offered wage. Households decide whether to accept based on their circumstances.

### Firm Side (Posting)
- Firms determine labor demand based on their production targets
- Each firm posts a wage for open positions
- Wage influenced by: sector conditions, competition for workers, profitability
- Wages tend to rise when: labor is scarce, the sector is profitable, firms compete for workers
- Wages tend to fall when: labor is abundant, profits are squeezed (but sticky downward — wage cuts are slow)

### Household Side (Accepting)
- Each household class has a **reservation wage** (minimum acceptable wage)
- Low income: lower reservation wage (desperate for work)
- High income: higher reservation wage (can afford to be selective)
- Households accept jobs that meet or exceed their reservation wage
- Preference for higher-paying jobs; workers move to better-paying sectors over time

### Unemployment
- Occurs when: firms don't post enough jobs, or posted wages are below reservation wages
- Unemployment is involuntary when workers want to work but no suitable jobs exist
- Unemployed households have zero wage income; rely on savings, transfers, or debt

### Wage Dynamics
- Tight labor market (low unemployment) → wages rise as firms compete for workers
- Loose labor market (high unemployment) → wages stagnate or fall slowly (downward rigidity)
- Sector-specific: agriculture wages may differ from services wages

## Bank Lending: Creditworthiness-Based

Banks create money through lending. Lending decisions are based on borrower creditworthiness with interest rates tied to the central bank policy rate.

### Lending Rate
```
Lending rate = Central bank policy rate + Bank spread + Risk premium

Where:
  Central bank policy rate: set by central bank (0 in MVP default)
  Bank spread: bank's base profit margin on lending
  Risk premium: varies by borrower creditworthiness
```

### Creditworthiness Assessment
Banks evaluate potential borrowers on:
1. **Income** — is the borrower's income sufficient to service the debt?
2. **Existing debt** — debt-to-income ratio. Higher existing debt → higher risk → higher premium or rejection.
3. **Collateral** — for secured loans (mortgages), the value of the asset backing the loan.

### Loan Types (MVP)
- **Consumer loans** — short-term, unsecured. For households needing to cover expenses.
- **Mortgages** — long-term, secured by housing. For household shelter needs.
- **Business loans** — for firms investing in capital goods or covering operating costs.

### Credit Creation Mechanics
When a loan is approved:
1. Bank creates new deposit in borrower's account (money created)
2. Bank records loan as an asset (receivable)
3. Borrower records loan as a liability (debt)
4. **No reserves are needed first** — banks lend first, seek reserves after

When a loan is repaid:
1. Borrower's deposit debited (money destroyed)
2. Bank's loan asset reduced
3. Interest payments are the bank's revenue (not destroyed — they stay as bank deposits)

### Defaults
- If a borrower cannot service their debt, they default
- Bank writes off the loan (loss)
- If bank losses exceed equity, bank becomes insolvent (systemic risk — post-MVP)

## Government Bonds: Auction-Based

The government issues bonds to manage the deficit, following real-world institutional practice.

### MMT Context
In MMT, bond issuance is a reserve-draining operation, not "financing" for government spending. The government spends first (creating reserves) and then issues bonds (swapping reserves for bonds). Bonds are a monetary policy tool, not a fiscal necessity.

The game models the real-world procedure (auctions) while allowing players to observe that the central bank can always ensure bonds are sold (making the interest rate a policy choice).

### Auction Mechanics
1. Government announces bond issuance (amount and maturity)
2. Commercial banks and high-income households submit bids (price/yield)
3. Bonds allocated to highest bidders (lowest yield demanded)
4. If demand is insufficient, interest rate rises to attract buyers
5. **Central bank as buyer of last resort:** the central bank can buy any unsold bonds, effectively setting a ceiling on rates

### Bond Properties
- **Face value:** fixed denomination
- **Coupon rate:** determined at auction (interest paid to holder)
- **Maturity:** time until the government repays the face value
- **Secondary market:** bond holders can sell bonds to other agents (post-MVP simplification: may be omitted initially)

### Interest Payments
- Government pays interest on outstanding bonds each period
- This is government spending (creates new currency)
- Interest payments flow to bond holders (banks, wealthy households)
- This creates a distributional dynamic: deficit → bonds → interest → flows to wealth holders

## Investment

### Public Investment (Government)
Government spending on **infrastructure** builds public capital that benefits all sectors:
- Improves transportation → reduces costs for all firms
- Improves utilities → increases productive capacity
- Improves education/health (public services) → increases labor productivity over time

Public capital depreciates slowly and requires maintenance spending.

### Private Investment (Firms)
Firms invest in capital goods to maintain and expand productive capacity:

**Investment decision factors:**
- Expected future demand (based on recent sales trends)
- Current capacity utilization (if near full, need to expand)
- Profitability (can they afford it from retained profits?)
- Borrowing costs (bank lending rate)
- Available capital goods on the market (produced by industry sector)

**Funding sources:**
1. Retained profits (internally generated funds)
2. Bank loans (endogenous money creation)

**Capital dynamics:**
- Capital goods depreciate over time (require replacement investment)
- New investment expands productive capacity
- Industry sector produces capital goods for all sectors (including itself)

## Time and Lags

### Tick Rate
Each simulation tick represents **one month**. Game speed is adjustable (1x, 2x, 5x).

### Policy Lags
Policy changes do not take effect instantaneously. Short lags with clear visual feedback:

| Policy change | Lag | Rationale |
|---|---|---|
| Tax rate change | 1 tick (1 month) | Tax collection adjusts next period |
| Spending level change | 1-2 ticks | Budget allocation and disbursement takes time |
| Spending reallocation | 2-3 ticks | Redirecting spending to new targets takes longer |
| Infrastructure effects | 6-12 ticks | Building takes time; benefits are gradual |
| Public services effects | 12-24 ticks | Education/health improvements are slow |

### Economic Lags
Natural economic processes also have delays:

| Process | Lag | Rationale |
|---|---|---|
| Wage adjustment | 1-3 ticks | Wages are sticky, contracts renegotiate slowly |
| Price adjustment | 1-2 ticks | Firms adjust prices based on recent cost/demand data |
| Hiring/firing | 1-2 ticks | Recruitment and layoffs take time |
| Investment to capacity | 3-6 ticks | Building new capacity takes months |
| Household spending adjustment | 1 tick | Households adjust spending quickly to income changes |

### Visual Feedback
The UI shows pending changes in a pipeline:
- "Government spending increase: $X arriving in 2 months"
- "Infrastructure project: 40% complete, capacity boost in 4 months"
- Color-coded indicators for "policy enacted" → "in pipeline" → "taking effect"

## Simulation Tick Sequence

Each monthly tick processes in this order:

```
1. GOVERNMENT PHASE
   a. Collect taxes from households and firms
      (deposits destroyed, reserves returned to treasury)
   b. Execute spending: pay for infrastructure, public services, transfers
      (new reserves created, flow to bank deposits)
   c. Pay interest on outstanding bonds
   d. Issue new bonds if needed (auction)

2. PRODUCTION PHASE
   a. Firms assess demand (based on previous period sales + expectations)
   b. Firms set production targets
   c. Firms post job openings with wages
   d. Households accept/reject job offers
   e. Production occurs using available labor + materials + capital
   f. Goods enter inventory

3. MARKET PHASE
   a. Firms set prices (cost-plus markup with demand adjustment)
   b. Households purchase goods according to hierarchical needs
   c. Firms sell from inventory
   d. Unsold goods remain in inventory (potential waste/depreciation)

4. FINANCIAL PHASE
   a. Banks assess loan applications
   b. Approved loans create new deposits (endogenous money)
   c. Borrowers make debt service payments (deposits destroyed)
   d. Defaults processed if borrowers cannot pay
   e. Banks earn interest income, pay interest on deposits

5. ACCOUNTING PHASE
   a. All balance sheets updated
   b. SFC consistency check (all balances must sum to zero)
   c. Economic indicators calculated (employment, inflation, GDP, etc.)
   d. UI updated with new data points
```

## Key Economic Indicators

These are tracked and displayed to the player:

| Indicator | Definition |
|---|---|
| **Employment rate** | Workers employed / Total labor force |
| **Unemployment rate** | Workers seeking work but not employed / Total labor force |
| **Inflation rate** | % change in average price level from previous period |
| **GDP (Output)** | Total value of goods and services produced |
| **Government balance** | Spending - Tax revenue (negative = deficit) |
| **Private sector balance** | Total private income - Total private spending - Taxes |
| **Debt-to-GDP ratio** | Total government bonds outstanding / GDP |
| **Capacity utilization** | Actual output / Maximum possible output (per sector) |
| **Unit labor costs** | Wages / Productivity (per sector) |
| **Private debt level** | Total household + firm loans outstanding |
| **Bank reserves** | Total reserves in the banking system |
| **Savings rate** | Household savings / Household income |
| **Wage growth** | % change in average wages from previous period |
| **Interest rates** | CB policy rate, bank lending rates, bond yields |

## MVP Simplifications

The MVP implements the full model above with these simplifications:

| Aspect | MVP simplification | Full game target |
|---|---|---|
| Household classes | 3 fixed classes | 5+ classes or continuous spectrum |
| Foreign sector | None (closed economy) | Multiple AI nations with trade |
| Bank competition | Single aggregate bank | Multiple competing banks |
| Bond secondary market | No resale | Full secondary market |
| Central bank policy rate | Fixed at 0 | Player-adjustable or rule-based |
| Provinces | Single province | Multiple geographic provinces |
| Firm heterogeneity | One representative firm per sector | Multiple competing firms per sector |
| Wage negotiation | Simple posting/acceptance | Collective bargaining, contracts |
| Government spending types | 3 (infrastructure, services, transfers) | More granular spending categories |
