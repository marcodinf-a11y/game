# Parameter Commentary

This document explains the economic rationale behind every value in `parameters.json`. It serves two purposes:

1. **For contributors:** Understand *why* a parameter has a particular value, not just what the value is.
2. **For modders:** Make informed decisions when overriding values via the modding system (FR-MOD-001).

All values listed here are tunable — modders can override any parameter by providing a replacement `parameters.json` in their mod directory. The base game values below represent reasonable defaults grounded in real-world data and macroeconomic research.

As new parameters are added to `parameters.json` in later implementation phases, their commentary should be added here.

## Conversion Note

Parameters expressed as "per tick" rates correspond to monthly values (one tick = one month). To convert between monthly and annual rates:

```
annual = 1 - (1 - monthly)^12
monthly = 1 - (1 - annual)^(1/12)
```

For small rates, `annual ≈ monthly × 12` is a reasonable approximation.

## Investment Parameters

| Parameter path | Value | Unit | Rationale |
|---|---|---|---|
| `investment.capacityUtilizationThreshold` | 0.85 | ratio | Standard macro threshold — firms begin expansion plans when running above ~85% capacity. Below this, idle capacity can absorb demand increases without new investment. Above it, bottlenecks and delivery delays make expansion profitable. Based on Federal Reserve capacity utilization data where investment spending historically accelerates above 80-85%. |
| `investment.depreciationRates.publicInfrastructure` | 0.005 | per tick (monthly) | ~6% annual depreciation. Real infrastructure (roads, bridges, utilities) depreciates at 2-4% annually, but game time is compressed and maintenance neglect accelerates decay. 6% balances realism with gameplay pressure to maintain spending. |
| `investment.depreciationRates.publicServices` | 0.004 | per tick (monthly) | ~4.7% annual depreciation. Public service capital (training programs, institutions) degrades more slowly than physical infrastructure but still requires sustained funding. Lower rate than infrastructure reflects that institutional knowledge persists longer. |
| `investment.depreciationRates.agricultureAndPrimary` | 0.003 | per tick (monthly) | ~3.5% annual. Agricultural and primary sector capital (land improvements, irrigation, extraction equipment) is durable. Slower depreciation reflects long-lived nature of agricultural and primary infrastructure. |
| `investment.depreciationRates.manufacturing` | 0.007 | per tick (monthly) | ~8.1% annual. Manufacturing equipment has higher wear and faster technological obsolescence than other sectors. Aligns with BEA estimates for industrial equipment useful life of 10-15 years. |
| `investment.depreciationRates.construction` | 0.008 | per tick (monthly) | ~9.2% annual. Construction sector capital (cranes, scaffolding, heavy equipment) operates in harsh conditions with high physical wear and frequent replacement cycles. Higher than manufacturing due to mobile deployment, exposure to elements, and project-based equipment turnover. |
| `investment.depreciationRates.services` | 0.005 | per tick (monthly) | ~6% annual. Service sector capital (office equipment, IT systems) has moderate depreciation. Faster than agriculture due to technology refresh cycles, slower than manufacturing due to less physical wear. |
| `investment.lags.infrastructureToCapacity.min` | 6 | ticks (months) | FR-TIM-001 specifies 6-12 ticks. Minimum reflects fast-track infrastructure projects (equipment upgrades, minor expansions). |
| `investment.lags.infrastructureToCapacity.max` | 12 | ticks (months) | FR-TIM-001 specifies 6-12 ticks. Maximum reflects major projects (new facilities, transportation infrastructure). Actual lag for each investment is drawn uniformly from [min, max]. |
| `investment.lags.servicesToProductivity.min` | 12 | ticks (months) | FR-TIM-001 specifies 12-24 ticks. Education, training, and institutional improvements take longer to translate into measurable productivity gains. |
| `investment.lags.servicesToProductivity.max` | 24 | ticks (months) | FR-TIM-001 upper bound. Full effect of workforce training and institutional reform. |
| `investment.lags.privateInvestmentToCapacity.min` | 3 | ticks (months) | FR-TIM-002 specifies 3-6 ticks. Firms can expand faster than government — purchasing and installing equipment takes months, not years. |
| `investment.lags.privateInvestmentToCapacity.max` | 6 | ticks (months) | FR-TIM-002 upper bound. Larger expansions (new production lines, facilities) take up to 6 months. |
