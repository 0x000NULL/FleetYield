# AI Agent Orchestration Platform: Dynamic Rate Management
**Budget Rent a Car Las Vegas (Malco Enterprises)**

---

## EXECUTIVE SUMMARY

A **hybrid multi-agent system** that:
- Runs **daily cron-based rate optimization** (6 AM PST before business opens)
- Deploys **consensus-driven agents** (master + 8 location agents + shared data agents)
- Maintains **complete rate history** for auditability and rollback
- Uses **web scraping + nightly reservation data** for competitive intelligence
- Focuses on **revenue optimization** per booking

**Phase 1 (MVP)**: 8-week build for one location (LAS Airport), then scale 2x/week.

---

## 1. ARCHITECTURE OVERVIEW

### 1.1 Agent Hierarchy (Hybrid Model)

```
┌─────────────────────────────────────────────────────────────┐
│ MASTER ORCHESTRATOR AGENT (Opus)                            │
│ ├─ Reads nightly reservation dump + competitor data         │
│ ├─ Dispatches tasks to location agents                      │
│ ├─ Collects consensus votes                                 │
│ ├─ Enforces global constraints & revenue targets            │
│ └─ Persists rate decisions to history DB                    │
└─────────────────────────────────────────────────────────────┘
         │                    │
    ┌────┴────┬───────┬──────┴─────┬──────────┐
    │          │       │            │          │
   ┌──────────────────────────────────────────────────────┐
   │ LOCATION AGENTS (Opus × 8) - one per site           │
   │ - LAS Airport                                         │
   │ - Center Strip                                        │
   │ - W Sahara                                            │
   │ - Golden Nugget                                       │
   │ - Tropicana                                           │
   │ - Gibson                                              │
   │ - Henderson Executive                                 │
   │ - Losee                                               │
   │                                                       │
   │ Each votes on: demand score, inventory pressure,      │
   │ competitive positioning, margin targets               │
   └──────────────────────────────────────────────────────┘
    │         │        │          │           │
   ┌──────────────────────────────────────────────────────┐
   │ SHARED DATA AGENTS (Haiku × 3 - cheaper)             │
   │ ├─ Competitor Scraper Agent (runs 8 PM nightly)      │
   │ ├─ Reservation Parser Agent (reads CSV, aggregates)  │
   │ └─ Event Calendar Agent (syncs external calendar)    │
   └──────────────────────────────────────────────────────┘
    │
   ┌──────────────────────────────────────────────────────┐
   │ PERSISTENT STATE                                      │
   │ ├─ Rate History DB (SQLite: location, date, rate)    │
   │ ├─ Competitor Snapshot (latest prices by brand)      │
   │ ├─ Reservation Cache (daily agg: bookings/fleet)     │
   │ └─ Decision Log (audit trail: agent votes & ratios)  │
   └──────────────────────────────────────────────────────┘
```

### 1.2 Data Flow (Daily Cron @ 6:00 AM PST)

```
1. [5:45 AM] Reservation Parser reads last 24h CSV export
   └─> Counts: total_bookings, by_car_class, by_channel, cancellations
   
2. [5:50 AM] Competitor Scraper finishes nightly scrape
   └─> Stores: Hertz, Enterprise, Avis rates for each location
   
3. [6:00 AM] Master Orchestrator wakes up
   ├─> Load: reservation snapshot, competitor prices, current rates
   ├─> Dispatch to 8 location agents (parallel)
   │   └─> Each agent analyzes: demand, inventory, competition
   │       └─> Returns: rate_recommendation + confidence + reasoning
   │
   ├─> Collect all votes (expect 8 per car class)
   ├─> Compute consensus (median or weighted average)
   ├─> Apply global constraints (margin floors, ceiling rules)
   ├─> Write new rates to system + history DB
   └─> Log all decisions (for audit trail)
   
4. [6:15 AM] Rates live in PMS/booking system
   └─> Customers see new prices for new reservations
```

---

## 2. DATA PIPELINE

### 2.1 Inputs (What Agents Need)

