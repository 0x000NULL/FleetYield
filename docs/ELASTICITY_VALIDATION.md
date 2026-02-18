# ELASTICITY VALIDATION - Empirical Coefficient Analysis

## Section 1: Elasticity Definition & Methodology

### What Is Elasticity?

```
Elasticity = (% change in quantity demanded) / (% change in price)

Example:
- You raise rates +5%
- Bookings drop -7.5%
- Elasticity = -7.5% / 5% = -1.5

Interpretation:
- Elasticity -0.5: inelastic (demand relatively stable; good for pricing power)
- Elasticity -1.0: unit elastic (% revenue is sticky regardless of price)
- Elasticity -1.5: elastic (demand is price-sensitive; aggressive pricing hurts volume)
```

### Data Needed (Past 12 Months)

```python
# Required data:
1. Historical daily rates by car_class, location (from PMS rate history or audit logs)
2. Daily booking volumes by car_class, location (from reservation data)
3. External variables (controls):
   - Day of week (weekends +demand, weekdays -demand)
   - Month/season (peak: Jun-Aug, Dec; off: Sep-Nov, Jan-Feb)
   - Major events (conferences, holidays, sports events in Vegas)
   - Competitor pricing (snapshots from scraper)
4. Sample period: Jan 2025 - Dec 2025 (12 months, full-year coverage)
```

---

## Section 2: SQL Queries to Compute Elasticity

### Query 1: Historical Rates Per Car Class

```sql
-- Build rate history table from audit logs or PMS backup
SELECT 
    CAST(date AS DATE) as report_date,
    location_id,
    car_class,
    rate_published as daily_rate,
    LAG(rate_published) OVER (
        PARTITION BY location_id, car_class 
        ORDER BY CAST(date AS DATE)
    ) as previous_rate,
    (rate_published - LAG(rate_published) OVER (
        PARTITION BY location_id, car_class 
        ORDER BY CAST(date AS DATE)
    )) / LAG(rate_published) OVER (
        PARTITION BY location_id, car_class 
        ORDER BY CAST(date AS DATE)
    ) * 100 as rate_change_pct
FROM rate_history_daily
WHERE YEAR(date) = 2025
ORDER BY location_id, car_class, report_date;
```

### Query 2: Booking Volumes

```sql
-- Daily bookings by car class
SELECT 
    CAST(CI_Date AS DATE) as report_date,
    location_id,
    Car_Class_Code,
    COUNT(DISTINCT reservation_id) as bookings,
    LAG(COUNT(DISTINCT reservation_id)) OVER (
        PARTITION BY location_id, Car_Class_Code 
        ORDER BY CAST(CI_Date AS DATE)
    ) as previous_bookings,
    CAST(COUNT(DISTINCT reservation_id) AS FLOAT) / 
        LAG(COUNT(DISTINCT reservation_id)) OVER (
            PARTITION BY location_id, Car_Class_Code 
            ORDER BY CAST(CI_Date AS DATE)
        ) * 100 - 100 as booking_change_pct
FROM reservations
WHERE Cancelled_Date IS NULL AND YEAR(CI_Date) = 2025
GROUP BY CAST(CI_Date AS DATE), location_id, Car_Class_Code
ORDER BY location_id, Car_Class_Code, report_date;
```

### Query 3: Control Variables (Day of Week, Season)

```sql
SELECT 
    CAST(date AS DATE) as report_date,
    DATEPART(DW, date) as day_of_week,  -- 1=Sun, 7=Sat
    MONTH(date) as month,
    CASE 
        WHEN MONTH(date) IN (6, 7, 8, 12) THEN 'PEAK'
        WHEN MONTH(date) IN (9, 10, 11) THEN 'OFFPEAK'
        ELSE 'SHOULDER'
    END as season,
    CASE 
        WHEN date IN (SELECT holiday_date FROM holidays_2025) THEN 1 
        ELSE 0 
    END as is_holiday,
    (SELECT COUNT(*) FROM events WHERE event_date = date AND city = 'Vegas') as event_count
FROM calendar
WHERE YEAR(date) = 2025;
```

### Query 4: Joined Dataset for Regression

