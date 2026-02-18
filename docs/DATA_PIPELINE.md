# Data Pipeline: Reservation Parsing, Competitor Scraping, Data Flow

## 1. Nightly Reservation CSV Export

**Source**: Your PMS (Wizard/WAND)  
**Schedule**: 8:00 PM PST nightly  
**Location**: `/data/reservations_YYYY-MM-DD.csv`

### 1.1 Expected CSV Schema

Your current export (from the sample provided) has **100+ columns**. We need these key columns:

```csv
Book Date, Book Time, Cancelled Date, CI Date, Car Class Code, 
Est Total Amt, Res Status, Res Adv Days, Account Segmentation, 
Booking Source, Booking Country, CI Loc Mnemonic
```

### 1.2 Data Processing Pipeline

```python
import pandas as pd
from datetime import datetime, timedelta
import sqlite3

def parse_reservation_csv(csv_path: str) -> dict:
    """Parse nightly reservation CSV and aggregate by car class / booking source."""
    
    df = pd.read_csv(csv_path)
    
    # Filter: only active (non-cancelled) reservations
    df = df[df['Cancelled Date'].isna()]
    
    # Parse dates
    df['CI Date'] = pd.to_datetime(df['CI Date'], format='%m/%d/%Y')
    df['Book Date'] = pd.to_datetime(df['Book Date'], format='%m/%d/%Y')
    
    # Calculate booking lead time (days)
    df['booking_lead_days'] = (df['CI Date'] - df['Book Date']).dt.days
    
    # Aggregate: last 7 days
    last_7d = datetime.now() - timedelta(days=7)
    df = df[df['CI Date'] >= last_7d]
    
    # Group by car class and booking source
    agg = df.groupby(['CI Date', 'Car Class Code', 'Booking Source']).agg({
        'Res Status': 'count',  # Count bookings
        'Est Total Amt': 'sum',  # Total revenue
        'booking_lead_days': 'mean'  # Avg lead time
    }).reset_index()
    
    agg.columns = ['ci_date', 'car_class', 'booking_source', 'bookings', 'revenue', 'avg_lead_days']
    
    return agg.to_dict('records')

def store_reservation_data(records: list):
    """Store aggregated reservation data in SQLite."""
    
    conn = sqlite3.connect('/data/rate_mgmt.db')
    cursor = conn.cursor()
    
    for rec in records:
        cursor.execute("""
            INSERT INTO reservation_cache (date, car_class, booking_source, bookings, revenue, avg_lead_days)
            VALUES (?, ?, ?, ?, ?, ?)
        """, (rec['ci_date'], rec['car_class'], rec['booking_source'], rec['bookings'], rec['revenue'], rec['avg_lead_days']))
    
    conn.commit()
    conn.close()
```

### 1.3 SQLite Schema: `reservation_cache`

```sql
CREATE TABLE reservation_cache (
  id INTEGER PRIMARY KEY,
  date DATE,
  car_class TEXT,  -- E, F, S, L, A, W, etc.
  booking_source TEXT,  -- Costco, Priceline, Budget.com, GDS, etc.
  bookings INTEGER,
  revenue REAL,
  avg_lead_days REAL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index for fast queries
CREATE INDEX idx_date_carclass ON reservation_cache(date, car_class);
```

---

## 2. Competitor Web Scraping

**Schedule**: 8:00 PM - 11:00 PM PST (staggered, 3 companies per hour to avoid rate limits)  
**Competitors**: Hertz, Enterprise, Avis, National, Alamo, Alamo (backup)

### 2.1 Scraper Architecture (Playwright + Celery)

```
┌─────────────────────────────────────────────┐
│ DigitalOcean VM 1 (Worker A) - $5/mo        │
│ Runs: Hertz, National scraper tasks         │
│ 8 PM - 9 PM                                  │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ DigitalOcean VM 2 (Worker B) - $5/mo        │
│ Runs: Enterprise, Avis scraper tasks        │
│ 9 PM - 10 PM                                 │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ DigitalOcean VM 3 (Worker C) - $5/mo        │
│ Runs: Alamo, Budget.com scraper tasks       │
│ 10 PM - 11 PM                                │
└─────────────────────────────────────────────┘

↓ All write to shared Redis or local SQLite ↓

┌─────────────────────────────────────────────┐
│ Main Server: SQLite Database                │
│ /data/rate_mgmt.db → competitor_rates table │
└─────────────────────────────────────────────┘
```