#### A. Nightly Reservation Dump
- **Source**: Your CSV export (already using this format)
- **Schedule**: Nightly at 8 PM (before scraping, before agent run)
- **Processing**:
  ```sql
  SELECT 
    ci_date,
    car_class_code,
    booking_source,
    count(*) as bookings,
    sum(est_total_amt) as revenue,
    count(case when cancelled_date is not null then 1 end) as cancellations
  FROM reservations
  WHERE ci_date >= CURRENT_DATE - 7 -- last 7 days
  GROUP BY ci_date, car_class_code, booking_source
  ```
- **Car Classes in Your Data**: E (compact), F (mid), S (sedan), L (large), A (van), W (wagon), etc.
- **Booking Sources**: Costco, Priceline, Budget.com, GDS, phone, etc.

#### B. Competitor Web Scrape
- **Competitors**: Hertz, Enterprise, Avis, National, Alamo, Budget.com direct
- **Schedule**: Nightly 8-11 PM (staggered, rate-limit friendly)
- **Data Points per Competitor/Location/CarClass**:
  ```json
  {
    "date": "2026-02-18",
    "location": "LAS_AIRPORT",
    "car_class": "E",
    "competitor": "hertz",
    "daily_rate": 45.99,
    "weekly_rate": 38.50,
    "timestamp": "2026-02-18T22:47:00Z"
  }
  ```
- **Scraping Tech**: Playwright + Celery workers (3-5 DigitalOcean $5/mo VMs recommended to avoid rate limits)
- **Storage**: SQLite `competitor_rates` table with 7-day rolling window

#### C. Current Rates
- **Source**: Your PMS (Wizard/WAND or API export)
- **Schedule**: Loaded fresh at 6 AM before agent run
- **Structure**:
  ```json
  {
    "location": "LAS_AIRPORT",
    "car_class": "E",
    "daily_rate": 42.00,
    "weekly_rate": 35.00,
    "last_updated": "2026-02-17T06:00:00Z"
  }
  ```

#### D. Fleet Inventory (Optional but Important)
- **Source**: Your GLPI or PMS
- **Schedule**: Hourly snapshots (or nightly aggregate)
- **Data**:
  ```json
  {
    "location": "LAS_AIRPORT",
    "car_class": "E",
    "available": 15,
    "reserved": 42,
    "capacity": 60,
    "utilization_pct": 95
  }
  ```

#### E. External Events (Optional, Phase 2)
- **Source**: Vegas event calendars (CES dates, conventions, sports schedules)
- **Schedule**: Weekly sync
- **Impact**: "CES week (Jan 6-10) ➜ high demand multiplier"

### 2.2 Agent Inputs at 6 AM (Phase 1 Baseline)

**Master passes to each Location Agent**:
```json
{
  "location": "LAS_AIRPORT",
  "date": "2026-02-18",
  "day_of_week": "Friday",
  "reservation_data": {
    "bookings_last_7d": { "E": 120, "F": 85, "S": 40 },
    "revenue_last_7d": { "E": 4800, "F": 3570, "S": 1800 },
    "cancellation_rate": 0.08,
    "booking_sources": { "costco": 0.35, "priceline": 0.25, "budget_com": 0.20, "gds": 0.15, "other": 0.05 },
    "bookings_last_24h": { "E": 45, "F": 32, "S": 18 },
    "velocity_multiplier": 1.41
  },
  "competitor_snapshot": {
    "E": { "hertz": 49.99, "enterprise": 44.99, "avis": 47.99, "national": 46.00, "alamo": 44.00, "budget_com": 42.00 },
    "F": { "hertz": 55.99, "enterprise": 51.99, "avis": 53.00, "national": 52.00, "alamo": 50.00, "budget_com": 48.00 }
  },
  "current_rates": {
    "E": 42.00,
    "F": 48.00,
    "S": 55.00
  },
  "inventory": {
    "E": { "available": 15, "reserved": 42, "capacity": 60, "utilization_pct": 70 },
    "F": { "available": 8, "reserved": 28, "capacity": 40, "utilization_pct": 70 }
  },
  "constraints": {
    "margin_floor": { "E": 45.93, "F": 49.60, "S": 53.73 },
    "max_rate_increase_pct": 0.15,
    "location_strategy": "competitive_parity",
    "location_variance_tolerance": 0.05
  },
  "phase_2_optional": {
    "dow_multiplier": 1.35,
    "lead_time_distribution": { "<1d": 0.12, "1-3d": 0.20, "4-7d": 0.25, "8-14d": 0.25, "15+d": 0.18 },
    "channel_margins": { "costco": 0.18, "priceline": 0.15, "gds": 0.30, "budget_com": 0.25 },
    "elasticity_coefficient": -1.6,
    "competitor_profiles": { "hertz": "premium", "enterprise": "volume", "budget_com": "self" },
    "booking_velocity": 1.41
  }
}
```

