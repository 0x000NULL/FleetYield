# MONITORING & ALERTING - Real-Time Ops Visibility

## Section 1: Health Checks & Logging

### Lock File Strategy (Prevent Duplicate Runs)

```bash
#!/bin/bash
# fleetyield_cron_main.sh

LOCK_FILE="/tmp/fleetyield.lock"
MAX_AGE=3600  # 1 hour

# Check if lock exists and is stale
if [[ -f "$LOCK_FILE" ]]; then
  LOCK_AGE=$(($(date +%s) - $(stat -f%m "$LOCK_FILE" 2>/dev/null || stat -c%Y "$LOCK_FILE")))
  
  if [[ $LOCK_AGE -lt $MAX_AGE ]]; then
    echo "ERROR: Previous job still running (lock age: ${LOCK_AGE}s)"
    log_to_json "{\"status\":\"ERROR\", \"error\":\"duplicate_run\", \"lock_age\":$LOCK_AGE}"
    exit 1
  else
    echo "WARNING: Stale lock found (age: ${LOCK_AGE}s); removing"
    rm "$LOCK_FILE"
  fi
fi

# Create lock
touch "$LOCK_FILE"
trap "rm $LOCK_FILE" EXIT

# ... run jobs
```

### Job Timeout Enforcement

```python
import signal
import sys

class TimeoutError(Exception):
    pass

def timeout_handler(signum, frame):
    raise TimeoutError("Job exceeded max duration")

def run_job_with_timeout(job_name, max_seconds, job_func):
    """Execute job with timeout"""
    
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(max_seconds)
    
    start = datetime.now()
    try:
        result = job_func()
        signal.alarm(0)  # Cancel alarm
        duration = (datetime.now() - start).total_seconds()
        
        log_json({
            "job": job_name,
            "status": "SUCCESS",
            "duration_seconds": duration,
            "timestamp": datetime.now().isoformat()
        })
        return result
    
    except TimeoutError:
        duration = (datetime.now() - start).total_seconds()
        
        log_json({
            "job": job_name,
            "status": "TIMEOUT",
            "max_seconds": max_seconds,
            "duration_seconds": duration,
            "error": f"Job exceeded {max_seconds}s limit",
            "timestamp": datetime.now().isoformat()
        })
        
        alert_ops(f"CRITICAL: {job_name} timed out after {duration}s")
        raise
    
    except Exception as e:
        duration = (datetime.now() - start).total_seconds()
        
        log_json({
            "job": job_name,
            "status": "ERROR",
            "duration_seconds": duration,
            "error": str(e),
            "timestamp": datetime.now().isoformat()
        })
        
        raise

# Usage
MAX_TIMEOUTS = {
    'parse_reservations': 300,      # 5 min
    'competitor_scraper': 3600,     # 60 min
    'agent_voting': 600,            # 10 min
    'pms_write': 300                # 5 min
}

for job_name, max_seconds in MAX_TIMEOUTS.items():
    run_job_with_timeout(job_name, max_seconds, globals()[job_name])
```

### Log Format (JSON)

```json
{
  "timestamp": "2026-02-18T06:15:30.123Z",
  "job_name": "parse_reservations",
  "status": "SUCCESS",
  "duration_seconds": 145,
  "rows_processed": 6750,
  "rows_valid": 6710,
  "rows_invalid": 40,
  "pct_invalid": 0.59,
  "error_codes": null,
  "exit_code": 0,
  "log_level": "INFO"
}
```

---

## Section 2: Alert Rules (Slack)

### Alert Decision Tree

```
Job completion?
├─ NO (timeout/crash)
│  └─ Alert: CRITICAL (ops notified immediately)
│
└─ YES (completed)
    ├─ Exit code 0 (success)
    │  └─ Log: INFO
    │
    └─ Exit code != 0 (error)
        ├─ Error type?
        │  ├─ CSV validation fail (>10% invalid)
        │  │  └─ Alert: ERROR + fallback triggered
        │  │
        │  ├─ Scraper fail (all competitors)
        │  │  └─ Alert: WARNING + 7-day avg fallback
        │  │
        │  ├─ PMS write fail
        │  │  └─ Alert: CRITICAL + auto-revert triggered
        │  │
        │  └─ Other
        │     └─ Alert: WARNING
```

