# DATA VALIDATION FRAMEWORK - FleetYield ETL Hardening

## Section 1: Validation Rules Table

| Field | Type | Required | Valid Range | Invalid Example | Action |
|-------|------|----------|-------------|-----------------|--------|
| CI_Date | DATE | YES | 0-365 days from today | '2027-12-31' | REJECT row |
| Car_Class_Code | STRING | YES | {E,F,S,L,A,W,X,B,V,C} | 'Z' | REJECT row |
| Booking_Source | STRING | YES | {Budget.com, Costco, Priceline, GDS, ...} | 'Unknown' | REJECT row |
| Est_Total_Amt | DECIMAL | YES | $10-$500 | -$100 or $5000 | REJECT row |
| Book_Date | DATE | YES | <=CI_Date | NULL | REJECT row |
| Cancelled_Date | DATE | NO | NULL or >=CI_Date | Before CI_Date | WARN, log |
| Customer_ID | STRING | YES | Non-empty | NULL | REJECT row |

---

## Section 2: Python Validator Code Examples

### Class-Based Validator

```python
import pandas as pd
import hashlib
from datetime import datetime, timedelta

class CSVValidator:
    """Validates FleetYield CSV input before parsing"""
    
    def __init__(self, filepath):
        self.filepath = filepath
        self.errors = []
        self.warnings = []
    
    def validate_schema(self):
        """Check required columns exist"""
        df = pd.read_csv(self.filepath)
        required_cols = ['CI_Date', 'Car_Class_Code', 'Booking_Source', 'Est_Total_Amt', 'Book_Date', 'Customer_ID']
        missing = [col for col in required_cols if col not in df.columns]
        
        if missing:
            self.errors.append(f"Schema validation FAILED: Missing columns {missing}")
            return False
        return True
    
    def validate_row_types(self):
        """Check data types"""
        df = pd.read_csv(self.filepath)
        try:
            df['CI_Date'] = pd.to_datetime(df['CI_Date'], errors='coerce')
            df['Est_Total_Amt'] = pd.to_numeric(df['Est_Total_Amt'], errors='coerce')
            if df['CI_Date'].isna().sum() > 0:
                self.errors.append(f"Type validation FAILED: {df['CI_Date'].isna().sum()} rows have invalid dates")
                return False
        except Exception as e:
            self.errors.append(f"Type validation FAILED: {str(e)}")
            return False
        return True
    
    def validate_bounds(self):
        """Check numeric and date bounds"""
        df = pd.read_csv(self.filepath, parse_dates=['CI_Date', 'Book_Date'])
        today = pd.Timestamp(datetime.today())
        
        # Revenue bounds: $10-$500
        invalid_revenue = df[(df['Est_Total_Amt'] < 10) | (df['Est_Total_Amt'] > 500)]
        if len(invalid_revenue) > 0:
            self.errors.append(f"Revenue bounds FAILED: {len(invalid_revenue)} rows have invalid amounts")
        
        # Date bounds: CI_Date within 365 days
        invalid_dates = df[(df['CI_Date'] > today + timedelta(days=365)) | (df['CI_Date'] < today)]
        if len(invalid_dates) > len(df) * 0.05:  # >5% invalid = fail
            self.errors.append(f"Date bounds FAILED: {len(invalid_dates)} rows have invalid CI_Date")
        
        # Book_Date <= CI_Date
        invalid_booking = df[df['Book_Date'] > df['CI_Date']]
        if len(invalid_booking) > 0:
            self.errors.append(f"Booking logic FAILED: {len(invalid_booking)} rows have Book_Date > CI_Date")
        
        return len(self.errors) == 0
    
    def validate_car_classes(self):
        """Check car class codes"""
        df = pd.read_csv(self.filepath)
        valid_classes = {'E', 'F', 'S', 'L', 'A', 'W', 'X', 'B', 'V', 'C'}
        invalid = df[~df['Car_Class_Code'].isin(valid_classes)]
        
        if len(invalid) > 0:
            self.errors.append(f"Car class validation FAILED: {len(invalid)} rows have invalid car classes")
            return False
        return True
    
    def detect_duplicates(self):
        """Check for duplicate rows"""
        df = pd.read_csv(self.filepath)
        
        # Create row hash (combination of key fields)
        df['_row_hash'] = df.apply(
            lambda row: hashlib.md5(
                f"{row['Book_Date']}{row['CI_Date']}{row['Car_Class_Code']}{row['Est_Total_Amt']}".encode()
            ).hexdigest(),
            axis=1
        )
        
        dupes = df[df.duplicated(subset=['_row_hash'], keep=False)]
        if len(dupes) > 0:
            self.warnings.append(f"Duplicates detected: {len(dupes)} rows appear to be duplicates")
            return False
        return True
    
    def run_all_validations(self):
        """Execute full validation suite"""
        results = {
            'schema_valid': self.validate_schema(),
            'types_valid': self.validate_row_types(),
            'bounds_valid': self.validate_bounds(),
            'classes_valid': self.validate_car_classes(),
            'no_dupes': self.detect_duplicates(),
        }
        
        if not all(results.values()):
            return False, self.errors + self.warnings
        return True, []


# Usage
validator = CSVValidator('/data/reservations_2026-02-18.csv')
is_valid, issues = validator.run_all_validations()

if not is_valid:
    log_error(f"CSV validation failed: {issues}")
    use_7day_cache()  # fallback
else:
    process_csv()
```

