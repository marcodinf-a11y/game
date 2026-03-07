# ADR-0005: AIDS Consumption Model over Hierarchical Needs

**Status:** Accepted

## Context

FR-AGT-004 originally required households to "consume according to hierarchical needs: survival > shelter > comfort > luxury." This Maslow-style hierarchy had no architectural specification — no definition of how goods map to need levels, how the hierarchy is evaluated, or how it interacts with the goods market.

The hierarchical model also had design problems:

- **Arbitrary thresholds:** Where does "survival" end and "shelter" begin? Any threshold is a design parameter that needs calibration with no empirical basis.
- **Cliff effects:** Strict priority ordering creates discontinuities — a small income change crossing a threshold causes a large demand shift.
- **No price sensitivity within tiers:** The model doesn't naturally handle substitution between goods at the same need level.
- **Goods abstraction overhead:** Required a separate "goods" layer mapping products to need levels.

## Decision

Replace the hierarchical needs model with the **Almost Ideal Demand System** (AIDS, Deaton & Muellbauer 1980).

AIDS computes budget shares (fraction of income spent on each sector) as a function of relative prices and real income. Parameters are per household class, empirically sourced from published US Consumer Expenditure Survey (CEX) estimates. The model satisfies standard economic theory constraints (adding-up, homogeneity, symmetry).

Key design consequences of adopting AIDS:

- **Dropped the goods abstraction.** Goods are replaced by a hierarchical sector structure (sectors with `parentId` for post-MVP sub-sector expansion).
- **No subsistence floor.** Welfare state is explicit policy, not hidden in consumption math.
- **Restructured sectors** from 3 (agriculture, industry, services) to 4 (agriculture & primary, manufacturing, construction, services) to match standard macroeconomic textbook aggregation.
- **Added government as direct employer and procurer** (`IGovernmentDemand` interface) — government competes in the labor market and procures from private sectors, creating visible real resource competition central to MMT.

## Consequences

- `IConsumptionEngine` interface with full AIDS formula, per-class parameterization.
- `IHouseholdClassState` exposes `BudgetShares`, `ConsumptionBySector`, and `TotalConsumption` instead of a single `ConsumptionSpending`.
- Parameters are data-driven and empirically grounded — calibration uses real CEX data rather than invented thresholds.
- Engel's law emerges naturally from the model (food budget share declines as income rises) rather than being hard-coded.
- Price substitution effects are built in — no separate elasticity system needed.
- Banking is excluded from AIDS demand (consistent with MMT/SFC tradition — financial intermediation is not consumption).
