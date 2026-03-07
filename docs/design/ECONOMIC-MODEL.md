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

The model explicitly tracks two layers of money, reflecting how modern monetary systems actually operate. **This two-layer structure is fundamental to the entire economic model.** All money flows in the simulation must respect which layer they belong to:

- **Reserve layer (central bank money):** Used for settlement between institutions — government↔bank transactions, bank↔bank transactions. Households and firms never touch this layer.
- **Deposit layer (bank money):** Used for all private sector activity — wages, purchases, savings. This is the only money households and firms see.

Any transaction that crosses a bank boundary (e.g., a household at Bank A paying a firm at Bank B) triggers movements in **both** layers simultaneously: deposits move in the lower layer, reserves move in the upper layer. Transactions within the same bank only move deposits.

### Central Bank Money (Reserves)

```
+--------------------------------------------------------------+
|                    CENTRAL BANK LEDGER                        |
|                                                               |
|   Treasury Account          Bank Reserve Accounts             |
|   +--------------+         +--------------+                   |
|   | Government   |         | Bank A       |                   |
|   | balance      |         | reserves     |                   |
|   +--------------+         +--------------+                   |
|                             | Bank B       |                   |
|                             | reserves     |                   |
|                             +--------------+                   |
+--------------------------------------------------------------+
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

**Interbank payment flow (e.g., household at Bank A pays firm at Bank B):**
1. Payer's deposit account debited at Bank A
2. Bank A's reserve account at central bank debited
3. Bank B's reserve account at central bank credited
4. Payee's deposit account credited at Bank B

**Intra-bank payment flow (payer and payee at same bank):**
1. Payer's deposit account debited
2. Payee's deposit account credited
3. No reserves move — internal ledger adjustment only

### Deposit Money (Bank Deposits)

```
+--------------------------------------------------------------+
|                  COMMERCIAL BANK LEDGERS                      |
|                                                               |
|   Household Accounts         Firm Accounts                    |
|   +--------------+         +--------------+                   |
|   | Low income   |         | Agriculture  |                   |
|   +--------------+         +--------------+                   |
|   | Mid income   |         | Manufacturing|                   |
|   +--------------+         +--------------+                   |
|   | High income  |         | Construction |                   |
|   +--------------+         +--------------+                   |
|                             | Services     |                   |
|                             +--------------+                   |
|                                                               |
|   This is the only money households and firms see.            |
+--------------------------------------------------------------+
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

> **Critical design rule:** The Treasury account balance never constrains government spending. The government, as currency issuer, can always spend by crediting bank reserve accounts. The Treasury balance is tracked for accounting purposes only — it is not a budget constraint. The real constraint on government spending is inflation (available real resources), not money.
>
> **Political constraints** (debt ceilings, balanced budget rules) are self-imposed policy choices, not operational limits. These are modeled in the policy constraint layer (post-MVP), not in the engine. See GAME-DESIGN.md Section 5.

**Actions:**
- Spend money (creates currency) allocated to: infrastructure, public services, direct transfers
- Collect taxes (destroys currency)
- Issue bonds (auction-based, see below)
- Set tax rate
- Set spending level and allocation

**Government as employer and procurer:**

The government participates in the real economy through two channels:

1. **Direct employment** — The government hires workers from the labor pool, competing with private firms. Public teachers, healthcare workers, engineers, administrators, and planners are government employees. Government wage rates are set by a data-driven pay scale that adjusts more slowly than private sector wages (civil service stickiness). Public sector employment is tracked separately from private employment.

2. **Procurement** — The government purchases goods and services from private sectors. Infrastructure spending creates demand for construction output and manufacturing materials. Public services may procure equipment and outsource certain functions.

**Spending allocation maps to real resources as follows:**

| Spending category | Public employment | Sector procurement | Direct resource competition |
|---|---|---|---|
| **Infrastructure** | Engineers, planners, project managers | Construction output, manufacturing materials | Yes — competes for construction workers and sector capacity |
| **Public services** | Teachers, doctors, administrators | Equipment (manufacturing), outsourced services | Yes — competes for skilled service and manufacturing workers |
| **Direct transfers** | None | None | No — money flows to households who spend via AIDS demand system |

This distinction is central to the MMT argument. Transfers increase household income and create demand distributed across sectors via the AIDS consumption model. Infrastructure and public services directly absorb real resources — labor and sector output. The same dollar amount produces very different economic effects depending on its composition.