---

## Section 3: Fallback Strategy

### Decision Tree

```
CSV received?
├─ NO (file missing) → Use 7-day rolling average + alert ops
└─ YES → Check MD5 hash (matches expected)
    ├─ NO (corruption detected) → Reject file + use 7-day average
    └─ YES → Run validation suite
        ├─ SCHEMA invalid → Reject + use 7-day average
        ├─ >10% rows invalid → Reject + use 7-day average
        ├─ >5% invalid rows → Process with warnings + log
        └─ <5% invalid → Skip invalid rows + process valid ones
```

### Action Map

| Scenario | Action | Log Level | Alert? |
|----------|--------|-----------|---------|
| CSV missing | Use 7-day avg | ERROR | YES (ops) |
| MD5 mismatch | Reject file | ERROR | YES (ops) |
| Schema invalid | Use yesterday's rates | ERROR | YES (ops) |
| >10% invalid rows | Use 7-day average | ERROR | YES (ops) |
| 5-10% invalid rows | Skip bad rows, process good | WARNING | YES (ops) |
| <5% invalid rows | Skip bad rows, process good | INFO | NO |
| All validations pass | Normal processing | INFO | NO |

---

## Section 4: Data Quality Dashboard Metrics

### Track Daily

```json
{
  "date": "2026-02-18",
  "csv_metrics": {
    "rows_received": 6750,
    "rows_valid": 6710,
    "rows_invalid": 40,
    "pct_invalid": 0.59,
    "null_count": {"CI_Date": 0, "Car_Class_Code": 5, "Est_Total_Amt": 2},
    "duplicate_rows": 3,
    "md5_hash": "abc123def456",
    "freshness_hours": 1.2
  },
  "scraper_metrics": {
    "competitors": [
      {"name": "Hertz", "success": true, "age_hours": 2, "rate_count": 48},
      {"name": "Enterprise", "success": false, "age_hours": 26, "fallback": "7day_avg"},
      {"name": "Avis", "success": true, "age_hours": 2, "rate_count": 48}
    ],
    "overall_success_rate": 0.83
  },
  "agent_metrics": {
    "agents_ran": 8,
    "votes_received": 8,
    "consensus_confidence": 0.87,
    "rate_change_pct": 3.2,
    "published": true
  },
  "occupancy_metrics": {
    "today": 0.82,
    "yesterday": 0.84,
    "delta_pct": -2.4,
    "alert": false
  }
}
```

### Dashboard UI (Grafana / Flask)
- **Data Quality Card**: % valid rows, duplicate count, freshness age
- **Scraper Health**: Per-competitor success rate, data age, fallback status
- **Agent Metrics**: Consensus quality, rate change, confidence distribution
- **Revenue KPI**: ASP trend, occupancy trend, net profit (accounting for both)

---

## Section 5: SLA & Alerts

### Critical SLAs

```yaml
csv_arrival_sla:
  deadline: "08:00 PM PST"
  alert_if_missing: "08:15 PM"
  fallback: "7-day rolling average"
  severity: "CRITICAL"

data_quality_sla:
  max_invalid_pct: 10%
  alert_if_exceeded: true
  severity: "WARNING"
  action: "Process with caution; log for review"

scraper_sla:
  min_success_rate: 80%
  alert_if_below: true
  severity: "MEDIUM"
  action: "Reduce competitive_score weight; use 7-day fallback"

freshness_sla:
  max_age: 24 hours
  alert_if_exceeded: true
  severity: "HIGH"
  action: "Skip agent run; use previous day rates"
```

### Alert Messages

```
🚨 CRITICAL: CSV validation failed (20% invalid rows). Using 7-day rolling average. Manual review required.
  File: /data/reservations_2026-02-18.csv
  Issues: 1350 rows invalid (missing CI_Date, negative revenue)
  Action: Fix CSV; re-upload for processing
  Link: <spreadsheet-review-url>

⚠️ WARNING: Data quality degrading (8% invalid rows). Monitoring.
  Recommendation: Check PMS export settings; possible schema change
  
⚠️ WARNING: Scraper success 75% (below 80% SLA). Hertz and Avis timed out.
  Using 7-day competitor average for voting.
  Action: Check if competitors blocked IPs; retry tomorrow
```

---

**Summary**: Comprehensive validation framework prevents bad data from reaching agents. Multi-tiered fallback strategy ensures system resilience. Dashboard provides real-time visibility into data quality + operational health.