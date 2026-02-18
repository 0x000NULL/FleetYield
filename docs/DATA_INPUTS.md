# Complete Data Inputs for AI Rate Optimization Agents

This document specifies all data that should be fed to the FleetYield agents to maximize decision quality and revenue impact.

---

## Phase 1 (MVP, Weeks 1-4) — Essential Inputs

### 1. Nightly Reservation Data
**Source**: PMS (Wizard/WAND)  
**Schedule**: 8:00 PM PST  
**Purpose**: Measure demand, identify trends, segment by channel

```sql
SELECT 
  CI Date as checkout_date,
  Car Class Code as car_class,
  Booking Source as channel,
  COUNT(*) as bookings,
  SUM(Est Total Amt) as total_revenue,
  COUNT(CASE WHEN Cancelled Date IS NULL THEN 1 END) as active_reservations,
  COUNT(CASE WHEN Cancelled Date IS NOT NULL THEN 1 END) as cancellations,
  AVG(Res Adv Days) as avg_lead_days
FROM reservations
WHERE CI Date >= CURRENT_DATE - 7  -- Last 7 days only
GROUP BY CI Date, Car Class Code, Booking Source
```

**Data Schema**:
```json
{
  "reservation_cache": [
    {
      "date": "2026-02-18",
      "car_class": "E",
      "booking_source": "costco",
      "bookings": 35,
      "revenue": 1260,
      "active": 34,
      "cancellations": 1,
      "avg_lead_days": 2
    }
  ]
}
```

---

### 2. Competitor Price Scraping
**Source**: Hertz, Enterprise, Avis, National, Alamo, Budget.com  
**Schedule**: 8-11 PM PST (staggered, 1 company/hour)  
**Purpose**: Understand competitive positioning, detect market shifts

```json
{
  "competitor_rates": [
    {
      "date": "2026-02-18",
      "location": "LAS_AIRPORT",
      "car_class": "E",
      "competitor": "hertz",
      "daily_rate": 45.99,
      "weekly_rate": 38.50,
      "timestamp": "2026-02-18T22:47:00Z"
    }
  ]
}
```

**Tech**: Playwright + Celery on 3-5 DigitalOcean VMs ($5/mo each)

---

### 3. Current Rates
**Source**: Your PMS  
**Schedule**: Fresh load at 5:55 AM  
**Purpose**: Baseline for rate change decisions

```json
{
  "current_rates": {
    "LAS_AIRPORT": {
      "E": { "daily": 42.00, "weekly": 35.00 },
      "F": { "daily": 48.00, "weekly": 40.00 },
      "S": { "daily": 55.00, "weekly": 46.00 }
    }
  }
}
```

---

### 4. Fleet Inventory
**Source**: GLPI or PMS  
**Schedule**: Daily at 5:55 AM  
**Purpose**: Understand availability, trigger surge pricing

```json
{
  "inventory": {
    "LAS_AIRPORT": {
      "E": { "available": 15, "reserved": 42, "capacity": 60, "utilization_pct": 70 },
      "F": { "available": 8, "reserved": 28, "capacity": 40, "utilization_pct": 70 }
    }
  }
}
```

---

### 5. Cost Structure
**Source**: Accounting data (specify once)  
**Schedule**: Annual review  
**Purpose**: Set margin floors, prevent unprofitable pricing

```json
{
  "economics": {
    "daily_variable_cost": {
      "E": 7.00,
      "F": 8.20,
      "S": 9.30,
      "L": 10.50
    },
    "daily_depreciation": {
      "E": 12.50,
      "F": 14.00,
      "S": 16.00,
      "L": 18.00
    },
    "fixed_cost_per_rental": 15.00,
    "true_marginal_cost": {
      "E": 34.50,
      "F": 37.20,
      "S": 40.30,
      "L": 43.50
    },
    "minimum_rates": {
      "E": 45.93,
      "F": 49.60,
      "S": 53.73,
      "L": 58.00
    }
  }
}
```

---

## Phase 1+ (Weeks 5+) — High-Impact Additions

### 6. Day-of-Week Demand Multipliers
**Source**: Historical Feb 2026 analysis (extract from your CSV)  
**Schedule**: Monthly update  
**Purpose**: Adjust rates by day-of-week (Friday/Saturday surge, Mon-Tue slump)

**Extract Query**:
```sql
SELECT 
  DAYNAME(CI Date) as day_of_week,
  COUNT(*) as bookings,
  AVG(Est Total Amt) as avg_revenue,
  ROUND(AVG(Est Total Amt) / (SELECT AVG(Est Total Amt) FROM reservations WHERE CI Date >= '2026-02-01'), 2) as multiplier
FROM reservations
WHERE CI Date >= '2026-02-01' AND CI Date < '2026-03-01' AND Cancelled Date IS NULL
GROUP BY DAYNAME(CI Date);
```