---

## 3. AGENT DECISION LOGIC (Per Location Agent)

### 3.1 Agent Prompt Template (Phase 1: Core Logic)

```
You are a revenue optimization agent for [LOCATION] Budget Rent a Car.

**Your Goal**: Recommend daily car rental rates to maximize revenue while staying competitive.

**Input Data**:
- Reservation trend (7d): [BOOKINGS]
- Current rates: [CURRENT]
- Competitor rates: [COMPETITORS] (Hertz, Enterprise, Avis, National, Alamo)
- Inventory pressure: [UTILIZATION]
- Booking sources breakdown: [SOURCES]
- Location strategy: [STRATEGY] (competitive_parity, moderate_premium, or aggressive_premium)

**Phase 1 Scoring Rules** (return as JSON):

1. **DEMAND SCORE** (0-100):
   - Bookings trend: if ↑20% week-over-week → +30 points
   - Cancellation rate: if low (<5%) → +10 points
   - Booking lead time: if advance (>3 days avg) → +15 points
   - Booking source premium mix: if GDS/corporate >10% → +5 points
   - Result: max 100 points

2. **INVENTORY PRESSURE** (0-100):
   - Utilization <60% → 0 points (excess supply)
   - Utilization 60-70% → 20 points (room for growth)
   - Utilization 70-85% → 40 points (balanced)
   - Utilization 85-95% → 70 points (tight; raise prices)
   - Utilization >95% → 100 points (critical shortage; aggressive increase)

3. **COMPETITIVE SCORE** (-50 to +50):
   - Compute: median(Hertz, Enterprise, Avis, National) as "big 3 median"
   - Ignore: Alamo (low threat), Budget.com (that's you)
   - If our price < big_3_median -5% → +50 (underpriced, raise)
   - If our price = big_3_median ±3% → 0 (fair position)
   - If our price > big_3_median +5% → -30 (overpriced, lower)
   - Adjustment: if location_variance_tolerance <= 0.05 (airport/strip) → be conservative

4. **MARGIN CHECK**:
   - Min rate = margin_floor per car class (pre-calculated in constraints)
   - If recommendation < min rate → adjust UP to min rate
   - Never price below margin floor

5. **LOCATION STRATEGY ADJUSTMENT**:
   - If strategy = "competitive_parity" → cap raise at +5%
   - If strategy = "moderate_premium" → allow raise up to +15%
   - If strategy = "aggressive_premium" → allow raise up to +25%

6. **FINAL RECOMMENDATION**:
   - Weighted average: (demand_score × 0.40) + (inventory_pressure × 0.35) + (competitive_score × 0.25)
   - Map score (0-100) to % adjustment: 50 = 0%, 70 = +10%, 30 = -10%, etc.
   - Apply location strategy cap
   - Apply margin floor
   - Cap at max_rate_increase_pct (15%) for safety
   - Return: { car_class, recommended_rate, confidence (0-100), reasoning }

**Output Format**:
{
  "location": "[LOCATION]",
  "analysis": {
    "demand_score": 75,
    "inventory_pressure": 65,
    "competitive_score": 10,
    "weighted_score": 58,
    "location_strategy": "competitive_parity"
  },
  "recommendations": [
    {
      "car_class": "E",
      "current_rate": 42.00,
      "recommended_rate": 46.00,
      "change_pct": 9.5,
      "confidence": 85,
      "reasoning": "Demand strong (bookings +18%), inventory balanced (70% util). Competitive position fair (median $46.50, ours $42.00). Raise 9.5% to $46.00, still below median. Margin floor ($45.93) respected."
    }
  ],
  "assumptions": "Based on last 7 days data; no external events this week."
}
```