### 2.2 Scraper Script Template (Playwright)

```python
from playwright.async_api import async_playwright
import asyncio
from datetime import datetime
import json
import sqlite3

class CompetitorScraper:
    def __init__(self, competitor: str, location: str):
        self.competitor = competitor  # "hertz", "enterprise", etc.
        self.location = location      # "LAS_AIRPORT", "CENTER_STRIP", etc.
        self.urls = self._get_competitor_urls()
    
    def _get_competitor_urls(self) -> dict:
        """Return booking page URLs by competitor."""
        urls = {
            "hertz": {
                "LAS_AIRPORT": "https://www.hertz.com/rentacar/reservation/index.jsp?targetPage=searchCar",
                "CENTER_STRIP": "https://www.hertz.com/rentacar/reservation/index.jsp",
                # ... other locations
            },
            "enterprise": {
                "LAS_AIRPORT": "https://www.enterprise.com/en/locations/united-states/nv/las-vegas/las-airport.html",
                # ... other locations
            },
            # ... other competitors
        }
        return urls.get(self.competitor, {})
    
    async def scrape(self) -> dict:
        """Scrape competitor rates for all car classes."""
        async with async_playwright() as p:
            browser = await p.chromium.launch(headless=True)
            page = await browser.new_page()
            
            # Set user agent to avoid detection
            await page.set_extra_http_headers({
                'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
            })
            
            results = {}
            
            # Navigate to booking page
            url = self.urls.get(self.location)
            if not url:
                print(f"No URL found for {self.competitor} in {self.location}")
                return results
            
            try:
                await page.goto(url, timeout=30000, wait_until="networkidle")
                
                # Example: scrape rate table (CSS selectors differ by competitor)
                if self.competitor == "hertz":
                    results = await self._scrape_hertz(page)
                elif self.competitor == "enterprise":
                    results = await self._scrape_enterprise(page)
                # ... etc.
                
            except Exception as e:
                print(f"Error scraping {self.competitor}: {e}")
            finally:
                await browser.close()
            
            return results
    
    async def _scrape_hertz(self, page) -> dict:
        """Hertz-specific scraping logic."""
        # Wait for rates to load
        await page.wait_for_selector('.car-class-row', timeout=10000)
        
        # Extract car class + rates
        rows = await page.query_selector_all('.car-class-row')
        results = {}
        
        for row in rows:
            car_class = await row.query_selector('.class-name').text_content()
            daily_rate = await row.query_selector('.daily-rate').text_content()
            weekly_rate = await row.query_selector('.weekly-rate').text_content()
            
            # Parse rates (remove $, commas)
            daily_rate = float(daily_rate.replace('$', '').replace(',', ''))
            weekly_rate = float(weekly_rate.replace('$', '').replace(',', ''))
            
            results[car_class] = {
                'daily_rate': daily_rate,
                'weekly_rate': weekly_rate,
                'timestamp': datetime.now().isoformat()
            }
        
        return results
    
    async def _scrape_enterprise(self, page) -> dict:
        """Enterprise-specific scraping logic."""
        # Different selectors for Enterprise
        await page.wait_for_selector('[data-car-class]', timeout=10000)
        
        rows = await page.query_selector_all('[data-car-class]')
        results = {}
        
        for row in rows:
            car_class = await row.get_attribute('data-car-class')
            daily_rate = await row.query_selector('.price-daily').text_content()
            
            daily_rate = float(daily_rate.replace('$', '').strip())
            
            results[car_class] = {
                'daily_rate': daily_rate,
                'timestamp': datetime.now().isoformat()
            }
        
        return results

async def scrape_and_store(competitor: str, location: str):
    """Scrape and store competitor rates."""
    scraper = CompetitorScraper(competitor, location)
    rates = await scraper.scrape()
    
    # Store in SQLite
    conn = sqlite3.connect('/data/rate_mgmt.db')
    cursor = conn.cursor()
    
    for car_class, pricing in rates.items():
        cursor.execute("""
            INSERT INTO competitor_rates (date, competitor, location, car_class, daily_rate, weekly_rate, timestamp)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, (
            datetime.now().date(),
            competitor,
            location,
            car_class,
            pricing.get('daily_rate'),
            pricing.get('weekly_rate'),
            datetime.now()
        ))
    
    conn.commit()
    conn.close()
    print(f"Stored {len(rates)} car classes for {competitor} at {location}")

# Celery task
from celery import Celery

app = Celery('competitor_scraper', broker='redis://localhost:6379')

@app.task
def scrape_competitor_task(competitor: str, location: str):
    """Celery task wrapper."""
    asyncio.run(scrape_and_store(competitor, location))

# Cron schedule (8 PM - 11 PM, staggered)
# 8:00 PM: hertz, national (VM A)
# 9:00 PM: enterprise, avis (VM B)
# 10:00 PM: alamo (VM C)
```