**Sample Output**:
```json
{
  "dow_multipliers": {
    "Monday": 0.88,
    "Tuesday": 0.90,
    "Wednesday": 0.92,
    "Thursday": 1.05,
    "Friday": 1.35,
    "Saturday": 1.42,
    "Sunday": 1.28
  },
  "note": "Weekend surge (Fri +35%, Sat +42%). Weekday slump (Mon -12%, Tue -10%)."
}
```

**Agent Application**: 
- If Friday/Saturday AND high demand AND >70% inventory → raise rates 15-25%
- If Monday/Tuesday AND low demand AND <60% inventory → lower rates 5-10%

**Impact**: +2-3% revenue lift just from day-of-week tuning

---

### 7. Booking Lead Time Curves
**Source**: Historical reservation analysis (Res Adv Days)  
**Schedule**: Weekly update  
**Purpose**: Price based on booking certainty (15+ day = committed, willing to pay)

**Extract Query**:
```sql
SELECT 
  CASE 
    WHEN Res Adv Days < 1 THEN '<1d'
    WHEN Res Adv Days BETWEEN 1 AND 3 THEN '1-3d'
    WHEN Res Adv Days BETWEEN 4 AND 7 THEN '4-7d'
    WHEN Res Adv Days BETWEEN 8 AND 14 THEN '8-14d'
    ELSE '15+d'
  END as lead_time_bucket,
  COUNT(*) as bookings,
  ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM reservations WHERE Cancelled Date IS NULL), 2) as volume_pct,
  AVG(Est Total Amt) as avg_revenue,
  ROUND(AVG(Est Total Amt) / (SELECT AVG(Est Total Amt) FROM reservations WHERE Res Adv Days BETWEEN 4 AND 7), 2) as multiplier,
  AVG(CASE WHEN Cancelled Date IS NOT NULL THEN 1 ELSE 0 END) as cancellation_rate
FROM reservations
WHERE Cancelled Date IS NULL
GROUP BY lead_time_bucket;
```

**Sample Output**:
```json
{
  "lead_time_impact": {
    "<1d": {
      "multiplier": 0.70,
      "volume": 0.12,
      "cancellation_rate": 0.18,
      "note": "Last-minute, desperate bookers, price-sensitive, high cancel rate"
    },
    "1-3d": {
      "multiplier": 0.85,
      "volume": 0.20,
      "cancellation_rate": 0.12,
      "note": "Short-notice, some flexibility"
    },
    "4-7d": {
      "multiplier": 1.0,
      "volume": 0.25,
      "cancellation_rate": 0.08,
      "note": "Baseline / standard booking"
    },
    "8-14d": {
      "multiplier": 1.15,
      "volume": 0.25,
      "cancellation_rate": 0.05,
      "note": "Advance planners, committed customers, low cancel"
    },
    "15+d": {
      "multiplier": 1.25,
      "volume": 0.18,
      "cancellation_rate": 0.02,
      "note": "Fully planned trips, locked in, willing to pay premium"
    }
  }
}
```

**Agent Application**:
- If booking is 15+ days advance AND demand medium/high → rate +15-25%
- If booking is <1d AND inventory high → rate -15% to fill the car

**Impact**: +3-4% revenue lift

---

### 8. Channel-Specific Economics
**Source**: Your booking history + financial data  
**Schedule**: Monthly review  
**Purpose**: Different rates per channel (GDS premium, Costco discount)

**Extract Query**:
```sql
SELECT 
  Booking Source as channel,
  COUNT(*) as bookings,
  ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM reservations), 2) as volume_pct,
  AVG(Est Total Amt) as avg_rate,
  AVG(CASE WHEN Cancelled Date IS NOT NULL THEN 1 ELSE 0 END) as cancellation_rate,
  AVG(Res Adv Days) as avg_lead_days,
  MIN(Est Total Amt) as min_rate,
  MAX(Est Total Amt) as max_rate
FROM reservations
WHERE CI Date >= '2026-02-01' AND CI Date < '2026-03-01'
GROUP BY Booking Source
ORDER BY volume_pct DESC;
```