### 3.2 Agent Prompt Template (Phase 2: Enhanced Logic with New Data)

Once Phase 1 is running, add these refinements:

```
**Phase 2 Enhancement**: Add to scoring after Phase 1 runs stable (Week 4+)

7. **DAY-OF-WEEK MULTIPLIER** (only if today is Fri/Sat):
   - Friday: +35% volume surge expected
   - Saturday: +42% volume surge expected
   - If day = Friday/Saturday AND inventory >70% → add +20 points to demand_score
   - This nudges rates higher on peak days naturally

8. **LEAD TIME DISTRIBUTION SCORING**:
   - Analyze: what % of bookings are 15+ days advance?
   - If 15+d % > 20% (good), demand is predictable → add +10 points
   - If 15+d % < 10% (bad), demand is unpredictable → subtract -10 points
   - This helps agents understand booking quality/certainty

9. **CHANNEL-SPECIFIC RATE RECOMMENDATIONS** (new):
   - For each channel, compute weighted recommendation
   - GDS channel: apply +0.30 margin floor (premium; can raise more)
   - Costco channel: apply +0.18 margin floor (discount; be competitive)
   - Priceline channel: apply +0.15 margin floor (OTA hunters; minimal margin)
   - Budget.com channel: apply +0.25 margin floor (your direct; optimize for profit)
   - Return: "For [channel], recommend [rate] (margin [pct])"

10. **BOOKING VELOCITY SURGE DETECTION**:
    - If bookings_last_24h > 1.3× baseline → demand surge detected
    - If surge + inventory >70% → add +25 points to inventory_pressure_score
    - This triggers faster rate increases when demand spikes

11. **ELASTICITY-ADJUSTED SCORING** (by car class):
    - Economy (E): demand_weight = 50%, inventory = 30%, competitive = 20%
      (price-sensitive; protect volume)
    - Mid-size (F): demand_weight = 40%, inventory = 35%, competitive = 25%
      (baseline)
    - Sedan (S): demand_weight = 30%, inventory = 35%, competitive = 35%
      (less elastic; can raise on competition)
    - Large (L): demand_weight = 25%, inventory = 35%, competitive = 40%
      (premium; match on competition)

12. **LOCATION-SPECIFIC AGGRESSIVENESS**:
    - Airport/Strip (tolerance <0.06): weighted_score < 60 → don't raise
    - Off-strip (tolerance 0.15-0.22): weighted_score > 50 → can raise aggressively
    - Apply: final_rate_change = weighted_adjustment × location_aggressiveness_factor

**Updated Output** (Phase 2):
{
  "location": "[LOCATION]",
  "dow_context": "Friday +35% volume expected",
  "channel_breakdown": {
    "gds": { "volume": 0.15, "recommended_rate": 58.00, "margin": 0.30 },
    "costco": { "volume": 0.35, "recommended_rate": 43.00, "margin": 0.18 },
    "priceline": { "volume": 0.25, "recommended_rate": 41.00, "margin": 0.15 }
  },
  "booking_velocity": "1.41× baseline (surge detected)",
  "analysis": {
    "demand_score": 85,
    "inventory_pressure": 75,
    "competitive_score": 15,
    "dow_boost": 20,
    "velocity_boost": 25,
    "elasticity_adjusted_demand_weight": 0.40,
    "final_weighted_score": 70
  },
  "recommendations": [
    {
      "car_class": "E",
      "current_rate": 42.00,
      "recommended_rate": 47.50,
      "change_pct": 13.1,
      "confidence": 88,
      "reasoning": "Friday surge (expected +35% volume) + real-time velocity 1.41× + inventory tight (70%) = aggressive raise opportunity. Elasticity-adjusted for economy cars (volume-focused): +13% is high but justified. Margin check: $47.50 > $45.93 (floor). Competitive check: $47.50 = median $46.50 + 2%. Clean raise."
    }
  ]
}
```

### 3.3 Example Agent Output (Phase 1)

