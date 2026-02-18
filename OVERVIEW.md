# FleetYield: AI Agent Orchestration for Dynamic Rate Management

**Budget Rent a Car Las Vegas (Malco Enterprises)**

## Project Goal

Build a **multi-agent AI system** that autonomously optimizes car rental rates across 8 Las Vegas locations using:
- Real-time reservation data (nightly CSV dumps)
- Competitor price scraping (Hertz, Enterprise, Avis, etc.)
- Consensus-driven decision making (master + location agents)
- Complete audit trail & reversibility

**Expected ROI**: 5-8% revenue lift ($5k+/month on $100k base revenue)

## Quick Links

- **[ARCHITECTURE_PLAN.md](./docs/ARCHITECTURE_PLAN.md)** — Full 8-week implementation roadmap
- **[DATA_PIPELINE.md](./docs/DATA_PIPELINE.md)** — Reservation parsing, competitor scraping, data flow
- **[AGENT_LOGIC.md](./docs/AGENT_LOGIC.md)** — Decision prompts, scoring rules, consensus algorithm
- **[RATE_HISTORY.md](./docs/RATE_HISTORY.md)** — Database schema, auditability, rollback procedures

## Key Metrics

| Metric | Target |
|--------|--------|
| **Revenue Lift** | +5-8% per booking |
| **Timeline** | 8 weeks to MVP (1 location) |
| **Cost** | ~$20-60/month |
| **Agent Consensus** | >70% within $2 of median |
| **Uptime** | 99%+ (cron-based) |

## Architecture at a Glance

```
Master Orchestrator (Opus)
├─ 8 Location Agents (Opus)
│  ├─ LAS Airport
│  ├─ Center Strip
│  ├─ W Sahara
│  ├─ Golden Nugget
│  ├─ Tropicana
│  ├─ Gibson
│  ├─ Henderson Executive
│  └─ Losee
└─ 3 Data Agents (Haiku)
   ├─ Competitor Scraper
   ├─ Reservation Parser
   └─ Event Calendar (Phase 2)
```

**Decision Flow**: Daily 6 AM cron → agents vote → consensus → constraints → rates live → history logged

## Why FleetYield?

"Fleet" = your 8 rental car locations  
"Yield" = revenue optimization (hotel/airline term for dynamic pricing)

Combined: **FleetYield** = autonomous dynamic pricing for multi-location fleet rental

---

## Status

- [ ] Phase 1: Data pipeline (Week 1-2)
- [ ] Phase 1: Agent logic (Week 3-4)
- [ ] Phase 1: PMS integration (Week 5)
- [ ] Phase 1: Automation (Week 6)
- [ ] Pilot: LAS Airport (Week 7)
- [ ] Scale: Remaining locations (Week 8+)

---

**Next**: Review ARCHITECTURE_PLAN.md, then begin Week 1 data pipeline development.
