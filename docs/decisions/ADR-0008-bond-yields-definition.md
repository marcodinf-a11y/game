# ADR-0008: Bond Yields as Weighted Average Coupon Rate

**Status:** Accepted

## Context

FR-SIM-004 lists "bond yields" as a required economic indicator. But the PRD explicitly excludes "bond secondary market" from scope (Section 4). Without a secondary market, there is no market price for bonds, and the traditional yield-to-maturity calculation (based on market price vs. face value) is meaningless. The "yield" indicator risks being either redundant with the coupon rate or misleading.

## Decision

Define "bond yields" as the **weighted average coupon rate** across outstanding bonds — the government's average cost of debt.

Formula: `Sum(bond.CouponRate * bond.FaceValue) / Sum(bond.FaceValue)`, returning 0 when no bonds are outstanding.

## Consequences

- Meaningful without a secondary market: auction demand (FR-BND-001) produces varying coupon rates across issuances, and the weighted average shifts as old bonds mature and new ones are issued.
- Pedagogically valuable: demonstrates the MMT insight that bond rates are policy-influenced, since the central bank as buyer-of-last-resort effectively sets a rate ceiling.
- If a secondary market is added post-MVP, the indicator definition can be updated to use market-implied yields without breaking the interface — only the calculation changes.