```json
{
  "location": "LAS_AIRPORT",
  "analysis": {
    "demand_score": 72,
    "inventory_pressure": 68,
    "competitive_score": 15,
    "weighted_score": 63
  },
  "recommendations": [
    {
      "car_class": "E",
      "current_rate": 42.00,
      "recommended_rate": 46.00,
      "change_pct": 9.52,
      "confidence": 87,
      "reasoning": "Bookings up 18% WoW (120 this week vs 102 last). Inventory at 92% util (42 reserved / 60 capacity). Competitor median $45.50 (Hertz $49.99, Enterprise $44.99). Current rate undercutting by 7.5%. Margin floor $35.00 (cost $28 × 1.25). Safe to increase 9.5% → $46.00, still competitive."
    },
    {
      "car_class": "F",
      "current_rate": 48.00,
      "recommended_rate": 50.40,
      "change_pct": 5.00,
      "confidence": 72,
      "reasoning": "Bookings stable (85 vs 87 last week). Inventory 70% util (lower pressure). Competitor median $52.50. We're $4.50 below—good positioning. Modest increase 5% → $50.40 balances margin and competitiveness."
    }
  ],
  "assumptions": "Last 7d data. No Vegas events this week. Costco (35% of bookings) is price-sensitive; GDS (15%) less so."
}
```

---

## 4. CONSENSUS & DECISION LOGIC (Master Agent)

### 4.1 Consensus Algorithm

```python
# After collecting votes from 8 location agents
for each car_class:
    all_votes = [agent[car_class].recommended_rate for agent in location_agents]
    # E.g., all_votes for "E" at 8 locations: [46.00, 45.50, 47.00, 45.00, 46.50, 44.00, 48.00, 46.50]
    
    # Consensus: weighted median (favor higher confidence agents)
    confidence_weights = [agent[car_class].confidence for agent in location_agents]
    weighted_median = weighted_quantile(all_votes, q=0.5, weights=confidence_weights)
    
    # Apply constraints
    margin_floor = cost_per_day × (1 + margin_floor_pct)
    rate = max(weighted_median, margin_floor)
    rate = min(rate, current_rate × 1.15)  # cap increase at 15%
    
    # Store decision
    decision = {
        "location": location,
        "car_class": car_class,
        "new_rate": rate,
        "votes": all_votes,
        "consensus": weighted_median,
        "agent_confidences": confidence_weights,
        "decision_timestamp": datetime.now()
    }
```

### 4.2 Rate History Schema

```sql
CREATE TABLE rate_history (
  id INTEGER PRIMARY KEY,
  date DATE,
  location TEXT,
  car_class TEXT,
  previous_rate REAL,
  new_rate REAL,
  change_pct REAL,
  consensus_value REAL,
  min_vote REAL,
  max_vote REAL,
  agent_confidences TEXT, -- JSON: [85, 72, 90, ...]
  master_notes TEXT,
  live_at TIMESTAMP,
  created_at TIMESTAMP
);

-- Example:
-- date: 2026-02-18
-- location: LAS_AIRPORT
-- car_class: E
-- previous_rate: 42.00
-- new_rate: 46.00
-- change_pct: 9.52
-- consensus_value: 46.00
-- min_vote: 44.00, max_vote: 48.00
-- agent_confidences: [87, 82, 91, 78, 85, 88, 80, 86]
-- master_notes: "Strong consensus. Demand up. Inventory tight."
-- live_at: 2026-02-18 06:15:00
```

### 4.3 Master Agent Constraint Enforcement

```
BEFORE publishing rates:

1. **Margin Floor**
   if new_rate < cost_per_day × (1 + 0.25):
       new_rate = cost_per_day × 1.25
       log: "Rate floor enforced"

2. **Max Day-over-Day Increase**
   if (new_rate - previous_rate) / previous_rate > 0.15:
       new_rate = previous_rate × 1.15
       log: "Rate increase capped at 15%"

3. **Competitive Parity Check**
   competitor_median = median([hertz, enterprise, avis, national])
   if new_rate > competitor_median × 1.10:
       log: "WARNING: 10%+ above median competitor. Confidence-check agent votes."
       (Don't block, but log for audit)

4. **Global Revenue Target** (optional, Phase 2)
   if location_revenue_forecast < revenue_target:
       new_rate = increase_rate_to_hit_target(new_rate, revenue_target)
       log: "Adjusted to meet revenue target"

5. **Publish**
   write new_rates to PMS
   write decision_log to rate_history DB
   send_slack_alert(summary)
```

