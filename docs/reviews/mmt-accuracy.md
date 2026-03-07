# MMT Accuracy Review

**Date:** 2026-03-02
**Documents reviewed:** PRD.md, ARCHITECTURE.md, IMPLEMENTATION-PLAN.md, ECONOMIC-MODEL.md, DESIGN.md, REVIEW-consistency.md
**References:** Wray (2015) *Modern Money Theory*, Mitchell/Wray/Watts (2019) *Macroeconomics*, Mosler (1995/2010), Godley/Lavoie (2012) *Monetary Economics*, Kelton (2020) *The Deficit Myth*, Tcherneva (2020) *The Case for a Job Guarantee*, Weber/Wasner (2023), Bank of England (2014)
**Purpose:** Verify that the game's economic model accurately represents MMT and post-Keynesian economics. Identify inaccuracies, missing mechanics, and framing issues.

## Design Decisions Informing This Review

- The government can always spend (currency issuer, no financial constraint) — confirmed by author
- The game prioritizes model accuracy; teaching is secondary (players learn by observing emergent behavior)
- Production function: Leontief (fixed proportions), consistent with Godley/Lavoie SFC tradition

## How to Read This Report

Each finding has a severity:

- **Critical** — Contradicts core MMT principles. The mechanism is wrong and would produce misleading behavior.
- **Medium** — Missing mechanic or framing issue that weakens MMT accuracy. Should be resolved before the affected phase.
- **Low** — Minor framing issue or acknowledged simplification. Can be resolved during implementation or post-MVP.

Findings are grouped by theme. Each finding cites specific document sections and MMT references.

---

## What the Documents Get Right

Before the findings, credit where it's due — the core framework is strong:

- **Endogenous money creation** (ECONOMIC-MODEL.md lines 85-102) — correctly described. "Loans create deposits" with explicit "NOT lending out existing deposits" language. Matches Bank of England (McLeay et al. 2014) and Wray (2015) Ch. 3-4.
- **Two money circuits** (ECONOMIC-MODEL.md lines 26-83) — correctly modeled with reserves vs. deposits, proper isolation rules, banks bridging both. Consistent with Fullwiler (2008).
- **Government spending as currency creation / taxation as destruction** (ECONOMIC-MODEL.md lines 7-12) — correct. Matches Mosler (1995), Wray (2015) Ch. 1-3.
- **Sectoral balances identity** (ECONOMIC-MODEL.md lines 14-24) — correct, with proper closed/open economy variants. Matches Godley (1999), Mitchell/Wray/Watts (2019) Ch. 6.
- **Bond issuance as reserve drain** (ECONOMIC-MODEL.md lines 350-357) — the "MMT Context" paragraph is accurate. Matches Bell/Kelton (2000), Fullwiler (2020).
- **Central bank as buyer of last resort** (ECONOMIC-MODEL.md line 364) — correct, establishing interest rate as policy choice. Matches Mosler (1993), Wray (2015) Ch. 5.
- **Cost-plus markup pricing with ULC** (ECONOMIC-MODEL.md lines 198-239) — consistent with post-Keynesian pricing theory (Lee 1998, Lavoie 2014).
- **Three inflation buffers** (ECONOMIC-MODEL.md lines 218-226) — distinctly MMT, correctly specified.
- **Bond interest as government spending** (ECONOMIC-MODEL.md line 374) — correct, often missed in other models.
- **No reserve constraint on bank lending** (ECONOMIC-MODEL.md line 338) — correct. "Banks lend first, seek reserves after."
- **Wage stickiness** (ECONOMIC-MODEL.md line 289) — downward wage rigidity is empirically well-established and consistent with post-Keynesian labor market theory.

---

## Critical Findings

### F1. Tax-Before-Spend Ordering Contradicts MMT Causation

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (lines 445-449), ARCHITECTURE.md (Section 4.1) |
| MMT reference | Wray (2015) Ch. 1-3; Mosler (1995); Mitchell/Wray/Watts (2019) Ch. 20 |

**The problem:** The Government Phase processes step (a) "Collect taxes" before step (b) "Execute spending." This ordering embeds the mainstream sequence: collect revenue, then spend.

MMT's central operational insight is the reverse: the government must spend (or lend) currency into existence before it can be taxed back. As Wray (2015) states: "The government must spend first, providing the currency the private sector needs to pay taxes." Mosler's business card analogy: you cannot collect cards you haven't handed out.

