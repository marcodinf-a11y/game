# ADR-0003: Banks-Only Bond Auctions, No Household Participation

**Status:** Accepted

## Context

FR-BND-001 originally stated: "Banks and high-income households must be able to bid" on government bonds. This created three problems:

1. **Real-world accuracy:** In virtually all sovereign bond markets (Germany, UK, Japan), only designated banks/dealers can bid at primary auctions. The US TreasuryDirect program is the exception, not the rule.
2. **MMT consistency:** MMT literature (Mosler, Wray) describes bond issuance as a reserve-draining operation where banks exchange reserves for securities. Households are absent from this operational description.
3. **Circuit isolation violation:** The game's own money circuit rules (FR-SIM-002) restrict non-bank private agents to the deposits circuit, while bond purchases at auction require reserves. Household auction participation would violate circuit isolation.

## Decision

Remove household participation from FR-BND-001 entirely. Only commercial banks participate in primary bond auctions.

## Consequences

- No `IHouseholdClass` bond properties needed.
- No Phase 6 household bond tests needed.
- Circuit isolation rules are internally consistent — non-bank private agents never touch reserves.
- Households can acquire bonds post-MVP via a secondary market (currently out of scope).
- Simpler bond auction implementation: only one bidder type (banks).