---

## 5. TECH STACK & INFRASTRUCTURE

### 5.1 Core Components

| Component | Tech | Cost | Notes |
|-----------|------|------|-------|
| **Agent Runtime** | Gradient (Opus) | ~$0.15 per 1M tokens | 9 agents × ~5K tokens/run = 45K tokens/day ≈ $0.007/day = $2.10/month |
| **Data Scraper** | Playwright + Celery | $15-25/month | 3-5 DigitalOcean VMs ($5 each), staggered requests |
| **Database** | SQLite (local) or PostgreSQL ($10/month) | $0-10 | Start with SQLite, migrate to PG if >10GB |
| **Scheduler** | System cron + Python | $0 | Or: APScheduler/Celery Beat |
| **Audit Log / History** | SQLite DB | $0 | 1GB ≈ 50K rate records × 8 locations × 365 days |
| **PMS Integration** | REST API or SSH tunnel | $0 | Depends on your PMS (Wizard/WAND) |
| **Monitoring** | Datadog free tier or Sentry | $0-20 | For error tracking |

**Monthly Infrastructure Cost**: ~$50-60 (most expensive = scraper VMs)

### 5.2 Code Structure (Python)

```
rate_mgmt_platform/
├── config/
│   ├── agents.yaml          # Agent definitions, model overrides
│   ├── constraints.yaml     # Margin floors, rate caps
│   └── locations.yaml       # 8 location configs
├── data/
│   ├── fetchers.py          # Reservation CSV parser, competitor scraper
│   ├── models.py            # SQLAlchemy models (RateHistory, etc.)
│   └── cache.py             # In-memory cache for latest data
├── agents/
│   ├── master.py            # Master orchestrator agent
│   ├── location.py          # Location agent template
│   ├── scraper.py           # Competitor scraper agent
│   ├── parser.py            # Reservation parser agent
│   └── prompts.py           # Agent prompt templates
├── consensus/
│   ├── voting.py            # Weighted median, consensus logic
│   └── constraints.py       # Margin floor, rate cap enforcement
├── pms_integration/
│   ├── wizard_connector.py  # Write rates to your PMS
│   └── audit_logger.py      # Log all changes
├── cron/
│   ├── daily_run.py         # 6 AM cron entry point
│   └── scraper_runner.py    # 8-11 PM scraper cron
├── tests/
│   ├── test_consensus.py
│   ├── test_constraints.py
│   └── test_agent_logic.py
├── logs/
│   └── rate_decisions.log
├── requirements.txt
└── main.py                  # CLI entry point
```

---

## 6. RATE HISTORY & AUDITABILITY

### 6.1 Query Examples

```sql
-- View all rate changes in last 7 days
SELECT * FROM rate_history
WHERE date >= CURRENT_DATE - 7
ORDER BY date DESC, location;

-- Revert to previous rate
BEGIN TRANSACTION;
UPDATE rate_history SET reverted_at = NOW() WHERE id = 12345;
-- Then write old rate back to PMS manually or via rollback script
COMMIT;

-- Analyze consensus variance (are agents agreeing?)
SELECT 
  location,
  car_class,
  date,
  (MAX(agent_confidences) - MIN(agent_confidences)) as confidence_variance,
  (MAX(votes) - MIN(votes)) as vote_spread_pct
FROM rate_history
GROUP BY location, car_class, date
ORDER BY vote_spread_pct DESC;
```

### 6.2 Audit Trail (Mandatory)

Every rate change must record:
1. **What changed**: location, car_class, old rate, new rate
2. **Why**: agent votes, consensus score, constraints applied
3. **When**: timestamp, who approved (system), when live
4. **Reversibility**: link to previous rate, one-click revert script

**Example UI Dashboard** (Phase 2):
```
Location: LAS Airport | Date: Feb 18, 2026
┌─────────────────────────────────────┐
│ Car Class E                          │
│ Previous: $42.00 → New: $46.00       │
│ Change: +9.5%                        │
│                                      │
│ Agent Votes:                         │
│  ├─ LAS Airport:    $46.00 (87%)     │
│  ├─ Center Strip:   $45.50 (82%)     │
│  ├─ W Sahara:       $47.00 (91%)     │
│  └─ ... (5 more)                     │
│                                      │
│ Consensus: $46.00                    │
│ Constraints Applied: Margin floor OK │
│ Master Notes: Strong demand, tight   │
│               inventory.             │
│                                      │
│ [Revert] [Approve] [View History]    │
└─────────────────────────────────────┘
```

