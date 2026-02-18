# FleetYield: Dynamic Rate Management for Multi-Location Rental Fleets

An **autonomous AI agent orchestration platform** that optimizes car rental rates across multiple locations using consensus-driven decision making, real-time competitive intelligence, and complete auditability.

## Features

✅ **Hybrid Multi-Agent Architecture**  
- Master orchestrator + 8 location agents (Opus)  
- 3 data agents (Haiku) for cost efficiency  

✅ **Daily Automated Optimization**  
- Cron-based rate decisions at 6 AM PST  
- Nightly competitor scraping (8-11 PM)  
- Complete audit trail with rollback capability  

✅ **Consensus-Driven Decisions**  
- Weighted median voting from all locations  
- Demand scoring + inventory pressure + competitive analysis  
- Global constraint enforcement (margin floors, rate caps)  

✅ **Production-Ready**  
- ~$20-60/month infrastructure cost  
- 5-8% expected revenue lift  
- Full reversibility and human oversight  

## Quick Start

### Prerequisites
- Python 3.9+
- SQLite3
- DigitalOcean account (for competitor scraper VMs, optional)

### Installation

```bash
git clone https://github.com/0x000NULL/FleetYield.git
cd FleetYield
pip install -r requirements.txt
```

### Configuration

1. **Database Setup**
   ```bash
   python -m src.db.init
   ```

2. **Environment Variables** (`.env`)
   ```
   PMS_API_URL=https://your-pms-api.com
   PMS_API_TOKEN=your_token_here
   GRADIENT_API_KEY=your_gradient_key
   SLACK_WEBHOOK_URL=https://hooks.slack.com/...
   ```

3. **Location Config** (`config/locations.yaml`)
   ```yaml
   locations:
     - code: LAS_AIRPORT
       name: LAS Airport
       cost_per_day:
         E: 28
         F: 32
         S: 35
     - code: CENTER_STRIP
       name: Center Strip
       cost_per_day:
         E: 30
         F: 34
   ```

### Running the Platform

**Full Pipeline (6 AM):**
```bash
python src/cron/daily_run.py
```

**Competitor Scraper Only (8-11 PM):**
```bash
python src/cron/scraper_runner.py hertz enterprise avis
```

**Parse Reservations Only (5:45 AM):**
```bash
python src/cron/parse_reservations.py /data/reservations_2026-02-18.csv
```

### View Rate History

```bash
# Last 7 days
sqlite3 /data/rate_mgmt.db "SELECT * FROM rate_history WHERE date >= date('now', '-7 days') ORDER BY date DESC;"

# Export to CSV for analysis
sqlite3 -header -csv /data/rate_mgmt.db "SELECT * FROM rate_history;" > rate_history.csv
```

### Revert a Rate Decision

```bash
python src/pms_integration/rollback.py --date 2026-02-18 --location LAS_AIRPORT --car_class E
```

## Architecture

```
Master Orchestrator (Opus)
├─ Location Agent: LAS Airport
├─ Location Agent: Center Strip
├─ Location Agent: W Sahara
├─ Location Agent: Golden Nugget
├─ Location Agent: Tropicana
├─ Location Agent: Gibson
├─ Location Agent: Henderson Executive
├─ Location Agent: Losee
│
└─ Data Agents (Haiku)
   ├─ Competitor Scraper
   ├─ Reservation Parser
   └─ Event Calendar (Phase 2)
```

**Data Flow:**
1. Reservation CSV → parse → `reservation_cache`
2. Competitor scrape → store → `competitor_rates`
3. Current rates from PMS → `current_rates`
4. Master assembles data → dispatch to 8 agents
5. Agents vote → consensus → constraints → publish
6. All decisions logged → `rate_history`

## Documentation

- **[OVERVIEW.md](./OVERVIEW.md)** — Project goals & quick reference
- **[ARCHITECTURE_PLAN.md](./docs/ARCHITECTURE_PLAN.md)** — Full 8-week implementation roadmap, agent hierarchy, decision logic (Phase 1 + Phase 2 enhancements)
- **[DATA_PIPELINE.md](./docs/DATA_PIPELINE.md)** — Reservation parsing, competitor scraping, data schemas, cron integration
- **[DATA_INPUTS.md](./docs/DATA_INPUTS.md)** — ⭐ **Comprehensive data specifications** for AI agents (Phase 1 baseline + Phase 1+ quick wins + Phase 2 enhancements). Includes SQL queries, JSON schemas, revenue impact projections, implementation timeline
- **[DATA_SOURCES_COMPLETE_INVENTORY.md](./docs/DATA_SOURCES_COMPLETE_INVENTORY.md)** — Complete SQL Server database inventory (4.6M records across 5 tables), use cases, Phase 1 & 2 integration strategy

## KPIs & Success Metrics

| Metric | Target |
|--------|--------|
| **Revenue Lift** | +5-8% per booking |
| **Agent Consensus** | >70% within $2 of median |
| **Decision Latency** | <15 min from trigger to live |
| **Scraper Uptime** | >95% success rate |
| **Monthly Cost** | ~$20-60 |

## Implementation Timeline

| Week | Focus | Deliverable |
|------|-------|-------------|
| 1-2 | Data Pipeline | CSV parser, competitor scraper, SQLite DB |
| 3-4 | Agent Logic | Location agents, master orchestrator, consensus |
| 5 | PMS Integration | Live rate push, staging validation |
| 6 | Automation | Cron jobs, monitoring, alerting |
| 7 | Pilot | Go-live on LAS Airport (1 week) |
| 8 | Scale | Remaining 7 locations (2x/week rollout) |

## Tech Stack

| Component | Tech | Cost |
|-----------|------|------|
| Agent Runtime | Gradient (Opus) | $2-5/mo |
| Competitor Scraper | Playwright + Celery | $15-25/mo |
| Database | SQLite (local) | $0 |
| Scheduling | System cron | $0 |
| Monitoring | Datadog free | $0-20 |

## Failure Modes & Mitigation

| Risk | Mitigation |
|------|-----------|
| Scraper blocked | 7-day rolling average fallback |
| Agent hallucination | Human review first week, revert within 2h |
| PMS integration break | Dry-run mode, rollback to manual |
| CSV delayed | Skip run if >6h old, retry at 6:30 AM |
| Token cost explosion | Capped usage, Haiku for data agents |

## Phase 2 Roadmap

- [ ] External events integration (CES, conventions, sports)
- [ ] Multi-car-class optimization (spillover logic)
- [ ] A/B testing framework (agent vs. rules rates)
- [ ] 30-90 day forecasting
- [ ] Customer segment tuning (Costco vs. GDS pricing)
- [ ] Real-time rate updates (not just daily cron)

## Contact & Support

- **Issues**: GitHub Issues
- **Docs**: See `/docs` folder
- **Questions**: Refer to ARCHITECTURE_PLAN.md or DATA_PIPELINE.md

## License

This project is proprietary to Malco Enterprises (dba Budget Rent a Car Las Vegas).

## Author

Vex (AI Agent)  
Built for Ethan Aldrich, CTO Malco Enterprises  
Started: February 2026
