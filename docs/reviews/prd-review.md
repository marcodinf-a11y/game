# PRD Review

**Date:** 2026-03-07
**Document reviewed:** `docs/requirements/PRD.md`
**Cross-referenced:** ECONOMIC-MODEL.md, GAME-DESIGN.md, ARCHITECTURE.md, IMPLEMENTATION-PLAN.md, all ADRs, PARAMETERS-COMMENTARY.md, mmt-accuracy.md

## Summary

Four-axis review of the PRD covering correctness, staleness, completeness, and ambiguity. Findings are grouped by priority and linked to GitHub issues for tracking.

---

## 1. Correctness -- Contradictions with Other Documents

### C1: FR-INV-002 says "industry sector" -- should be "manufacturing"
- **PRD:** FR-INV-002 (line 196): "Capital goods must be produced by the industry sector"
- **Conflict:** FR-AGT-005 (line 112) in the same PRD says "manufacturing sector." ADR-0005, ECONOMIC-MODEL, and GAME-DESIGN all use "manufacturing." No "industry" sector exists in the 4-sector model.

### C2: FR-PRC-002 omits seller's inflation from ADR-0010
- **PRD:** FR-PRC-002 (line 127): "Inflation must only occur when all three buffers are exhausted"
- **Conflict:** ADR-0010 added seller's inflation (supply-side markup pressure) as a fourth channel independent of demand buffers. ECONOMIC-MODEL SS313-327 documents this.

### C3: FR-PRC-001 treats markup adjustment as symmetric
- **PRD:** FR-PRC-001 (line 121): "Markup must adjust based on demand relative to capacity"
- **Conflict:** ADR-0010 specifies asymmetric speeds (upward fast 0.5, downward slow 0.1) with per-sector parameters `markupUpwardSpeed` and `markupDownwardSpeed`.

### C4: ECONOMIC-MODEL describes CB acting in secondary market
- **PRD:** Section 5 (line 468) excludes secondary market from MVP. FR-BND-001 describes CB as auction backstop only.
- **Conflict:** ECONOMIC-MODEL SS172 says CB "Buy/sell government bonds on secondary market." The PRD is correct; ECONOMIC-MODEL needs fixing.

### C5: ECONOMIC-MODEL uses soft language for CB rate
- **PRD:** FR-AGT-002: "fixed at 0 for MVP" (mandatory)
- **Conflict:** ECONOMIC-MODEL SS173: "can default to 0" (optional). ECONOMIC-MODEL's own MVP table agrees with PRD. ECONOMIC-MODEL needs fixing.

### C6: ECONOMIC-MODEL uses two-component lending rate formula
- **PRD:** FR-AGT-003/FR-BNK-002: "CB policy rate + spread + risk premium" (three components)
- **Conflict:** ECONOMIC-MODEL SS194 uses two components. Its own SS465 agrees with PRD. ECONOMIC-MODEL needs fixing.

### C7: PRD does not specify Leontief production function
- **PRD:** FR-AGT-005 (line 107): "Each sector must have different input/output mixes"
- **Conflict:** ADR-0009, ECONOMIC-MODEL SS248-284, and GAME-DESIGN SS133 all specify Leontief I-O with fixed technical coefficients. The PRD is too vague; a non-Leontief implementation would satisfy it.

> **Note:** C4-C6 are issues in ECONOMIC-MODEL.md, not the PRD. They should be fixed in that document.

---

## 2. Staleness -- Outdated Requirements

### S1: FR-INV-002 "industry sector" (same as C1)
Leftover from old 3-sector model before ADR-0005.

### S2: FR-PRC-002 three-buffer-only model (same as C2)
Missing seller's inflation from ADR-0010.

### S3: NFR-EXT-001 references "goods/resources"
- **PRD:** NFR-EXT-001 (line 417): "Adding a new good/resource must not require modifying existing good code"
- **Issue:** Goods abstraction dropped by ADR-0005. Should reference "sector/sub-sector."

### S4: FR-PRC-003 "weighted average across all goods"
- **PRD:** FR-PRC-003 (line 131): "Price level must be a weighted average across all goods"
- **Issue:** Should say "across all sectors" since goods are not a first-class concept.

### S5: FR-AGT-005 / FR-INV-002 missing Leontief reference (same as C7)
Neither section references the Leontief I-O production function from ADR-0009.

---

## 3. Completeness -- Missing Requirements

### G1: Leontief I-O production function (same as C7/S5)
No FR for fixed-proportion production or inter-sector I-O matrix.

### G2: Seller's inflation / supply-side markup pressure (same as C2/S2)
No FR for the supply pressure mechanism from ADR-0010.

### G3: Government spending cannot be financially constrained
Critical MMT invariant (ECONOMIC-MODEL SS133-135). Spending must never be rejected based on Treasury balance.

### G4: No sovereign default on domestic-currency bonds
Government must always pay bond interest/principal regardless of Treasury balance.