---

## 7. IMPLEMENTATION ROADMAP (8 Weeks)

### **Week 1-2: Foundation**
- [ ] Design SQLite schema (rate_history, competitor_snapshot, reservation_cache)
- [ ] Build reservation CSV parser (ingest your nightly export)
- [ ] Build competitor scraper (Playwright agents for Hertz, Enterprise, Avis)
- [ ] Write unit tests for data pipeline
- [ ] **Deliverable**: Data ingestion working, 7-day history populated

### **Week 3: Agent Design**
- [ ] Write location agent prompt + testing with Opus
- [ ] Write master orchestrator logic (consensus voting, constraints)
- [ ] Mock data for 8 locations with realistic demand/competitive data
- [ ] **Deliverable**: Agents runnable locally, producing votes + consensus

### **Week 4: Consensus & History**
- [ ] Implement weighted median voting
- [ ] Implement constraint enforcement (margin floor, rate caps)
- [ ] Build rate_history DB logging
- [ ] Build revert/rollback logic
- [ ] **Deliverable**: Full decision pipeline end-to-end, logged cleanly

### **Week 5: PMS Integration (Real Data)**
- [ ] Integrate with your PMS (Wizard/WAND API or SSH tunnel to push rates)
- [ ] Test on staging/non-prod location first (maybe Henderson Executive)
- [ ] Validate rates appear correctly in booking system
- [ ] **Deliverable**: Live rate push working on one location

### **Week 6: Cron & Automation**
- [ ] Schedule daily cron at 6:00 AM PST
- [ ] Schedule nightly scraper at 8 PM PST (Celery workers on DO VMs)
- [ ] Add error handling (scraper fails, data stale, etc.)
- [ ] Monitoring + alerting (Slack notifications on rate changes)
- [ ] **Deliverable**: Fully automated, running daily

### **Week 7: Pilot (One Location)**
- [ ] Go live on **LAS Airport** for 1 week
- [ ] Manual oversight: review agent votes daily, revert if needed
- [ ] Measure: revenue per booking, occupancy %, conversion rate
- [ ] **Deliverable**: Proof-of-concept metrics, human sign-off

### **Week 8: Scale & Docs**
- [ ] Document decision logic, agent prompts, constraints for other team members
- [ ] Plan rollout to remaining 7 locations (2 per week)
- [ ] Build audit dashboard (view history, explain decisions, revert)
- [ ] **Deliverable**: Production-ready, documented, scalable

---

## 8. SUCCESS METRICS & KPIs

### 8.1 Revenue Metrics

| Metric | Baseline | Target (4 weeks) | Measurement |
|--------|----------|------------------|-------------|
| **Revenue per Booking** | Existing | +5-8% | (total_revenue / bookings) |
| **Occupancy Rate** | ~85% | +2-3% | (reserved / capacity) |
| **Gross Margin** | 25%+ | 25%+ (maintained) | (rate - cost) / rate |
| **ASP (Avg Selling Price)** | TBD | +8% | avg daily rate across classes |

### 8.2 Operational Metrics

| Metric | Target |
|--------|--------|
| **Agent Consensus** | >70% within $2 of median vote |
| **Decision Latency** | <15 min from 6 AM trigger to rates live |
| **Scraper Success Rate** | >95% (all competitors, all locations) |
| **Rate Change Frequency** | 1× daily (morning cron only) |
| **Revert Rate** | <5% of decisions (human override rare) |

### 8.3 Cost Metrics

| Component | Monthly Cost |
|-----------|--------------|
| Gradient (Opus × 9 agents) | $2-5 |
| Competitor Scraper (3 VMs) | $15 |
| Database (SQLite local) | $0 |
| **Total** | **~$20/month** |

**ROI Target**: If +5% revenue on $100k/month rental revenue = +$5k/month. Cost is <0.5%.