### Alert Rules Configuration

```yaml
alerts:
  - name: "Job Timeout"
    condition: "exit_code == 124"  # SIGALRM
    severity: "CRITICAL"
    channel: "#fleetyield-alerts"
    mention: "@oncall"
    message: "🚨 {job_name} exceeded max time limit. Job killed."
  
  - name: "CSV Validation Failed"
    condition: "job == 'parse_reservations' && error_code == 'CSV_INVALID'"
    severity: "ERROR"
    channel: "#fleetyield-alerts"
    mention: "@ops"
    message: ":x: CSV validation failed ({pct_invalid}% invalid rows). Using 7-day rolling average."
  
  - name: "Scraper All Failed"
    condition: "job == 'competitor_scraper' && success_rate < 0.5"
    severity: "CRITICAL"
    channel: "#fleetyield-alerts"
    mention: "@oncall"
    message: ":warning: Scraper success rate {success_rate}% is critically low. Using 7-day average."
  
  - name: "PMS Write Failed"
    condition: "job == 'pms_write' && exit_code != 0"
    severity: "CRITICAL"
    channel: "#fleetyield-alerts"
    mention: "@oncall @ethan"
    message: ":x: PMS rate push FAILED. Auto-revert triggered. Manual verification required."
  
  - name: "Data Quality Warning"
    condition: "pct_invalid > 0.05 && pct_invalid <= 0.10"
    severity: "WARNING"
    channel: "#fleetyield-alerts"
    mention: "@ops"
    message: ":warning: Data quality degrading ({pct_invalid}% invalid rows). Monitoring."
```

---

## Section 3: Slack Bot Integration

### Slash Commands

```python
@slack_app.command("/fleetyield-status")
def handle_status_command(ack, body, respond):
    """Get current FleetYield system status"""
    ack()
    
    status = get_system_health()
    
    blocks = [
        {
            "type": "header",
            "text": {"type": "plain_text", "text": "🚗 FleetYield Status"}
        },
        {
            "type": "section",
            "fields": [
                {
                    "type": "mrkdwn",
                    "text": f"*Last Run:* {status['last_run_time']}"
                },
                {
                    "type": "mrkdwn",
                    "text": f"*Status:* {status['status']}"
                },
                {
                    "type": "mrkdwn",
                    "text": f"*CSV Quality:* {status['csv_pct_valid']}% valid"
                },
                {
                    "type": "mrkdwn",
                    "text": f"*Scraper Health:* {status['scraper_success_rate']}% success"
                },
                {
                    "type": "mrkdwn",
                    "text": f"*Occupancy Trend:* {status['occupancy_today']} (Δ {status['occupancy_delta']}%)"
                },
                {
                    "type": "mrkdwn",
                    "text": f"*Revenue Impact:* {status['revenue_impact_pct']}% lift"
                }
            ]
        }
    ]
    
    respond({"blocks": blocks})

@slack_app.command("/fleetyield-logs")
def handle_logs_command(ack, body, respond):
    """Fetch recent logs"""
    ack()
    
    logs = get_recent_logs(limit=10)
    text = "\n".join([f"`{log['timestamp']}` {log['status']}: {log['job_name']}" for log in logs])
    
    respond(f"```\n{text}\n```")
```

### Daily Digest (Scheduled)

