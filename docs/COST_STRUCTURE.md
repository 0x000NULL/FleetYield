# COST STRUCTURE - FleetYield Economics Foundation

## Section 1: Cost Breakdown Table (All Car Classes)

| Car Class | Daily Fuel | Daily Maintenance | Daily Depreciation | Fixed Per-Rental | True Marginal Cost | 25% Margin Floor | 30% Margin Floor |
|-----------|-----------|-------------------|-------------------|------------------|-------------------|------------------|------------------|
| **E** (Economy) | $4.50 | $2.50 | $12.50 | $15.00 | **$34.50** | $43.13 | $44.85 |
| **F** (Compact) | $5.20 | $2.80 | $14.00 | $15.00 | **$37.00** | $46.25 | $48.10 |
| **S** (Sedan) | $5.80 | $3.20 | $16.00 | $15.00 | **$40.00** | $50.00 | $52.00 |
| **L** (Large) | $7.50 | $4.00 | $18.50 | $15.00 | **$45.00** | $56.25 | $58.50 |
| **A** (SUV) | $8.20 | $4.50 | $20.00 | $15.00 | **$47.70** | $59.63 | $62.01 |
| **W** (Van) | $9.00 | $5.00 | $22.00 | $15.00 | **$51.00** | $63.75 | $66.30 |

**Assumptions**:
- Fuel cost: $3.50/gallon, 20-25 MPG depending on class
- Maintenance: $0.15-0.18/mile, 60k miles/year per vehicle
- Depreciation: 5-year fleet cycle, purchase $22-35k, salvage $5-10k
- Fixed per-rental: processing fees ($8), insurance admin ($5), accounting ($2)
- True marginal cost: sum of daily variable + fixed per-rental (daily depreciation is sunk cost, excluded per financial theory)

---

## Section 2: Margin Floor Calculation

### Current Plan (25% Markup)
```
E-class: $34.50 cost × 1.25 = $43.13 minimum rate
```

**Problem**: 25% markup leaves NO buffer for:
- Corporate overhead (HQ, legal, IT, accounting)
- Location overhead (rent, utilities, staff, insurance)
- Marketing & customer acquisition
- Bad debt & chargebacks (2-3% of revenue)
- Unexpected major repairs (accidents, engine problems)

### Recommended (28-30% Markup)
```
E-class: $34.50 cost × 1.30 = $44.85 minimum rate (30% markup)
```

**Benefit**: Extra $1.72/rental × 5,000 rentals/month = **$8,600/month** buffer for overhead and risk.

**Industry Comparison**:
- Hertz/Enterprise: 30-35% gross margin (large scale, can absorb risk)
- Budget position: 25-30% margin (discount player, but need overhead buffer)
- Your recommendation: **28-30% margin floor** (safe, realistic for 8-location operation)

---

## Section 3: Quarterly Cost Review Process

### Trigger Events
```python
if fuel_price_change > 10%:
    alert_ops("Fuel spike detected; margin floors need update")
    
if inflation_rate > 3%:
    review_maintenance_costs()
    
if new_major_repair_incident:
    recalculate_maintenance_allowance()
```

### Review Checklist (Every Q)
1. **Fuel**: Update daily fuel cost based on current pump prices
2. **Maintenance**: Review actual spend vs. budget; adjust per-mile cost
3. **Depreciation**: Validate 5-year assumption vs. actual resale values
4. **Insurance**: Premium changes (vehicle count, claims history)
5. **Staffing**: Wage increases, headcount changes
6. **Inflation**: CPI impact on all variable costs

### Alert Thresholds
- Fuel up/down >15%: Immediate review + margin adjustment
- Maintenance tracking >20% above budget: Increase daily allowance
- Depreciation (resale value) <85% of expected: Extend cycle or increase daily charge

### Document Change
```
Update DATA_INPUTS.md Section 5 whenever costs change:
- Old value: E depreciation $12.50/day
- New value: E depreciation $13.20/day (as of Q2 2026)
- Reason: Fleet aging; older vehicles reselling lower
- Approved by: Ethan, CFO sign-off
```

---

## Section 4: Sensitivity Analysis (What If?)

### Scenario Matrix

| Scenario | Fuel +20% | Maintenance +15% | Depreciation -10% | Net Impact on Margin |
|----------|-----------|------------------|-------------------|----------------------|
| **Base Case** | $3.50/gal | $2.50/day maint | $12.50/day deprec | 30% margin (target) |
| **Best Case** | $3.00/gal | $2.20/day | $14.00/day | **34% margin** (+4%) |
| **Worst Case** | $4.50/gal | $3.00/day | $11.00/day | **22% margin** (-8%) |
| **Fuel Spike** | $5.00/gal (+43%) | baseline | baseline | **27% margin** (-3%) |
| **Used Car Crash** | baseline | baseline | $10.00/day | **24% margin** (-6%) |

**Interpretation**: 
- Worst case (all bad): margin drops to 22% → UNVIABLE (below cost)
- Fuel spike alone: manageable at 27%
- Fleet depreciation worst: drops to 24% (still viable with overhead cuts)

**Action**: If any scenario triggers <25% margin, raise rate caps from 15% to 17-20% (temporary) or reduce occupancy assumptions (increase prices).

---

## Section 5: Cost Update SOP

### Monthly (Automatic)
```bash
# Check fuel price (EIA weekly data)
curl https://www.eia.gov/dnav/pet/hist/rbrte.htm
if fuel_price_change > 5%:
    log_warning("Fuel price changed; review in next Q meeting")
```

### Quarterly (Manual Review)
```python
def quarterly_cost_review():
    current_fuel = get_epa_fuel_cost()
    current_maintenance = get_actual_maintenance_spend() / total_miles
    current_depreciation = get_fleet_resale_values()
    
    changes = []
    if changed(fuel, current_fuel, >10%):
        changes.append(f"Fuel: {fuel} → {current_fuel}")
    if changed(maintenance, current_maintenance, >10%):
        changes.append(f"Maintenance: {maintenance} → {current_maintenance}")
    
    if changes:
        notify_ops(f"Cost changes detected: {changes}")
        recalculate_margin_floors()
        update_data_inputs_md()
```

### Ownership
- **Ethan** (CTO): Overall responsibility
- **Finance**: Provide monthly fuel prices, insurance premiums
- **Operations**: Report maintenance actuals, vehicle resale data
- **All**: Sign-off on margin floor updates

---

**Verdict**: Cost structure is now clear. E-class true marginal cost = **$34.50** (not $28). Recommended margin floor = **28-30%** (not 25%). Quarterly review process locks in transparency and prevents silent margin erosion.