---

## 9. FAILURE MODES & MITIGATION

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| **Scraper blocked by competitor** | Medium | Stale competitor data | Fallback to 7-day rolling avg; manual override |
| **Agent hallucination (bad rate rec)** | Low | Revenue loss if not caught | Human review 1st week; revert within 2h if needed |
| **PMS integration breaks** | Low | Rates don't update | Dry-run mode first; rollback to manual rates |
| **Data stale (res CSV delayed)** | Low | Outdated decisions | Check timestamp; skip run if >6h old |
| **Cost explosion** | Very Low | Budget overrun | Capped token usage; Haiku for scrapers not main agents |

---

## 10. PHASE 2 ENHANCEMENTS (Future)

- [ ] **External Events** integration (CES, conventions, sports schedules)
- [ ] **Multi-car-class optimization** (if economy full, push customers to mid-size)
- [ ] **A/B testing** (run rules-based vs. agent rates on different locations)
- [ ] **Long-term forecasting** (30-90 day rate planning, seasonal patterns)
- [ ] **Customer segment tuning** (different rates for Costco vs. GDS vs. corporate)
- [ ] **Margin target optimization** (target revenue vs. occupancy tradeoff)
- [ ] **Real-time rate updates** (react to booking surge, not just daily cron)

---

## 11. GETTING STARTED: Week 1 Action Items

1. **Prepare reservation data**:
   - Set up nightly CSV export from your PMS to `/data/reservations_YYYY-MM-DD.csv`
   - Confirm schema matches Budget's standard columns (ci_date, car_class, bookings, revenue, etc.)

2. **Start competitor scraper**:
   - List all 6 competitors (Hertz, Enterprise, Avis, National, Alamo, Budget.com)
   - Identify each competitor's URL pattern for LAS Airport economy car (as test)
   - Set up 3 DigitalOcean VMs with Playwright + Celery

3. **Design database**:
   - Create SQLite DB: `rate_mgmt.db`
   - Create tables: `rate_history`, `competitor_snapshot`, `reservation_cache`
   - Test data pipeline with sample CSV

4. **Kick off agent development**:
   - Write location agent prompt
   - Test with Opus on sample data
   - Iterate on scoring logic

5. **Slack/Monitoring**:
   - Set up Slack integration for daily alerts
   - Choose monitoring tool (Datadog free or Sentry)

---

## 12. SUMMARY

| Aspect | Decision |
|--------|----------|
| **Agent Model** | Opus (9 agents: 1 master + 8 location) |
| **Data Agent Model** | Haiku (3 data agents: scraper, parser, events) |
| **Schedule** | Daily cron 6 AM PST; nightly scraper 8 PM |
| **Consensus** | Weighted median with confidence weighting |
| **History** | SQLite `rate_history` table (reversible, auditable) |
| **Scraping** | Playwright + Celery on 3-5 DO VMs ($15-25/mo) |
| **Timeline** | 8 weeks (MVP on 1 location, then scale 2×/week) |
| **Cost** | ~$20-60/month (infrastructure + API) |
| **ROI** | 5-8% revenue lift = $5k+/month on $100k base |

---

## APPENDIX: Complete Data Specifications

**See `docs/DATA_INPUTS.md` for detailed specifications of all data inputs:**

### Phase 1 (MVP) Inputs
1. Nightly reservation data
2. Competitor price scraping
3. Current rates from PMS
4. Fleet inventory
5. Cost structure

### Phase 1+ Quick Wins (High-Impact, Low Effort)
6. Day-of-week demand multipliers (+2-3% revenue)
7. Booking lead time curves (+3-4% revenue)
8. Channel-specific economics (+2-3% revenue)
9. Price elasticity by car class (+1-2% revenue)
10. Competitor strength profiles (prevents margin leakage)
11. Location market context (+1-2% revenue)

### Phase 2 Enhancements
12. Real-time booking velocity (surge detection)
13. Customer lifetime value & repeat impact
14. External events calendar & demand spikes

**Total potential revenue lift from all inputs: 7-13%** (conservative estimates; can be higher)

See DATA_INPUTS.md for SQL queries, JSON schemas, implementation timelines, and ROI projections.