```python
# Runs daily at 7:30 AM PST
def send_daily_digest():
    """Send morning digest of previous day's operations"""
    
    yesterday = datetime.now() - timedelta(days=1)
    stats = get_daily_stats(yesterday)
    
    digest = f"""
    📊 **FleetYield Daily Report** — {yesterday.strftime('%Y-%m-%d')}
    
    **Data Pipeline**
    • CSV: {stats['csv_rows']} rows, {stats['csv_pct_valid']}% valid
    • Scraper: {stats['scraper_success_rate']}% success ({stats['scraper_failures']} failures)
    
    **Agent Decisions**
    • Rates published: {stats['rates_published']} locations
    • Avg confidence: {stats['avg_confidence']}%
    • Rate changes: {stats['avg_rate_change_pct']}% (min: {stats['min_rate_change']}%, max: {stats['max_rate_change']}%)
    
    **Business Impact**
    • Occupancy: {stats['occupancy_today']}% (Δ {stats['occupancy_delta']}%)
    • Revenue lift: {stats['revenue_lift_pct']}%
    • Estimated daily impact: ${stats['daily_impact_usd']}
    
    **Alerts**: {stats['alert_count']} ({stats['critical_count']} critical, {stats['warning_count']} warnings)
    """
    
    slack_app.client.chat_postMessage(
        channel="#fleetyield-alerts",
        text=digest
    )
```

---

## Section 4: Data Quality Dashboard Spec

### Metrics Schema (Grafana / InfluxDB)

```python
class MetricsCollector:
    def record_csv_metrics(self, rows_valid, rows_invalid, duplicates):
        """Record CSV data quality metrics"""
        
        metrics = {
            "csv_rows_valid": rows_valid,
            "csv_rows_invalid": rows_invalid,
            "csv_duplicates": duplicates,
            "csv_pct_valid": rows_valid / (rows_valid + rows_invalid) * 100,
            "csv_timestamp": datetime.now().isoformat()
        }
        
        self.influxdb.write_points([{
            "measurement": "fleetyield_csv",
            "tags": {"location": "global"},
            "fields": metrics,
            "time": datetime.now()
        }])
    
    def record_scraper_metrics(self, competitors):
        """Record scraper health per competitor"""
        
        for competitor, success in competitors.items():
            self.influxdb.write_points([{
                "measurement": "fleetyield_scraper",
                "tags": {"competitor": competitor},
                "fields": {
                    "success": 1 if success else 0,
                    "age_hours": get_data_age(competitor),
                    "timestamp": datetime.now().isoformat()
                },
                "time": datetime.now()
            }])
    
    def record_agent_metrics(self, location_id, confidence, rate_change_pct):
        """Record agent decision metrics"""
        
        self.influxdb.write_points([{
            "measurement": "fleetyield_agents",
            "tags": {"location_id": str(location_id)},
            "fields": {
                "confidence": confidence,
                "rate_change_pct": rate_change_pct,
                "timestamp": datetime.now().isoformat()
            },
            "time": datetime.now()
        }])
```

### Dashboard Panels

1. **CSV Data Quality Card**
   - % valid rows (target: >95%)
   - Row count trend (7-day)
   - Duplicate count

2. **Scraper Health (Per-Competitor)**
   - Success rate % (target: >80%)
   - Data age (hours)
   - Last update timestamp

3. **Agent Metrics**
   - Consensus confidence distribution (histogram)
   - Rate change % (min/avg/max)
   - Votes per location

4. **Revenue KPI**
   - Daily revenue trend
   - Occupancy % + trend
   - Net profit (accounting for occupancy cost)

5. **Alerts**
   - Alert count (7-day)
   - Critical/Warning/Info distribution
   - Failure rate trend

---

## Section 5: Slack Health Check Bot

```python
# Every 30 minutes (via cron), send healthcheck notification

@slack_app.message("heartbeat_check")
def handle_heartbeat(message, say):
    """Respond to heartbeat with system status"""
    
    health = get_system_health()
    
    if health['all_ok']:
        emoji = ":green_circle:"
        status_text = "All systems nominal"
    else:
        emoji = ":orange_circle:"
        status_text = f"{health['warning_count']} warnings"
    
    say(f"{emoji} {status_text}")
```

---

**Summary**: Comprehensive monitoring via structured JSON logging, real-time Slack alerting by severity, daily digest, data quality dashboard, and health check integration. No silent failures; all issues escalate to ops within 5 minutes.