# OCCUPANCY MONITORING - Revenue-Occupancy Tradeoff Analysis

## Section 1: Occupancy KPI Definition

### Daily Occupancy Metric

```sql
-- Calculate daily occupancy percentage
SELECT 
    location_id,
    CAST(reservation_date AS DATE) as date,
    COUNT(DISTINCT unit_id) as reserved_units,
    (SELECT MAX(total_capacity) FROM location_capacity WHERE location_id = lc.location_id) as total_capacity,
    CAST(COUNT(DISTINCT unit_id) AS FLOAT) / 
        (SELECT MAX(total_capacity) FROM location_capacity WHERE location_id = lc.location_id) * 100 
        as occupancy_pct,
    COUNT(DISTINCT booking_id) as bookings,
    SUM(Est_Total_Amt) as total_revenue
FROM reservations r
JOIN location_capacity lc ON r.location_id = lc.location_id
WHERE reservation_date >= DATEADD(day, -30, CAST(GETDATE() AS DATE))
GROUP BY location_id, CAST(reservation_date AS DATE)
ORDER BY date DESC;
```

### KPI Tiers

| Occupancy Tier | Range | Pricing Strategy | Risk Level |
|---|---|---|---|
| Low | <60% | Aggressive discount (-10%) | Low margin pressure |
| Normal | 60-80% | Neutral (baseline) | Healthy |
| High | 80-90% | Premium (+5-10%) | Risk: demand elasticity |
| Critical | >90% | Max premium (+15%) | Risk: customer satisfaction |

### Baseline Calculation (Pre-FleetYield)

```python
def calculate_occupancy_baseline():
    """30-day pre-FleetYield average occupancy per location"""
    
    query = """
    SELECT 
        location_id,
        AVG(occupancy_pct) as baseline_occupancy,
        STDEV(occupancy_pct) as occupancy_volatility
    FROM daily_occupancy
    WHERE date BETWEEN DATEADD(day, -60, GETDATE()) AND DATEADD(day, -30, GETDATE())
    GROUP BY location_id
    """
    
    return db.execute(query)
```

---

## Section 2: SQL Queries for Occupancy

### Query 1: Daily Occupancy Per Location

```sql
SELECT 
    r.location_id,
    CAST(r.CI_Date AS DATE) as reservation_date,
    COUNT(DISTINCT r.unit_id) as units_reserved,
    lc.total_capacity,
    CAST(COUNT(DISTINCT r.unit_id) AS FLOAT) / lc.total_capacity * 100 as occupancy_pct,
    LAG(CAST(COUNT(DISTINCT r.unit_id) AS FLOAT) / lc.total_capacity * 100) 
        OVER (PARTITION BY r.location_id ORDER BY CAST(r.CI_Date AS DATE)) as occupancy_pct_prev_day
FROM reservations r
JOIN location_capacity lc ON r.location_id = lc.location_id
WHERE r.Cancelled_Date IS NULL
GROUP BY r.location_id, CAST(r.CI_Date AS DATE), lc.total_capacity
ORDER BY r.location_id, CAST(r.CI_Date AS DATE) DESC;
```

---

## Section 3: Occupancy Monitoring Logic (Cron)

```python
def monitor_occupancy_daily():
    """Daily occupancy check with alerts"""
    
    locations = get_all_locations()
    
    for location_id in locations:
        today_occ = get_occupancy(location_id, datetime.now())
        yesterday_occ = get_occupancy(location_id, datetime.now() - timedelta(days=1))
        baseline_occ = get_baseline_occupancy(location_id)
        
        occ_delta = today_occ - yesterday_occ
        rate_change_pct = get_avg_rate_change(location_id)
        
        # Alert if occupancy drops >5% when rate increased
        if occ_delta < -5 and rate_change_pct > 3:
            slack_alert(f"⚠️ Occupancy drop {occ_delta:.1f}% after {rate_change_pct:.1f}% rate increase")
            log_occupancy_alert(location_id, "HIGH", occ_delta, rate_change_pct)
        
        # Critical: occupancy drops >10%
        elif occ_delta < -10:
            slack_alert(f"🚨 CRITICAL: Occupancy drop {occ_delta:.1f}%")
            
            # Consider auto-revert
            if should_auto_revert(location_id):
                execute_revert(location_id, mode="7day_avg")
```

---

## Section 4: Revenue-Occupancy Breakeven Analysis

```python
def calculate_net_profit(location_id, date):
    """True net profit = revenue gain - occupancy loss"""
    
    today_rev = get_revenue(location_id, date)
    yesterday_rev = get_revenue(location_id, date - timedelta(days=1))
    rev_lift = (today_rev - yesterday_rev) / yesterday_rev * 100
    
    today_occ = get_occupancy(location_id, date)
    yesterday_occ = get_occupancy(location_id, date - timedelta(days=1))
    occ_delta = today_occ - yesterday_occ
    
    # Units lost from occupancy drop
    total_capacity = get_location_capacity(location_id)
    units_lost = total_capacity * abs(occ_delta) / 100
    
    # Average rate
    avg_rate = today_rev / get_units_reserved(location_id, date)
    
    # Profit impact
    rev_gain = rev_lift / 100 * yesterday_rev
    profit_loss = units_lost * avg_rate
    net = rev_gain - profit_loss
    
    return {
        "revenue_gain": rev_gain,
        "profit_loss": profit_loss,
        "net_profit": net,
        "positive": net > 0
    }

# Example: LAS_AIRPORT
# +5% rate → +$250 revenue
# -3% occupancy → -2.4 units → -$120 cost
# NET: +$130 ✓ POSITIVE
```

---

**Summary**: Real-time occupancy monitoring with alerts on drops >5%, correlation analysis with rate changes, and true net profit accounting (revenue minus occupancy loss). Critical visibility into elasticity.