### G5: Overlapping policy change behavior (replace model)
ADR-0002 decides replace-and-reset for duplicate in-pipeline changes. PRD is silent.

### G6: Deterministic simulation given same seed + inputs
Load-bearing for SFC replay recovery (ADR-0004) and future save/load (ADR-0006). Currently only mentioned as a test concern in NFR-CQA-002, not as a simulation-engine requirement.

### G7: Policy input log recording
Required by ADR-0004 (SFC error recovery) and ADR-0006 (replay-based save/load). No FR exists.

### G8: Inventory rationing when demand exceeds supply
ECONOMIC-MODEL SS411-413 describes rationing. PRD only says "hold inventory of unsold goods."

### G9: AIDS parameter constraint validation
Adding-up, homogeneity, symmetry constraints (ECONOMIC-MODEL SS375-381, ARCHITECTURE SS3.13) need explicit validation beyond generic schema checks.

### G10: Disposable income definition for AIDS
ECONOMIC-MODEL SS369 defines it as "post-tax income + available credit - debt service." PRD never specifies what income feeds into AIDS.

### G11: Government wage behavior (civil service stickiness)
ECONOMIC-MODEL SS148, SS429-431 specifies slower wage adjustment via stickiness parameter. PRD only says "compete in the labor market."

### G12: Government procurement-to-sector demand mapping as data-driven
Needed for testability and modding. PRD mentions procurement at high level only.

---

## 4. Ambiguity -- Vague, Under-specified, or Contradictory

### Vague Thresholds / Missing Formulas

| ID | Requirement | Issue |
|---|---|---|
| A1 | FR-TIM-001, FR-TIM-002 | Lag ranges (1-2, 2-3, 6-12 ticks) with no rule for picking a value |
| A2 | FR-INV-001/002 | Depreciation "over time" -- no rate, method, or formula |
| A3 | FR-PRC-001 | Markup adjustment "based on demand relative to capacity" -- no functional form |
| A4 | FR-PRC-002 | "Buffers exhausted" -- no thresholds defined |
| A5 | FR-LBR-001 | "Sticky downward" -- no asymmetry factor or max per-tick decline |
| A6 | FR-LBR-002 | Sector mobility "over time" -- no friction cost or transition delay |

### Under-specified Mechanics

| ID | Requirement | Issue |
|---|---|---|
| A7 | FR-AGT-003 / Section 5 | Plural "Commercial Banks" but MVP is single aggregate bank |
| A8 | FR-BND-001 | Auction type unspecified (uniform price? discriminatory?) |
| A9 | FR-BND-002 | Bond maturity length unspecified |
| A10 | FR-BNK-002 | No DTI thresholds, collateral haircuts, or approval criteria |
| A11 | FR-BNK-003 | Loan durations and amortization unspecified |
| A12 | FR-PRC-003 | Price level weighting method unspecified |
| A13 | FR-AGT-004 | No population counts, class ratios, or reclassification rules |
| A14 | FR-SIM-003 | Tick phase names listed but operations per phase undefined |

### Internal Contradictions

| ID | Requirements | Issue |
|---|---|---|
| A15 | FR-CTL-001 vs FR-AGT-001 | Player controls 3 spending buckets but public employment costs relationship unspecified |
| A16 | FR-SIM-001 | "Government + Private = 0" but CB balance sheet treatment unclear |
| A17 | FR-BNK-003 | Mortgages "secured by housing" but no housing asset/sector exists |

### Unaddressed Edge Cases

| ID | Issue |
|---|---|
| A18 | All firms in a sector go bankrupt / zero output |
| A19 | Bank at zero reserves -- can it still lend? |
| A20 | 100% unemployment -- AIDS needs positive expenditure |
| A21 | Government spending set to zero -- money supply drains to zero |
| A22 | NFR-DTA-001 requires "restore to last consistent state" but no save mechanism |
| A23 | FR-GMD-002 "economic collapse" fail condition undefined |

---

## Suggested Priority

### High (blocks implementation or is factually wrong)
- C1/S1: "industry" to "manufacturing" (trivial fix)
- C2/S2/G2: Add seller's inflation requirement per ADR-0010
- C3: Add asymmetric markup adjustment per ADR-0010
- C7/S5/G1: Add Leontief I-O production function requirement per ADR-0009
- G3/G4: Add MMT invariants (no financial constraint on spending, no sovereign default)
- S3/S4: "goods" to "sectors" (trivial fix)

### Medium (important for implementation clarity)
- G5-G7: Policy replacement, deterministic simulation, input log
- G8-G12: Rationing, AIDS validation, disposable income, gov wages, procurement mapping
- A15-A17: Internal contradictions
- C4-C6: ECONOMIC-MODEL.md cross-doc fixes

### Lower (design detail to resolve during implementation)
- A1-A6: Vague thresholds -- acceptable if data files will specify values
- A7-A14: Under-specified mechanics -- may be intentionally deferred
- A18-A23: Edge cases -- important but can be resolved incrementally