### 2.3 SQLite Schema: `competitor_rates`

```sql
CREATE TABLE competitor_rates (
  id INTEGER PRIMARY KEY,
  date DATE,
  competitor TEXT,  -- "hertz", "enterprise", "avis", etc.
  location TEXT,    -- "LAS_AIRPORT", "CENTER_STRIP", etc.
  car_class TEXT,   -- "E", "F", "S", etc.
  daily_rate REAL,
  weekly_rate REAL,
  timestamp TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index for fast lookups
CREATE INDEX idx_competitor_location_carclass ON competitor_rates(competitor, location, car_class, date);
```

---

## 3. Current Rates (From Your PMS)

**Source**: Wizard/WAND API or SSH export  
**Schedule**: Fresh load at 5:55 AM (before agent run)  
**Endpoint**: `/api/rates?location={location}&date={date}` or direct DB query

### 3.1 Rate Fetch Script

```python
import requests
import json
from datetime import datetime

class RateFetcher:
    def __init__(self, pms_api_url: str, api_token: str):
        self.base_url = pms_api_url
        self.headers = {'Authorization': f'Bearer {api_token}'}
    
    def get_current_rates(self, location: str) -> dict:
        """Fetch current rates for a location."""
        url = f"{self.base_url}/rates?location={location}&date={datetime.now().date()}"
        
        response = requests.get(url, headers=self.headers)
        response.raise_for_status()
        
        rates = response.json()
        
        # Normalize to our format
        normalized = {}
        for car_class in rates.get('car_classes', []):
            normalized[car_class['code']] = {
                'daily_rate': car_class['daily_rate'],
                'weekly_rate': car_class.get('weekly_rate'),
                'last_updated': datetime.now().isoformat()
            }
        
        return normalized

# Usage
fetcher = RateFetcher('https://wizard-pms.budgetvegas.com/api', 'YOUR_API_TOKEN')
rates = fetcher.get_current_rates('LAS_AIRPORT')
print(json.dumps(rates, indent=2))
# Output:
# {
#   "E": {"daily_rate": 42.00, "weekly_rate": 35.00, "last_updated": "2026-02-18T05:55:00"},
#   "F": {"daily_rate": 48.00, "weekly_rate": 40.00, "last_updated": "2026-02-18T05:55:00"},
#   ...
# }
```

---

## 4. Fleet Inventory (Optional)

**Source**: Your GLPI or PMS  
**Schedule**: Hourly snapshots (aggregate at 5:55 AM)  
**Data**: available, reserved, capacity per car class per location

### 4.1 Inventory Query

```sql
-- From GLPI or inventory DB
SELECT 
  location,
  car_class,
  COUNT(*) as total_fleet,
  SUM(CASE WHEN status = 'available' THEN 1 ELSE 0 END) as available,
  SUM(CASE WHEN status = 'reserved' THEN 1 ELSE 0 END) as reserved,
  ROUND(100.0 * SUM(CASE WHEN status = 'reserved' THEN 1 ELSE 0 END) / COUNT(*), 2) as utilization_pct
FROM vehicle_inventory
WHERE location = 'LAS_AIRPORT'
  AND car_class IN ('E', 'F', 'S', 'L', 'A', 'W')
GROUP BY location, car_class;
```

---

## 5. Master Orchestrator Data Assembly (6:00 AM)