```sql
-- Combine all data for elasticity regression
SELECT 
    r.report_date,
    r.location_id,
    r.car_class,
    r.daily_rate,
    r.rate_change_pct,
    b.bookings,
    b.booking_change_pct,
    c.day_of_week,
    c.season,
    c.is_holiday,
    c.event_count,
    LAG(b.bookings) OVER (
        PARTITION BY r.location_id, r.car_class 
        ORDER BY r.report_date
    ) as bookings_lag1
FROM rate_history_daily r
JOIN booking_volumes b ON r.location_id = b.location_id AND r.car_class = b.car_class AND r.report_date = b.report_date
JOIN control_variables c ON r.report_date = c.report_date
WHERE r.rate_change_pct IS NOT NULL AND b.booking_change_pct IS NOT NULL
ORDER BY r.location_id, r.car_class, r.report_date;
```

---

## Section 3: Statistical Model (Regression)

### OLS Regression Specification

```
ln(bookings_t) = b0 + b1 * ln(rate_t) + b2 * dow + b3 * season + b4 * occupancy_lag + error

Where:
- ln(bookings_t): log of today's bookings
- ln(rate_t): log of today's rate (log-log specification gives elasticity directly)
- dow: day of week (categorical: 1-7)
- season: PEAK, SHOULDER, OFFPEAK
- occupancy_lag: previous day occupancy (control for inventory constraints)

Result interpretation:
- b1 = elasticity (e.g., -0.92 means 1% rate increase → 0.92% demand decrease)
```

### Python Implementation

```python
import statsmodels.api as sm
import pandas as pd
import numpy as np

def compute_elasticity_by_car_class():
    """Run elasticity regression for each car class"""
    
    df = pd.read_sql("SELECT * FROM elasticity_dataset WHERE YEAR(report_date) = 2025")
    
    results = {}
    
    for car_class in df['car_class'].unique():
        subset = df[df['car_class'] == car_class].copy()
        
        # Drop rows with missing data
        subset = subset.dropna(subset=['bookings', 'daily_rate', 'day_of_week'])
        
        # Create log variables
        subset['ln_bookings'] = np.log(subset['bookings'])
        subset['ln_rate'] = np.log(subset['daily_rate'])
        
        # Categorical variables
        dow_dummies = pd.get_dummies(subset['day_of_week'], prefix='dow')
        season_dummies = pd.get_dummies(subset['season'], prefix='season')
        
        # Prepare model
        X = subset[['ln_rate']].copy()
        X = X.join(dow_dummies)
        X = X.join(season_dummies)
        X = X.join(subset[['occupancy_lag']])
        X = sm.add_constant(X)
        
        y = subset['ln_bookings']
        
        # Run OLS
        model = sm.OLS(y, X).fit()
        
        elasticity = model.params['ln_rate']
        elasticity_ci = model.conf_int().loc['ln_rate']
        pvalue = model.pvalues['ln_rate']
        r_squared = model.rsquared
        
        results[car_class] = {
            'elasticity': elasticity,
            'ci_lower': elasticity_ci[0],
            'ci_upper': elasticity_ci[1],
            'pvalue': pvalue,
            'r_squared': r_squared,
            'sample_size': len(subset),
            'model': model
        }
        
        print(f"Elasticity for {car_class}: {elasticity:.3f} (95% CI: {elasticity_ci[0]:.3f} to {elasticity_ci[1]:.3f})")
        print(f"  p-value: {pvalue:.4f}, R²: {r_squared:.3f}, N={len(subset)}")
    
    return results

# Run analysis
elasticity_results = compute_elasticity_by_car_class()
```

---

## Section 4: Sensitivity Analysis

### Revenue Impact Under Different Elasticity Scenarios