The government's wage offers create a de facto wage floor. If the government pays 3,000/month for public service workers, private firms offering significantly less will lose workers. This demonstrates how fiscal policy sets an implicit minimum wage — a key MMT/post-Keynesian insight and the precursor to the Job Guarantee's role as a wage anchor (post-MVP).

### Central Bank

Operates the reserve system and sets monetary policy.

**Actions:**
- Maintain reserve accounts for commercial banks
- Set the policy interest rate (base rate)
- Provide reserves to commercial banks on demand at the policy rate (lender of last resort / standing facility). Post-MVP: standing facility rate may differ from policy rate (corridor/floor system — see F10)
- Buy/sell government bonds on secondary market (buyer of last resort)
- In MVP: policy rate can default to 0

Reserve provision ensures that bank lending is never constrained by reserve availability. Banks lend based on creditworthiness of borrowers and profitability, then obtain any needed reserves from the central bank (see [Bank Credit Creation](#bank-credit-creation-endogenous-money) above). In the MVP with a single aggregate bank, reserve provision is simplified (the single bank's reserves are the net result of government spending minus taxation, plus any central bank operations).

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

Banking is modeled as financial intermediation, not as a production sector. Banks do not produce output that households consume. Their function is to create purchasing power (credit money) that enables real economic activity. This is consistent with the MMT/post-Keynesian SFC tradition (Godley & Lavoie) where the banking sector has a balance sheet and behavioral rules, not a production function. Bank revenue (interest spread) appears on balance sheets through interest flows, not through the AIDS demand system.

### Households (3 Classes in MVP)

| Class | Characteristics |
|---|---|
| **Low income** | High propensity to consume. Budget share heavily weighted toward agriculture and basic manufacturing. Minimal savings. May take on debt for necessities. |
| **Middle income** | Moderate consumption and saving. More balanced budget shares across sectors. Take on debt for housing (mortgages). Some discretionary spending on services. |
| **High income** | Low propensity to consume (relative to income). Budget share shifts toward services and discretionary manufacturing. Significant savings. |

**Balance sheet tracks:**
- Deposit account at bank (asset)
- Debts / loans (liabilities)
- Net wealth

**Behavior:**
- Supply labor to firms and government (see Labor Market)
- Consume according to the AIDS demand system — budget shares across sectors determined by income and prices (see Consumption Model)
- Save surplus income
- Borrow from banks when needed/eligible
- Pay taxes on income

### Firms (4 Sectors)

All firms are profit-driven: they estimate demand, consider costs, and make production and hiring decisions to maximize profit.

| Sector | Inputs | Outputs | Characteristics |
|---|---|---|---|
| **Agriculture & Primary** | Labor, land | Food, raw materials | Land-constrained, seasonal, low labor elasticity. Small GDP share but essential. |
| **Manufacturing** | Labor, raw materials, capital goods | Manufactured goods, capital goods | Capital-intensive. Produces both consumer goods and investment goods for all sectors. |
| **Construction** | Labor, manufactured materials, capital goods | Built structures, infrastructure | Highly cyclical, labor-intensive. Key channel for government infrastructure investment. |
| **Services** | Labor, capital goods | Services (retail, transport, healthcare, education, hospitality, professional) | Labor-intensive, largest employment share. Heterogeneous but shares key input characteristics. |

**Post-MVP sector expansion:** Sectors expand via a sub-sector hierarchy. Each top-level sector may contain sub-sectors with independent production functions and AIDS demand parameters. For example, Services may split into Healthcare, Education, Hospitality, and Professional Services. Sub-sectors are identified by a `parentId` field in `sectors.json`. The AIDS demand system extends naturally — a 4-sector parameter matrix becomes an 8-sector or 12-sector matrix.

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
- Invest in capital via bank loans or internal funds
- Hold inventory of unsold goods

## Production Function: Leontief Input-Output

Each sector uses **fixed technical coefficients** (Leontief / fixed proportions). To produce one unit of output, a firm needs fixed quantities of each input — inputs cannot substitute for one another.

```
Output_s = min(Input_1 / a_s1, Input_2 / a_s2, ..., Labor / a_sL)

Where:
  a_si = technical coefficient for input i in sector s
  a_sL = labor coefficient for sector s
```

For example, if Manufacturing has coefficients `labor: 0.4, raw_materials: 0.35, capital: 0.25`, then producing 100 units requires exactly 40 units of labor, 35 units of raw materials, and 25 units of capital goods. A shortage of any input constrains output proportionally.

### Inter-Sector Input-Output Matrix

Sectors consume each other's output as intermediate inputs. These linkages are defined by an input-output matrix specifying how much of each sector's output is required per unit of production in every other sector:

| Producing ↓ / Input → | Agriculture | Manufacturing | Construction | Services | Labor |
|---|---|---|---|---|---|
| **Agriculture** | — | low | — | low | high |
| **Manufacturing** | moderate | low | — | low | moderate |
| **Construction** | — | high | — | low | high |
| **Services** | low | low | — | — | high |

The exact coefficients are data-driven, loaded from `sectors.json`. The matrix above shows qualitative magnitudes for the MVP.

**Key properties:**
- Supply shocks propagate through the matrix: a disruption in Agriculture constrains Manufacturing (raw materials), which constrains Construction (manufactured materials)
- The matrix makes inter-sector dependencies explicit and produces emergent supply chain dynamics
- Government procurement enters through this matrix — infrastructure spending creates demand in Construction, which pulls from Manufacturing
- Coefficients are fixed in the short run; long-run technological change (coefficient drift) is a post-MVP enhancement

This follows the Godley/Lavoie (2012) SFC modeling tradition and Sraffa's (1960) *Production of Commodities by Means of Commodities*. Input coefficients in `sectors.json` are Leontief technical coefficients, not Cobb-Douglas exponents.

> **Design note — why Leontief, not Cobb-Douglas or CES:**
> Cobb-Douglas assumes smooth input substitution (σ=1), which implies wages equal marginal product — a neoclassical result incompatible with MMT/post-Keynesian wage theory. Empirical estimates of the elasticity of substitution consistently fall in the 0.4–0.7 range (Chirinko 2008, Oberfield & Raval 2021), much closer to Leontief (σ=0) than Cobb-Douglas. CES with low σ would be more empirically precise but adds calibration complexity without meaningful gameplay difference. Pure Leontief is the simplest model consistent with both the empirical evidence and the post-Keynesian theoretical framework.

## Pricing Model

### Cost-Plus Markup with Demand Adjustment

Prices are based on **unit costs**, not raw input prices. This is critical: wage increases alone do not cause inflation if productivity keeps pace.

```
Price = (Unit labor cost + Unit material cost) x (1 + markup)

Where:
  Unit labor cost = Total wages paid / Total output produced
  Unit material cost = Total material costs / Total output produced
  Markup = Base markup x Demand adjustment factor
```

**Demand adjustment factor:**
- When demand < capacity (slack): markup tends toward base or below (competitive pressure)
- When demand ~ capacity: markup at base level
- When demand > capacity (overheating): markup increases (sellers have pricing power)

**Markup adjustment asymmetry:**
Markups adjust at different speeds in each direction:
- Upward: fast — firms exploit pricing power quickly (e.g., speed factor 0.5)
- Downward: slow — firms resist margin compression (e.g., speed factor 0.1)

This mirrors the existing downward wage rigidity. Per-sector parameters (`markupUpwardSpeed`, `markupDownwardSpeed`) loaded from sectors.json.

### Supply-Side Markup Pressure (Seller's Inflation)

When input availability drops (e.g., material shortage from another sector), firms with constrained supply can raise markups independently of demand:

```
Supply pressure factor = Normal input availability / Actual input availability
```

When supply pressure > 1, it multiplies the markup alongside the demand factor:

```
Markup = Base markup × Demand adjustment × Supply pressure factor
```

This models seller's inflation (Weber/Wasner 2023): firms use supply scarcity as cover to widen margins even without excess demand. The supply pressure factor uses the same asymmetric adjustment speeds — it ratchets up fast but decays slowly as supply normalizes.

### Three Buffers Against Inflation

Higher wages do NOT automatically cause inflation. Three mechanisms absorb wage increases before prices must rise:

1. **Productivity gains** — if output per worker rises with wages, unit labor costs are unchanged
2. **Demand slack** — if there is unsold inventory or idle capacity, higher wages increase sales volume without requiring more production (e.g., less food waste, fuller capacity utilization)
3. **Profit margin compression** — firms may accept lower markups under competitive pressure rather than raise prices

**Inflation occurs when:** all three buffers are exhausted — productivity can't keep up, there is no slack, and margins are already compressed. Then unit costs rise and firms pass them through as higher prices.

### Unit Labor Costs (Lohnstuckkosten)

The model tracks unit labor costs per sector, not raw wages:

```
Unit labor cost = Wage rate / Labor productivity
Labor productivity = Output / Workers employed
```

- If wages rise 3% and productivity rises 3% -> ULC unchanged -> no price pressure
- If wages rise 5% and productivity rises 2% -> ULC rises 3% -> cost-push price pressure
- If wages rise 2% and productivity rises 4% -> ULC falls 2% -> prices can fall or margins improve

## Consumption Model: Almost Ideal Demand System (AIDS)

Household consumption is modeled using the Almost Ideal Demand System (Deaton & Muellbauer, 1980). AIDS determines how each household class allocates its budget across production sectors based on current prices and real income.

### The AIDS Formula

For household class _c_ and sector _i_:

```
w_ci = alpha_ci + SUM_j(gamma_cij * ln(p_j)) + beta_ci * ln(M_c / P)
```

Where:
- `w_ci` is the budget share household class _c_ spends on sector _i_ output
- `alpha_ci` is the intercept — the baseline budget share at reference prices and income
- `gamma_cij` captures how the budget share for sector _i_ responds to a change in sector _j_'s price (own-price and cross-price effects)
- `beta_ci` captures how the budget share changes as real income changes (income elasticity)
- `M_c` is disposable income for household class _c_ (post-tax income + available credit - debt service)
- `P` is a price index (Stone price index: `ln(P) = SUM_i(w_ci * ln(p_i))`)
- `p_j` is the price of sector _j_'s output

### Parameter Constraints

The AIDS model satisfies three theoretical constraints that ensure economic consistency:

1. **Adding-up:** Budget shares sum to 1 for each class. `SUM_i(alpha_ci) = 1`, `SUM_i(gamma_cij) = 0`, `SUM_i(beta_ci) = 0`.
2. **Homogeneity:** Demand is homogeneous of degree zero in prices and income (doubling all prices and income leaves budget shares unchanged). `SUM_j(gamma_cij) = 0` for each _i_.
3. **Symmetry:** Cross-price effects are symmetric. `gamma_cij = gamma_cji`.

These constraints reduce the effective number of free parameters and ensure the model produces economically meaningful results.

### Per-Class Parameterization

Each household class has its own set of alpha, beta, and gamma parameters. This reflects the empirical finding that preferences differ structurally across income groups — rich households don't just spend more, they spend _differently_ even at the same price levels.

**Key behavioral properties that emerge from the parameters:**

- **Engel's law:** Low-income households spend a larger budget share on agriculture (food). As income rises, the agriculture share declines (`beta_agriculture < 0`). This is one of the most robust findings in economics.
- **Income elasticity:** Services budget share increases with income (`beta_services > 0`). Manufacturing and construction shares are relatively stable.
- **Price elasticity:** When a sector's price rises, its budget share adjusts based on gamma parameters. Agriculture demand is price-inelastic (people buy food regardless of price). Services demand is more elastic.
- **Cross-price substitution:** When manufacturing prices rise, some demand shifts to other sectors based on gamma cross-terms.

### Empirical Foundation

AIDS parameters are among the most empirically estimated in all of economics. The MVP uses parameters sourced from published estimates based on US household expenditure data (Consumer Expenditure Survey, Bureau of Labor Statistics). Key references include AIDS estimates by income group using CEX data and USDA Economic Research Service food demand elasticity estimates.

Post-MVP, different country parameter sets (Germany, Brazil, Nigeria, etc.) can be loaded as scenario data files. Different countries have structurally different consumption patterns — a US economy is services-dominated (~77% of GDP) while a German economy has a significantly larger manufacturing share (~20% of GDP). These differences are captured entirely by different alpha, beta, gamma parameters.

### Edge Cases

- **Near-zero income:** The `ln(M/P)` term diverges as income approaches zero. A small epsilon floor is applied to real income before the logarithm to prevent undefined behavior. This is a numerical safeguard, not an economic subsistence floor — the welfare state (government transfers) is the policy mechanism that prevents destitution, and its absence should be visible as a crisis.
- **Extreme prices:** Budget shares are clamped to `[0, 1]` after computation and renormalized to sum to 1. This prevents the model from producing economically meaningless negative demand at extreme price levels.

### From Budget Shares to Nominal Demand

Each tick, the consumption engine:

1. Computes budget shares `w_ci` for each household class and sector using current prices and income
2. Converts to nominal demand: `demand_ci = w_ci * M_c`
3. Passes the demand vector to the goods market for fulfillment against sector inventory/capacity
4. If demand exceeds supply, rationing occurs — unmet demand feeds into the next tick's price adjustments via the pricing engine

## Labor Market: Wage Posting

Firms and the government post job openings with an offered wage. Households decide whether to accept based on their circumstances.

### Employer Side (Posting)

**Firms:**
- Firms determine labor demand based on their production targets
- Each firm posts a wage for open positions
- Wage influenced by: sector conditions, competition for workers, profitability
- Wages tend to rise when: labor is scarce, the sector is profitable, firms compete for workers
- Wages tend to fall when: labor is abundant, profits are squeezed (but sticky downward — wage cuts are slow)

**Government:**
- Government determines labor demand based on spending allocation (infrastructure, public services)
- Government posts wages based on a data-driven pay scale
- Government wages adjust more slowly than private wages (civil service stickiness)
- Government wage rates are not directly profit-driven — they follow policy-set pay scales
- Government job postings enter the same labor market pool as firm postings

### Household Side (Accepting)
- Each household class has a **reservation wage** (minimum acceptable wage)
- Low income: lower reservation wage (desperate for work)
- High income: higher reservation wage (can afford to be selective)
- Households accept jobs that meet or exceed their reservation wage
- Preference for higher-paying jobs; workers choose between public and private employment based on offered wage
- Workers move to better-paying sectors/employers over time

### Unemployment
- Occurs when: employers (firms + government) don't post enough jobs, or posted wages are below reservation wages
- Unemployment is involuntary when workers want to work but no suitable jobs exist
- Unemployed households have zero wage income; rely on savings, transfers, or debt

### Wage Dynamics
- Tight labor market (low unemployment) -> wages rise as employers compete for workers
- Loose labor market (high unemployment) -> wages stagnate or fall slowly (downward rigidity)
- Sector-specific: agriculture wages may differ from services wages
- Government wages anchor the lower end of the distribution — private firms cannot push wages far below public pay without losing workers

### Employment Tracking
- Private sector employment tracked per sector (agriculture, manufacturing, construction, services)
- Public sector employment tracked per spending function (infrastructure, public services)
- Total employment = private + public
- Employment indicators distinguish public and private shares

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
2. **Existing debt** — debt-to-income ratio. Higher existing debt -> higher risk -> higher premium or rejection.
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

The government issues bonds following real-world institutional practice. Bond issuance is a reserve management operation, not a means of "financing" the deficit (see MMT Context below).

### MMT Context
In MMT, bond issuance is a reserve-draining operation, not "financing" for government spending. The government spends first (creating reserves) and then issues bonds (swapping reserves for bonds). Bonds are a monetary policy tool, not a fiscal necessity.

The game models the real-world procedure (auctions) while allowing players to observe that the central bank can always ensure bonds are sold (making the interest rate a policy choice).

### Auction Mechanics
1. Government announces bond issuance (amount and maturity)
2. Commercial banks submit bids (price/yield)
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
- Interest payments flow to bond holders (banks in MVP; post-MVP secondary market enables household bond holding)
- This creates a distributional dynamic: deficit -> bonds -> interest -> flows to bond holders

**Note on the fiscal channel of interest rate policy:** Higher interest rates increase government bond interest payments, which are themselves government spending (currency creation). This means higher rates have a stimulative fiscal effect — more income flows to bond holders — that partially offsets the contractionary effect on borrowing. This is a key reason MMT prefers fiscal policy (spending and taxation) over monetary policy (interest rates) for demand management. See Mosler (1993, 1995), Fullwiler (2006), Wray (2015) Ch. 5.

### No Sovereign Default
The government, as currency issuer, always pays bond interest and principal when due. Bond payments are government spending (currency creation). Sovereign default on domestic-currency-denominated debt is a political choice, never an operational necessity. The game does not model voluntary default.

## Investment

### Public Investment (Government)
Government spending on **infrastructure** builds public capital that benefits all sectors:
- Improves transportation -> reduces costs for all firms
- Improves utilities -> increases productive capacity
- Improves education/health (public services) -> increases labor productivity over time

Public capital depreciates slowly and requires maintenance spending.

### Private Investment (Firms)
Firms invest in capital goods to maintain and expand productive capacity:

**Investment decision factors:**
- Expected future demand (based on recent sales trends)
- Current capacity utilization (if near full, need to expand)
- Borrowing costs (bank lending rate)
- Available capital goods on the market (produced by manufacturing sector)

**Funding sources:**
1. Bank loans (credit creation — new money is created to fund the investment)
2. Internal funds (retained profits redirected to investment)

Bank lending is the primary enabler of investment. A firm's decision to invest is based on expected demand, capacity utilization, and borrowing costs — not on the pre-existence of savings. When investment is bank-funded, the loan creates new deposits (which become someone's saving), so at the macro level investment creates saving, not the reverse.

**Capital dynamics:**
- Capital goods depreciate over time (require replacement investment)
- New investment expands productive capacity
- Manufacturing sector produces capital goods for all sectors (including itself)

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
- Color-coded indicators for "policy enacted" -> "in pipeline" -> "taking effect"

## Simulation Tick Sequence

Each monthly tick processes in this order:

```
1. GOVERNMENT PHASE
   a. Execute spending: pay for infrastructure, public services, transfers
      (new reserves created, flow to bank deposits)
   b. Pay interest on outstanding bonds
      (new reserves created, flow to bank deposits)
   c. Compute government resource demand from spending allocation:
      - Register public sector job postings with labor market
        (infrastructure workers, public service employees)
      - Register procurement demand with sectors
        (construction output, manufacturing materials)
      - Direct transfers: no resource demand, income flows to households
   d. Collect taxes from households and firms
      (deposits destroyed, reserves returned to treasury)
   e. Issue new bonds if needed (auction)
      (drains excess reserves from banking system)

2. PRODUCTION PHASE
   a. Firms assess demand (based on previous period sales + expectations)
   b. Firms set production targets
   c. Firms post job openings with wages
   d. Government posts job openings with public pay scale
   e. Households evaluate ALL job postings (firm + government),
      accept/reject based on reservation wage, prefer higher pay
   f. Production occurs using available labor + materials + capital
   g. Goods enter inventory

3. MARKET PHASE
   a. Firms set prices (cost-plus markup with demand adjustment)
   b. ConsumptionEngine computes AIDS budget shares per household class
      using current prices and disposable income
   c. Nominal demand per sector = budget shares x disposable income
   d. Government procurement demand added to sector demand
   e. Firms sell from inventory to meet combined demand
   f. Unsold goods remain in inventory (potential waste/depreciation)

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
| **Public sector employment share** | Public sector workers / Total employed workers |
| **Inflation rate** | % change in average price level from previous period |
| **GDP (Output)** | Total value of goods and services produced |
| **Government balance** | Spending - Tax revenue (positive = net currency creation, negative = net currency withdrawal) |
| **Private sector balance** | Total private income - Total private spending - Taxes |
| **Private sector net financial assets** | Total government bonds held by private sector + net deposits (mirror image of government bonds outstanding) |
| **Government bonds outstanding / GDP** | Total government bonds outstanding / GDP |
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
| Sector granularity | 4 top-level sectors | Sub-sector hierarchy (8-15 sectors) via parentId |
| Foreign sector | None (closed economy) | Multiple AI nations with trade |
| Bank competition | Single aggregate bank | Multiple competing banks |
| Bond secondary market | No resale | Full secondary market |
| Central bank policy rate | Fixed at 0 | Player-adjustable or rule-based |
| Provinces | Single province | Multiple geographic provinces |
| Firm heterogeneity | One representative firm per sector | Multiple competing firms per sector |
| Wage negotiation | Simple posting/acceptance | Collective bargaining, contracts |
| Government spending types | 3 (infrastructure, services, transfers) | More granular spending categories |
| AIDS parameters | US-based estimates | Country-specific parameter sets |
| Currency demand | Implicit — single currency, no alternative exists (Chartalism not modeled) | Explicit tax-driven currency demand for exchange rate dynamics |