**Sample Output**:
```json
{
  "channel_intelligence": {
    "costco": {
      "volume_pct": 0.35,
      "avg_rate": 46.50,
      "price_sensitivity": "high",
      "typical_discount_vs_median": 0.15,
      "cancellation_rate": 0.12,
      "margin_floor": 0.18,
      "avg_lead_days": 1,
      "repeat_customer_rate": 0.40,
      "strategy": "Volume player—discount aggressively to keep bookings"
    },
    "priceline": {
      "volume_pct": 0.25,
      "avg_rate": 42.00,
      "price_sensitivity": "very_high",
      "typical_discount_vs_median": 0.20,
      "cancellation_rate": 0.14,
      "margin_floor": 0.15,
      "avg_lead_days": 2,
      "repeat_customer_rate": 0.08,
      "strategy": "OTA shoppers are price-hunters—be competitive but don't sacrifice margin"
    },
    "gds": {
      "volume_pct": 0.15,
      "avg_rate": 58.00,
      "price_sensitivity": "low",
      "typical_discount_vs_median": 0.0,
      "cancellation_rate": 0.05,
      "margin_floor": 0.30,
      "avg_lead_days": 14,
      "repeat_customer_rate": 0.70,
      "strategy": "Corporate customers; command premium, lock in long-term loyalty"
    },
    "budget_com": {
      "volume_pct": 0.15,
      "avg_rate": 51.00,
      "price_sensitivity": "medium",
      "typical_discount_vs_median": 0.08,
      "cancellation_rate": 0.08,
      "margin_floor": 0.25,
      "avg_lead_days": 7,
      "repeat_customer_rate": 0.35,
      "strategy": "Your direct channel—no middleman, but you control pricing"
    }
  }
}
```

**Agent Application**:
- GDS booking + low cancel rate → allow rates 20-30% above baseline (they pay)
- Costco booking + high demand → rate can hold steady (volume is locked)
- Priceline booking + low inventory → rate competitive but not desperate
- Budget.com booking → optimize for your margin, not competitor parity

**Impact**: +2-3% revenue lift

---

### 9. Price Elasticity by Car Class
**Source**: Historical analysis or estimated  
**Schedule**: Quarterly review  
**Purpose**: Adjust scoring by class (economy is price-sensitive; sedan less so)

**Sample Data**:
```json
{
  "elasticity": {
    "E": {
      "label": "Economy (highly price-sensitive)",
      "elasticity_coefficient": -1.6,
      "pricing_insight": "5% rate increase → 8% demand drop. Protect volume.",
      "agent_instruction": "Higher demand_weight (50%), lower competitive_weight (20%)"
    },
    "F": {
      "label": "Mid-size (moderate elasticity)",
      "elasticity_coefficient": -1.0,
      "pricing_insight": "5% rate increase → 5% demand drop. Balanced.",
      "agent_instruction": "Standard weighting (40% demand, 35% inventory, 25% competitive)"
    },
    "S": {
      "label": "Sedan (less elastic, business segment)",
      "elasticity_coefficient": -0.4,
      "pricing_insight": "5% rate increase → 2% demand drop. Can raise rates.",
      "agent_instruction": "Lower demand_weight (30%), higher competitive_weight (35%)"
    },
    "L": {
      "label": "Large/Premium (low elasticity, niche)",
      "elasticity_coefficient": -0.2,
      "pricing_insight": "5% rate increase → 1% demand drop. Premium pricing works.",
      "agent_instruction": "Aggressive competitive positioning (40%), margin-focused (30%)"
    }
  }
}
```