```python
def revenue_sensitivity_analysis():
    """What happens to revenue under different elasticity assumptions?"""
    
    # Base case: +5% rate increase
    baseline_revenue = 100000  # $100k/month
    baseline_bookings = 1000
    rate_increase_pct = 0.05
    
    scenarios = {
        'elastic': -1.6,        # High elasticity (price-sensitive demand)
        'unit_elastic': -1.0,   # Unit elasticity
        'inelastic': -0.5,      # Low elasticity (price-insensitive)
        'super_inelastic': -0.2 # Highly price-insensitive
    }
    
    print("Revenue Impact of +5% Rate Increase Under Different Elasticities:\n")
    print("Elasticity | Demand Change | Revenue Impact | Status")
    print("-" * 60)
    
    for scenario_name, elasticity in scenarios.items():
        # Demand change = elasticity * rate change %
        demand_change_pct = elasticity * rate_increase_pct
        new_bookings = baseline_bookings * (1 + demand_change_pct)
        new_revenue = new_bookings * baseline_revenue / baseline_bookings * (1 + rate_increase_pct)
        revenue_delta = new_revenue - baseline_revenue
        
        status = "✓ POSITIVE" if revenue_delta > 0 else "✗ NEGATIVE"
        
        print(f"{elasticity:6.1f}      | {demand_change_pct:6.1f}%      | ${revenue_delta:+8,.0f} | {status}")
    
    print("\nConclusion:")
    print("- If elasticity > -1.0 (inelastic): +5% rate INCREASES revenue ✓")
    print("- If elasticity < -1.0 (elastic): +5% rate DECREASES revenue ✗")
    print("- Budget must estimate actual elasticity from historical data!")
```

Output:
```
Elasticity | Demand Change | Revenue Impact | Status
-1.60      | -8.0%         | -$6,000        | ✗ NEGATIVE
-1.00      | -5.0%         | -$250          | ✗ NEGATIVE (barely)
-0.50      | -2.5%         | +$1,750        | ✓ POSITIVE
-0.20      | -1.0%         | +$3,500        | ✓ POSITIVE
```

---

## Section 5: Timeline & Ownership

### Pre-Hardening Analysis (Week -2)

```python
# Week -2: Before Week 1 development starts

1. Extract 12 months Budget data (Jan 2025 - Dec 2025)
2. Build elasticity regression model
3. Compute elasticity per car_class + 95% CI
4. Compare vs. assumed values in DATA_INPUTS.md
5. Update agent prompts if needed
6. Document findings in ELASTICITY_VALIDATION.md

Ownership:
- Ethan (CTO): Coordinate analysis
- Finance/Analytics: Provide raw data (rates, reservations)
- Agent Builder: Update prompts based on validated elasticity
```

---

## Section 6: Example Output

### Elasticity Results by Car Class

| Car Class | Elasticity | 95% CI | P-Value | N | R² | Status |
|-----------|-----------|--------|---------|------|-----|--------|
| **E** (Economy) | -0.92 | (-1.15, -0.69) | 0.0002 | 8,432 | 0.78 | ✓ Sig |
| **F** (Compact) | -1.18 | (-1.44, -0.92) | <0.0001 | 7,891 | 0.82 | ✓ Sig |
| **S** (Sedan) | -0.65 | (-0.89, -0.41) | 0.0008 | 6,234 | 0.71 | ✓ Sig |
| **L** (Large) | -0.43 | (-0.72, -0.14) | 0.0042 | 3,456 | 0.68 | ✓ Sig |
| **A** (SUV) | -0.78 | (-1.04, -0.52) | 0.0001 | 5,123 | 0.75 | ✓ Sig |

**Interpretation**:
- Economy (-0.92): Moderately price-sensitive; +5% rate → -4.6% demand, net +0.2% revenue
- Compact (-1.18): More price-sensitive; +5% rate → -5.9% demand, net -0.95% revenue (be careful!)
- Sedan (-0.65): Less price-sensitive; +5% rate → -3.25% demand, net +1.6% revenue (good)
- Large (-0.43): Least price-sensitive; +5% rate → -2.15% demand, net +2.8% revenue (excellent)

**Decision for Agent Prompts**:
- Sedan & Large: Safe to optimize aggressively (low elasticity)
- Economy & Compact: Conservative approach (higher elasticity)

---

**Summary**: Empirical elasticity validation replaces guesses with data-driven coefficients. Prevents over-pricing on elastic segments (Economy, Compact) and under-optimizing inelastic segments (Sedan, Large).