```python
import sqlite3
import json
from datetime import datetime, timedelta

def assemble_agent_input(location: str) -> dict:
    """Assemble all data for location agents at 6 AM."""
    
    conn = sqlite3.connect('/data/rate_mgmt.db')
    cursor = conn.cursor()
    
    # 1. Reservation data (last 7 days)
    cursor.execute("""
        SELECT car_class, SUM(bookings) as total_bookings, SUM(revenue) as total_revenue
        FROM reservation_cache
        WHERE location = ? AND date >= date('now', '-7 days')
        GROUP BY car_class
    """, (location,))
    
    reservation_data = {}
    for row in cursor.fetchall():
        car_class, bookings, revenue = row
        reservation_data[car_class] = {'bookings': bookings, 'revenue': revenue}
    
    # 2. Competitor rates (latest snapshot)
    cursor.execute("""
        SELECT car_class, competitor, daily_rate
        FROM competitor_rates
        WHERE location = ? AND date = date('now')
        GROUP BY car_class, competitor
    """, (location,))
    
    competitor_data = {}
    for car_class_code, comp, rate in cursor.fetchall():
        if car_class_code not in competitor_data:
            competitor_data[car_class_code] = {}
        competitor_data[car_class_code][comp] = rate
    
    # 3. Current rates (from PMS)
    current_rates = get_current_rates(location)
    
    # 4. Inventory (from GLPI)
    cursor.execute("""
        SELECT car_class, available, reserved, capacity
        FROM inventory_snapshot
        WHERE location = ? AND date = date('now')
    """, (location,))
    
    inventory_data = {}
    for car_class, avail, reserved, capacity in cursor.fetchall():
        inventory_data[car_class] = {
            'available': avail,
            'reserved': reserved,
            'capacity': capacity,
            'utilization_pct': round(100.0 * reserved / capacity, 2)
        }
    
    # 5. Constraints
    constraints = {
        'margin_floor_pct': 0.25,
        'cost_per_vehicle_day': {'E': 28, 'F': 32, 'S': 35, 'L': 40, 'A': 45, 'W': 50},
        'max_rate_increase_pct': 0.15,
        'competitor_parity_tolerance_pct': 0.10
    }
    
    # Assemble
    agent_input = {
        'location': location,
        'date': datetime.now().isoformat(),
        'reservation_data': reservation_data,
        'competitor_snapshot': competitor_data,
        'current_rates': current_rates,
        'inventory': inventory_data,
        'constraints': constraints
    }
    
    conn.close()
    
    return agent_input

# Example output:
# {
#   "location": "LAS_AIRPORT",
#   "date": "2026-02-18T05:55:00",
#   "reservation_data": {
#     "E": {"bookings": 120, "revenue": 4800},
#     "F": {"bookings": 85, "revenue": 3570}
#   },
#   "competitor_snapshot": {
#     "E": {"hertz": 49.99, "enterprise": 44.99, "avis": 47.99},
#     "F": {"hertz": 55.99, "enterprise": 51.99}
#   },
#   "current_rates": {
#     "E": 42.00,
#     "F": 48.00
#   },
#   "inventory": {
#     "E": {"available": 15, "reserved": 42, "capacity": 60, "utilization_pct": 70.0},
#     "F": {"available": 8, "reserved": 28, "capacity": 40, "utilization_pct": 70.0}
#   },
#   "constraints": {...}
# }
```

---

## 6. Daily Cron Schedule

```bash
# /etc/cron.d/rate_mgmt

# 5:45 AM - Parse nightly reservation CSV
45 5 * * * python /app/cron/parse_reservations.py >> /var/log/rate_mgmt/parse_res.log 2>&1

# 8:00 PM - Start competitor scraper (3 workers, staggered)
0 20 * * * python /app/cron/scraper_runner.py hertz national >> /var/log/rate_mgmt/scraper.log 2>&1
0 21 * * * python /app/cron/scraper_runner.py enterprise avis >> /var/log/rate_mgmt/scraper.log 2>&1
0 22 * * * python /app/cron/scraper_runner.py alamo >> /var/log/rate_mgmt/scraper.log 2>&1

# 5:55 AM - Load current rates from PMS
55 5 * * * python /app/cron/load_pms_rates.py >> /var/log/rate_mgmt/load_rates.log 2>&1

# 6:00 AM - Run main orchestrator (master + 8 location agents)
0 6 * * * python /app/cron/daily_run.py >> /var/log/rate_mgmt/daily_run.log 2>&1
```

---

## 7. Error Handling & Fallbacks

| Scenario | Action |
|----------|--------|
| **Scraper fails** | Use 7-day rolling average of competitor rates |
| **CSV delayed (>6h old)** | Skip agent run, log warning, retry at 6:30 AM |
| **PMS API down** | Use yesterday's rates as fallback |
| **Inventory data missing** | Assume 70% utilization (neutral) |
| **Agent timeout** | Use previous recommendation + log |

---

**Next**: Implement Week 1-2 foundation with these data pipeline steps.