In the very first tick of the simulation, the private sector has no money yet. If taxes are collected first, taxes on tick 1 must either be zero (because there's nothing to tax) or the ordering is cosmetic. Either way, the sequence is pedagogically misleading and operationally inverted.

**Why it matters:** The sequence is visible in the transaction log, the UI pipeline view (FR-TIM-003), and the Architecture's data flow diagram. Even if the game's teaching role is secondary, the model itself should reflect the correct operational sequence — spending logically and operationally precedes taxation.

**Suggested fix:** Reorder the Government Phase:

```
1. GOVERNMENT PHASE
   a. Execute spending (creates reserves + deposits)
   b. Pay interest on outstanding bonds (creates reserves + deposits)
   c. Collect taxes (destroys deposits + reserves)
   d. Issue new bonds if needed (drains reserves)
```

This also correctly places bond issuance last: the government spends (adding reserves to the banking system), then drains excess reserves via bond sales. This matches the MMT description already in the ECONOMIC-MODEL.md "MMT Context" section (line 355): "The government spends first (creating reserves) and then issues bonds (swapping reserves for bonds)."

---

### F2. Government Must Not Have a Financial Spending Constraint

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (lines 110-113), ARCHITECTURE.md (Section 3.2) |
| MMT reference | Wray (2015) Ch. 1; Mosler (2010) Fraud #1; Kelton (2020) Ch. 1 |

**The problem:** The Government balance sheet tracks a "Treasury account at central bank (reserves)" (ECONOMIC-MODEL.md line 111). The documents never explicitly state that this balance cannot constrain spending.

**Design decision confirmed:** The author has confirmed that the government can always spend (no financial constraint). This must be made explicit in the documents and enforced in implementation.

**Why it matters:** This is Mosler's "Deadly Innocent Fraud #1": the belief that the government must raise money (via taxes or borrowing) before it can spend. If any code path checks the Treasury balance before allowing spending, the model is fundamentally wrong. The constraint on government spending is inflation (real resource availability), not the Treasury account balance.

**Suggested fix:** Add an explicit statement to ECONOMIC-MODEL.md, Government section (after line 113):

> **Critical design rule:** The Treasury account balance never constrains government spending. The government, as currency issuer, can always spend by crediting bank reserve accounts. The Treasury balance is tracked for accounting purposes only — it is not a budget constraint. The real constraint on government spending is inflation (available real resources), not money.

Also add to ARCHITECTURE.md Section 3.2 (`ISimulationCommands`), as a comment or documentation note:

> `SetSpendingLevel()` must never reject a spending level based on the Treasury account balance. The inflation consequences of excessive spending are modeled through the pricing engine, not through a financial constraint on the government.

---

### F3. Production Function Form Undefined — Must Be Leontief

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (lines 175-179, 204-210), MODDING.md (sectors.json) |
| MMT reference | Godley/Lavoie (2012) throughout; Lavoie (2014) *Post-Keynesian Economics* Ch. 3 |

**The problem:** Sector data includes input weights (labor 0.7, land 0.3 for agriculture) but the production function form is never specified.

**Design decision confirmed:** Leontief (fixed proportions), consistent with the Godley/Lavoie SFC tradition.

**Why it matters:** The production function determines how the economy responds to input changes. Cobb-Douglas (smooth substitution) implies that capital can freely replace labor — this underpins neoclassical marginal productivity theory and the claim that wages equal marginal product. This is theoretically incompatible with MMT/post-Keynesian economics, where wages are determined by bargaining power, institutional factors, and labor market conditions — not marginal productivity.

Leontief (fixed proportions) means: to produce X units of output, you need exactly Y workers and Z materials. You cannot substitute one for the other in the short run. This is:
- Empirically accurate for most production processes
- Consistent with every SFC model in Godley/Lavoie (2012)
- Simpler to implement (no optimization/substitution logic needed)

**Suggested fix:** Add to ECONOMIC-MODEL.md, before or within the Pricing Model section:

> ### Production Function: Fixed Proportions (Leontief)
>
> Each sector uses fixed technical coefficients for production. To produce one unit of output, a firm needs fixed quantities of each input:
>
> ```
> Output = min(Input_1 / coefficient_1, Input_2 / coefficient_2, ...)
> ```
>
> For example, if agriculture has coefficients labor: 0.7, land: 0.3, then producing 100 units of food requires exactly 70 units of labor and 30 units of land. A shortage of either input constrains output proportionally.
>
> This follows the Godley/Lavoie (2012) SFC modeling tradition. Input coefficients are fixed in the short run but are data-driven (loaded from sectors.json), allowing mods to represent different technological levels. Long-run technological change (coefficient drift) is a post-MVP enhancement.

Update the sector data schema description to clarify that `inputWeights` are Leontief coefficients, not Cobb-Douglas exponents.

---

## Medium Findings

### F4. Missing: Seller's Inflation / Profit-Push Inflation

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (lines 213-216) |
| MMT reference | Weber/Wasner (2023) "Sellers' Inflation"; Lavoie (2014) Ch. 8; Mitchell/Wray/Watts (2019) Ch. 14 |

**The problem:** The pricing model's markup demand adjustment only responds to demand-capacity ratios. There is no mechanism for firms to proactively increase markups independently of demand conditions.

Weber & Wasner (2023) identified "seller's inflation" as central to the 2021-2023 inflation episode: firms used supply disruptions as cover to increase profit margins even when demand was below capacity. Post-Keynesian conflict theory (Rowthorn 1977, Lavoie 2014) models inflation as a conflict over income distribution between workers and firms.

**Why it matters:** Without profit-push inflation, the model's inflation dynamics are incomplete. The three-buffer system (ECONOMIC-MODEL.md lines 218-226) includes profit margin compression (buffer 3) but has no mechanism for the reverse: margin expansion by firms with market power. In reality, markup increases are a primary source of inflation, and markup adjustment is asymmetric — firms raise markups quickly but lower them slowly.

**Suggested fix:** Add asymmetric markup adjustment to ECONOMIC-MODEL.md:

> **Markup Asymmetry:**
> Markups adjust asymmetrically. Firms raise markups more readily than they lower them:
> - Upward adjustment speed: fast (firms exploit pricing power quickly when demand is strong or supply is constrained)
> - Downward adjustment speed: slow (firms resist margin compression, similar to downward wage stickiness)
>
> This can be parameterized per sector in sectors.json (e.g., `markupUpwardSpeed: 0.5`, `markupDownwardSpeed: 0.1`).

Also consider adding a supply-disruption trigger: when input availability drops (e.g., raw material shortage), surviving firms can raise markups even without excess demand. This models the seller's inflation mechanism.

---

### F5. "Debt-to-GDP Ratio" Indicator Embeds Mainstream Framing

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (line 493), PRD (FR-SIM-004) |
| MMT reference | Kelton (2020) Ch. 3-4; Wray (2015) Ch. 6 |

**The problem:** "Debt-to-GDP ratio: Total government bonds outstanding / GDP" is listed as a key indicator. In mainstream economics, this ratio is presented as a fiscal sustainability measure, implying "high ratio = danger."

For a monetarily sovereign government, the debt-to-GDP ratio is not a meaningful constraint. Japan operates at 250%+ without fiscal crisis. The "national debt" is the accounting record of net financial assets the government has added to the non-government sector. As Kelton (2020) argues, calling it "debt" is itself misleading — it is the non-government sector's net savings in government securities.

**Why it matters (even with teaching as secondary):** If this indicator is used in scenario win/lose conditions, it could create gameplay that punishes MMT-consistent policy. Players might try to "reduce the debt ratio" by running surpluses, which drains private sector savings — exactly what MMT warns against.

**Suggested fix:** Keep the indicator (it's a real-world metric players will recognize) but:
1. Add a companion indicator: "Private sector net financial assets" (which is the mirror image — government bonds outstanding are private sector assets)
2. In scenario design, never use the debt-to-GDP ratio as a lose condition. Use inflation, unemployment, and private sector financial distress instead.
3. Consider renaming to "Government bonds outstanding / GDP" (factually neutral).

---

### F6. Missing: Explicit Statement That Government Cannot Default on Own Currency

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (Government section), DESIGN.md |
| MMT reference | Wray (2015) Ch. 1; Mosler (2010) Fraud #1 |

**The problem:** DESIGN.md line 31 states the government "cannot 'run out' of its own money." But the documents never state the corollary: a monetarily sovereign government **cannot involuntarily default** on obligations denominated in its own currency. The bond system (ECONOMIC-MODEL.md lines 350-376) describes bond issuance and interest payments but never addresses default.

**Why it matters:** If the game ever allows or implies government default on bonds (e.g., if the player accumulates "too much debt"), this would contradict a core MMT proposition. The model must make it mechanically impossible for the government to fail to make a bond interest or principal payment.

**Suggested fix:** Add to ECONOMIC-MODEL.md, Government Bonds section (after line 357):

> **No sovereign default:** The government, as currency issuer, always pays bond interest and principal when due. Bond payments are government spending (currency creation). Sovereign default on domestic-currency-denominated debt is a political choice, never an operational necessity. The game does not model voluntary default.

---

### F7. Saving-Investment Causation Direction Ambiguous

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (lines 388-406) |
| MMT reference | Wray (2015) Ch. 2; Godley/Lavoie (2012) Ch. 7 (Model BMW); Keynes (1936) General Theory Ch. 7 |

**The problem:** The investment section describes funding from "retained profits AND/OR bank loans." Listing retained profits first, with bank loans as the alternative, implicitly suggests that prior saving is the primary funding source for investment.

In post-Keynesian/MMT economics, the causation runs from investment to saving, not the reverse. When a firm invests (funded by a bank loan), the loan creates a deposit. That deposit is someone's saving. At the macro level, investment spending generates income, part of which is saved. As Keynes established: saving is the residual of income after consumption, and income is determined by investment and government spending decisions.

Retained profits are themselves the result of prior investment and spending — they are not an independent "pool" that funds new investment. The availability of bank credit is the enabling factor for investment, not prior saving.

**Why it matters:** The loanable funds theory (saving funds investment) is one of the key mainstream ideas that MMT rejects. If the game's investment mechanics work by checking "does the firm have enough retained profits?" before allowing investment, this models loanable funds.

**Suggested fix:** Reframe the investment funding section (ECONOMIC-MODEL.md lines 398-401):

> **Funding sources:**
> 1. Bank loans (credit creation — new money is created to fund the investment)
> 2. Internal funds (retained profits redirected to investment)
>
> Bank lending is the primary enabler of investment. A firm's decision to invest is based on expected demand, capacity utilization, and borrowing costs — not on the pre-existence of savings. When investment is bank-funded, the loan creates new deposits (which become someone's saving), so at the macro level investment creates saving, not the reverse.

---

### F8. No Mechanism for Central Bank Reserve Provision

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (lines 85-102, 127-130), ARCHITECTURE.md (Section 4.1) |
| MMT reference | Fullwiler (2008) "Modern Central Bank Operations"; Wray (2015) Ch. 4 |

**The problem:** The ECONOMIC-MODEL.md correctly states "banks lend first and seek reserves after (or the central bank provides them)" (line 102). But no mechanism exists for banks to obtain reserves from the central bank. The Central Bank section (lines 124-130) lists "set the policy interest rate" and "buy/sell government bonds" but not the core operational function: **providing reserves to banks on demand** (via discount window, standing lending facility, or repo operations).

**Why it matters:** If banks cannot obtain reserves when needed, the model breaks down. After a bank creates a loan (adding deposits), the borrower may spend those deposits, causing reserves to flow to another bank. The lending bank may then need reserves for settlement. Without central bank reserve provision, the banking system could seize up — which contradicts the endogenous money framework.

**Suggested fix:** Add to ECONOMIC-MODEL.md, Central Bank section (after line 130):

> - Provide reserves to commercial banks on demand at the policy rate (lender of last resort / standing facility)
>
> This ensures that bank lending is never constrained by reserve availability. Banks lend based on creditworthiness of borrowers and profitability, then obtain any needed reserves from the central bank. In the MVP with a single aggregate bank, reserve provision is simplified (the single bank's reserves are the net result of government spending minus taxation, plus any central bank operations).

---

### F9. Bond Issuance Described as Deficit Management, Not Reserve Management

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (line 352, line 360) |
| MMT reference | Bell/Kelton (2000); Fullwiler (2006, 2020) |

**The problem:** Line 352 says "The government issues bonds to manage the deficit." Lines 355-356 correctly describe bonds as a reserve-draining operation. These two statements contradict each other. Bond issuance manages **reserves** (monetary policy), not the deficit (fiscal policy). The deficit is the result of spending minus taxation — it exists regardless of whether bonds are sold.

The "MMT Context" paragraph (lines 354-357) is correct. The preceding sentence (line 352) contradicts it.

**Suggested fix:** Change line 352 from:

> The government issues bonds to manage the deficit, following real-world institutional practice.

To:

> The government issues bonds following real-world institutional practice. As described in the MMT Context below, bond issuance is a reserve management operation — it drains excess reserves from the banking system, not a means of "financing" the deficit.

---

### F10. Missing: Interest Rate as Fiscal Channel

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (lines 313-319, 372-376) |
| MMT reference | Mosler (1993, 1995); Fullwiler (2006); Wray (2015) Ch. 5 |

**The problem:** The model describes the central bank policy rate affecting lending rates (line 317: "central bank policy rate: set by central bank") and bond interest flowing to wealth holders (line 376). But it never notes the fiscal channel of interest rate policy: **higher interest rates increase government interest payments to bondholders, which is a fiscal stimulus** (net income flow to the private sector).

This is a distinctive MMT insight. Mainstream economics assumes higher rates are contractionary (reducing borrowing and investment). MMT points out the countervailing fiscal channel: higher rates → higher government interest payments → more income to bond holders → potentially stimulative. The net effect is ambiguous, which is one reason MMT prefers fiscal policy over monetary policy as the primary demand management tool.

**Why it matters:** With the MVP's fixed 0% rate, this is not immediately relevant. But the bond system already generates interest payments that "create new currency" (line 374). If interest rates ever become variable (post-MVP), the fiscal channel will be an emergent property of the model — worth documenting now.

**Suggested fix:** Add a note to ECONOMIC-MODEL.md, Interest Payments section (after line 376):

> **Note on the fiscal channel of interest rate policy:** Higher interest rates increase government bond interest payments, which are themselves government spending (currency creation). This means higher rates have a stimulative fiscal effect (more income to bond holders) that partially offsets the contractionary effect on borrowing. This is a key reason MMT prefers fiscal policy (spending and taxation) over monetary policy (interest rates) for demand management.

---

## Low Findings

### F11. Population Is Fixed — Limits Demonstration of Real Resource Constraints

| Aspect | Detail |
|---|---|
| Documents | Implicit throughout all documents |
| MMT reference | Wray (2015) Ch. 7; Mitchell/Wray/Watts (2019) Ch. 12 |

Fixed population means the labor force is constant. This is an acceptable MVP simplification, but it limits the demonstration of MMT's core constraint: **real resources**. With a fixed population, there is a hard ceiling on employment, making the "spending into full capacity = inflation" dynamic work. But the game cannot demonstrate:

- Demographic change affecting the fiscal position
- Immigration expanding the labor force (and thus fiscal space)
- Skills mismatch (all workers within a class are identical)

**No fix needed for MVP.** Note as a post-MVP enhancement.

---

### F12. No Currency Demand Mechanism (Taxes Drive Currency)

| Aspect | Detail |
|---|---|
| Documents | All documents |
| MMT reference | Wray (1998) *Understanding Modern Money*; Mosler (1995); Knapp (1905) |

The Chartalist argument — taxes create demand for the currency, which is why fiat money has value — is absent from all documents. DESIGN.md line 31 says "taxation removes currency from circulation" and ECONOMIC-MODEL.md describes tax as destruction, but neither explains why the currency has value in the first place.

**In the MVP closed economy with a single currency, this is an acceptable simplification.** There is no alternative currency to compete with, so currency demand is implicit. When the foreign sector is added (post-MVP), currency demand becomes important — exchange rates depend on it.

**No fix needed for MVP.** Add a note to ECONOMIC-MODEL.md acknowledging this:

> **MVP simplification:** The model does not explicitly model currency demand. In MMT, tax obligations drive demand for the currency (Chartalism). In the MVP closed economy with a single currency, this is implicit — all agents must use the sovereign currency because no alternative exists. When the foreign sector is added, currency demand and exchange rate dynamics will require explicit modeling of this mechanism.

---

### F13. No Automatic Stabilizers in MVP

| Aspect | Detail |
|---|---|
| Documents | PRD (Section 4 — out of scope), DESIGN.md (lines 36-44) |
| MMT reference | Tcherneva (2020); Mitchell/Wray (2005); Mitchell/Muysken (2008) |

The MVP has no automatic fiscal stabilizers: no progressive taxation, no unemployment benefits, and no Job Guarantee. All fiscal policy is manual player action.

The Job Guarantee is MMT's signature policy proposal — a buffer stock of employed workers replacing the mainstream buffer stock of unemployed (NAIRU). Its absence from the MVP is the single largest gap from an MMT completeness perspective. However, the JG is correctly described in DESIGN.md (lines 36-44) and explicitly scoped as post-MVP.

**No fix required.** The MVP scope decision is reasonable. The JG should be the first major post-MVP addition.

---

### F14. "Government balance" Parenthetical Frames Deficit as Negative

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (line 491) |

"Government balance: Spending - Tax revenue (negative = deficit)" — the parenthetical frames deficits as the negative/abnormal case. In MMT, deficits are the normal state for a growing economy, especially one running a trade deficit. Surpluses are the anomaly that drains private sector savings.

**Suggested fix:** Remove the parenthetical, or change to: "Government balance: Spending - Tax revenue (positive = net currency creation, negative = net currency withdrawal)"

---

### F15. "Profit-Driven" Firms Only

| Aspect | Detail |
|---|---|
| Documents | ECONOMIC-MODEL.md (line 173) |

All firms are described as "profit-driven." This excludes public enterprises, non-profits, and cooperatives — all of which exist in real economies and have different pricing/investment behavior. Post-Keynesian pricing theory (Lee 1998) distinguishes between price-setting behavior in competitive vs. oligopolistic markets.

**No fix needed for MVP.** Acceptable simplification. Note for post-MVP: adding public enterprises (which might price at cost, not cost-plus-markup) would create interesting policy dynamics around public vs. private provision.

---

## Summary Table

| ID | Severity | Finding | Status |
|---|---|---|---|
| F1 | Critical | Tax-before-spend ordering contradicts MMT | **Resolved** — reordered in ECONOMIC-MODEL.md and ARCHITECTURE.md |
| F2 | Critical | Government must not have financial spending constraint | **Resolved** — critical design rule added to ECONOMIC-MODEL.md and ARCHITECTURE.md |
| F3 | Critical | Production function undefined — must be Leontief | **Resolved** — Leontief I-O production function added to ECONOMIC-MODEL.md |
| F4 | Medium | Missing seller's inflation / profit-push mechanism | **Resolved** — asymmetric markup adjustment + supply-side markup pressure added to ECONOMIC-MODEL.md |
| F5 | Medium | Debt-to-GDP framing embeds mainstream assumptions | **Resolved** — renamed to "Government bonds outstanding / GDP", added "Private sector net financial assets" companion, neutral government balance framing |
| F6 | Medium | Missing: government cannot default on own currency | **Resolved** — "No Sovereign Default" subsection added to ECONOMIC-MODEL.md Government Bonds section |
| F7 | Medium | Saving-investment causation direction ambiguous | **Resolved** — reframed: bank loans listed first, removed "can they afford it" framing, added investment-creates-saving note |
| F8 | Medium | No mechanism for central bank reserve provision | Fix: add reserve provision to CB actions |
| F9 | Medium | Bond issuance framed as deficit management | Fix: correct to reserve management |
| F10 | Medium | Missing: interest rate fiscal channel | Fix: add explanatory note |
| F11 | Low | Fixed population limits resource constraint demo | Post-MVP |
| F12 | Low | No currency demand mechanism (Chartalism) | Post-MVP; add note |
| F13 | Low | No automatic stabilizers (JG) in MVP | Post-MVP; already planned |
| F14 | Low | "Government balance" parenthetical frames deficit as negative | **Resolved** — reframed as "net currency creation / net currency withdrawal" |
| F15 | Low | Profit-driven firms only | Post-MVP |

## Cross-References to Existing Review

Several findings from REVIEW-consistency.md are reinforced or reframed by this audit:

| Consistency Finding | MMT Audit Relevance |
|---|---|
| C4 (Policy lag system) | Policy lags are important to MMT (real-world institutional constraints). The lag system architecture gap remains. |
| C5 (Investment absent from Architecture) | Investment mechanics are important AND must follow post-Keynesian causation (F7). |
| C6 (Three inflation buffers) | Confirmed as core MMT mechanism. Must also add seller's inflation (F4). |
| M6 (FR-SIM-002 oversimplifies circuit access) | Resolved by C2 fix. Banks correctly bridge both circuits. |
| M7 (Bond yields without secondary market) | Acceptable for MVP. "Average coupon rate" is the honest label. |