**Agent Application**: Different scoring weights per car class (don't use one-size-fits-all)

**Impact**: +1-2% revenue lift

---

### 10. Competitor Strength & Market Profile
**Source**: Manual configuration + market research  
**Schedule**: Quarterly review  
**Purpose**: Know which competitors to match, which to beat, which to ignore

```json
{
  "competitors": {
    "hertz": {
      "market_share": 0.28,
      "brand_strength": 0.90,
      "threat_level": "critical",
      "primary_segments": ["airport", "premium"],
      "rule": "If Hertz < your rate by >5%, raise immediately. Protect premium positioning."
    },
    "enterprise": {
      "market_share": 0.25,
      "brand_strength": 0.75,
      "threat_level": "high",
      "primary_segments": ["all"],
      "rule": "Stay within 5% of Enterprise median. They're everywhere."
    },
    "avis": {
      "market_share": 0.18,
      "brand_strength": 0.70,
      "threat_level": "medium",
      "primary_segments": ["business"],
      "rule": "Match but don't obsess. Watch GDS channel."
    },
    "national": {
      "market_share": 0.15,
      "brand_strength": 0.65,
      "threat_level": "medium",
      "primary_segments": ["leisure", "tour"],
      "rule": "Can be 3-5% above National. They're value-focused."
    },
    "alamo": {
      "market_share": 0.10,
      "brand_strength": 0.60,
      "threat_level": "low",
      "primary_segments": ["discount-seekers"],
      "rule": "Ignore for premium; match for volume."
    },
    "budget_com": {
      "market_share": 0.04,
      "brand_strength": 1.0,
      "threat_level": "you",
      "rule": "You control this. Price for profit, not parity."
    }
  },
  "consolidation_rule": "Compute median of Hertz + Enterprise + Avis (the 'big 3'). Use as anchor."
}
```

**Agent Application**: 
- When evaluating competitive_score, weight big 3 (Hertz, Enterprise, Avis) more heavily
- Ignore Alamo unless you're in discount war
- Never match Budget.com competitors on your own direct channel

**Impact**: Prevents margin leakage (-1-2% revenue if ignored)

---

### 11. Location Market Context
**Source**: Manual configuration based on location type  
**Schedule**: Quarterly review  
**Purpose**: Understand location-specific dynamics, adjust aggressiveness

```json
{
  "locations": {
    "LAS_AIRPORT": {
      "location_type": "airport_hub",
      "primary_competitor": "hertz",
      "oligopoly_concentration": 0.78,
      "customer_comparison_intensity": "very_high",
      "pricing_strategy": "competitive_parity",
      "rate_variance_tolerance": 0.05,
      "notes": "All 6 major brands here. Customers compare aggressively. Stay within 5% of median."
    },
    "CENTER_STRIP": {
      "location_type": "resort_strip",
      "primary_competitor": "national",
      "oligopoly_concentration": 0.82,
      "customer_comparison_intensity": "very_high",
      "pricing_strategy": "competitive_parity_strict",
      "rate_variance_tolerance": 0.04,
      "notes": "Tourist customers use OTAs heavily. Stay close to median."
    },
    "HENDERSON_EXECUTIVE": {
      "location_type": "business_airport",
      "primary_competitor": "enterprise",
      "oligopoly_concentration": 0.65,
      "customer_comparison_intensity": "medium",
      "pricing_strategy": "moderate_premium",
      "rate_variance_tolerance": 0.15,
      "notes": "Corporate/charter customers. Less price-shopping. Room for premium (10-15% above median OK)."
    },
    "GIBSON": {
      "location_type": "off_strip_neighborhood",
      "primary_competitor": "enterprise",
      "oligopoly_concentration": 0.55,
      "customer_comparison_intensity": "low",
      "pricing_strategy": "aggressive_premium",
      "rate_variance_tolerance": 0.20,
      "notes": "Off-strip location; customers less price-sensitive. Room for +15-20% premium."
    },
    "W_SAHARA": {
      "location_type": "neighborhood",
      "primary_competitor": "enterprise",
      "oligopoly_concentration": 0.60,
      "customer_comparison_intensity": "medium",
      "pricing_strategy": "moderate_premium",
      "rate_variance_tolerance": 0.15,
      "notes": "Local rentals; less price-shopping. Room to differentiate (+10% OK)."
    },
    "GOLDEN_NUGGET": {
      "location_type": "resort_downtown",
      "primary_competitor": "national",
      "oligopoly_concentration": 0.70,
      "customer_comparison_intensity": "high",
      "pricing_strategy": "competitive_with_upside",
      "rate_variance_tolerance": 0.08,
      "notes": "Convention/downtown. Competitive but room for +5% premium if demand spikes."
    },
    "TROPICANA": {
      "location_type": "resort_south_strip",
      "primary_competitor": "enterprise",
      "oligopoly_concentration": 0.75,
      "customer_comparison_intensity": "high",
      "pricing_strategy": "competitive_parity",
      "rate_variance_tolerance": 0.06,
      "notes": "South Strip tourists. Stay competitive."
    },
    "LOSEE": {
      "location_type": "north_neighborhood",
      "primary_competitor": "enterprise",
      "oligopoly_concentration": 0.50,
      "customer_comparison_intensity": "very_low",
      "pricing_strategy": "aggressive_premium",
      "rate_variance_tolerance": 0.22,
      "notes": "Least competitive location. Room for +20-25% premium."
    }
  }
}
```

**Agent Application**: Location-specific rate aggressiveness
- Airport/Strip → conservative (within 5% of median)
- Henderson/Gibson/Losee → aggressive (15-25% premium room)

**Impact**: +1-2% revenue from location optimization

---

## Phase 2 (Weeks 7+) — Nice-to-Have Enhancements

### 12. Real-Time Booking Velocity
**Source**: PMS (hourly or daily)  
**Schedule**: Daily at 5:55 AM  
**Purpose**: Detect surge demand in real-time

```json
{
  "booking_velocity": {
    "LAS_AIRPORT": {
      "E": {
        "bookings_last_24h": 45,
        "bookings_7d_avg": 32,
        "velocity_multiplier": 1.41,
        "interpretation": "+41% above baseline; demand surging → raise rates"
      }
    }
  }
}
```

**Agent Application**: If velocity >1.3x baseline + inventory >70% → raise rates 10-20%

**Impact**: +1-2% revenue from real-time surge capture

---

### 13. Customer Lifetime Value & Repeat Impact
**Source**: Loyalty/repeat booking analysis  
**Schedule**: Quarterly (Phase 2 only)  
**Purpose**: Balance short-term margin vs. long-term loyalty

```json
{
  "ltv_by_channel": {
    "gds": {
      "repeat_rate": 0.70,
      "avg_lifetime_bookings": 12,
      "lifetime_value": 2100,
      "note": "High-value repeats. Lower initial rate to lock loyalty."
    },
    "budget_com": {
      "repeat_rate": 0.35,
      "avg_lifetime_bookings": 4,
      "lifetime_value": 750,
      "note": "Medium repeats. Price aggressively."
    },
    "priceline": {
      "repeat_rate": 0.08,
      "avg_lifetime_bookings": 1.2,
      "lifetime_value": 50,
      "note": "No loyalty. Price for one-off margin."
    }
  }
}
```

**Agent Application** (Phase 2): GDS rate -5% for loyalty premium. Priceline rate +10% for one-off margin.

---

### 14. External Events & Demand Calendar
**Source**: Vegas event APIs, sports calendars, convention schedules  
**Schedule**: Weekly update  
**Purpose**: Predictive demand surges (CES, NAB, Super Bowl, etc.)

```json
{
  "events_2026": {
    "2026-01-06_to_2026-01-10": {
      "event": "CES",
      "demand_multiplier": 2.5,
      "availability": "critical (book out)",
      "rate_guidance": "Max out rates, limit discounts"
    },
    "2026-03-24_to_2026-03-27": {
      "event": "NCAA Tournament (Vegas hosts)",
      "demand_multiplier": 1.8,
      "availability": "tight",
      "rate_guidance": "Premium pricing (+25%)"
    }
  }
}
```

**Agent Application**: If event week detected, multiply all rate recommendations by event multiplier.

---

## Summary: Data Priority & Quick-Win Sequence

| # | Data Source | Difficulty | Time to Implement | Revenue Impact | Recommended Week |
|---|-------------|------------|-------------------|-----------------|-----------------|
| 1 | Reservation CSV (7-day) | Easy | Already building | Baseline | Week 1-2 |
| 2 | Competitor scrape | Medium | Week 2 | Baseline | Week 2 |
| 3 | Current rates | Easy | Week 2 | Baseline | Week 2 |
| 4 | Inventory | Easy | Week 2 | +1% (surge detection) | Week 2 |
| 5 | Cost structure | Easy (one-time) | 1 hour | +0% (prevents loss) | Week 1 |
| **6** | **Day-of-week multipliers** | **Easy (analysis)** | **1 hour** | **+2-3%** | **Week 3** |
| **7** | **Lead time curves** | **Easy (analysis)** | **1.5 hours** | **+3-4%** | **Week 3** |
| **8** | **Channel economics** | **Medium (analysis)** | **2 hours** | **+2-3%** | **Week 4** |
| 9 | Elasticity by class | Medium (optional) | Week 4 | +1-2% | Week 4 |
| 10 | Competitor profiles | Easy (manual) | 1 hour | -1-2% (prevent loss) | Week 3 |
| 11 | Location context | Easy (manual) | 2 hours | +1-2% | Week 3 |
| 12 | Booking velocity | Medium | Week 5 | +1-2% | Week 6 |
| 13 | LTV/repeat impact | Hard | Phase 2 | +2-3% | Week 8+ |
| 14 | Events calendar | Medium | Phase 2 | +5-10% (if active) | Week 7+ |

---

## Implementation Checklist

- [ ] Week 1-2: Build data pipeline (1-5 above)
- [ ] Week 3: Add day-of-week + lead time + location context (6, 7, 11)
- [ ] Week 4: Add channel economics + elasticity (8, 9)
- [ ] Week 5-6: Add booking velocity + real-time tuning (12)
- [ ] Week 7+: Add events + LTV optimization (13, 14)

**Expected Revenue Impact**:
- Week 1-2: Baseline optimization (+3-4% vs. no system)
- Week 3: +1-2% more (+4-6% total)
- Week 4: +1-2% more (+5-8% total)
- Week 7+: +2-5% more (+7-13% total)

All figures are conservative. Real-world results may be higher with proper agent tuning.
