# PMS INTEGRATION HARDENING - Atomic Transactions & Safety

## Section 1: Atomic Transaction Architecture

### Transaction Flow (Pseudocode)

```python
def push_rates_atomic(rates_dict, location_id, dry_run=False):
    """
    Atomically push rates to PMS with full rollback capability
    
    rates_dict: {car_class: new_rate, ...}
    location_id: which location (1=LAS_AIRPORT, etc)
    dry_run: if True, log proposed but don't push
    """
    
    transaction_id = generate_uuid()
    timestamp = datetime.now()
    
    try:
        # PHASE 1: PRE-FLIGHT VALIDATION
        log(f"[{transaction_id}] Starting rate push for {location_id}")
        
        if not validate_pms_connectivity():
            raise PMS_CONNECTION_ERROR("PMS API unreachable")
        
        for car_class, rate in rates_dict.items():
            if not validate_rate(rate):  # Check: rate > margin_floor, rate <= previous*1.15
                raise RATE_VALIDATION_ERROR(f"{car_class}: {rate} violates constraints")
        
        log(f"[{transaction_id}] Pre-flight validation PASSED")
        
        # PHASE 2: DRY RUN (if enabled)
        if dry_run:
            log(f"[{transaction_id}] DRY RUN: Would push rates {rates_dict}")
            log_to_rate_history(
                transaction_id=transaction_id,
                location_id=location_id,
                rates=rates_dict,
                status="PROPOSED_DRY_RUN",
                timestamp=timestamp
            )
            return True
        
        # PHASE 3: PUSH TO PMS (with retry logic)
        for attempt in range(3):
            try:
                response = pms_api.update_rates(location_id, rates_dict)
                if response.status == "OK":
                    log(f"[{transaction_id}] PMS write SUCCEEDED on attempt {attempt+1}")
                    break
            except PMS_API_TIMEOUT:
                if attempt < 2:
                    wait(5 ** (attempt + 1))  # 5s, 25s, 125s exponential backoff
                    continue
                raise PMS_API_ERROR("PMS timeout after 3 retries")
            except PMS_VALIDATION_ERROR as e:
                raise RATE_VALIDATION_ERROR(f"PMS rejected rates: {e}")
        
        # PHASE 4: VERIFY (read-back from PMS)
        log(f"[{transaction_id}] Verifying rates in PMS...")
        current_rates = pms_api.get_rates(location_id)
        
        for car_class, expected_rate in rates_dict.items():
            actual_rate = current_rates.get(car_class)
            if abs(actual_rate - expected_rate) > 0.01:  # $0.01 tolerance
                raise VERIFICATION_ERROR(
                    f"{car_class}: expected {expected_rate}, got {actual_rate}"
                )
        
        log(f"[{transaction_id}] Verification PASSED")
        
        # PHASE 5: COMMIT (log to rate_history)
        log_to_rate_history(
            transaction_id=transaction_id,
            location_id=location_id,
            rates=rates_dict,
            status="PUBLISHED",
            timestamp=timestamp,
            previous_rates=get_previous_rates(location_id)
        )
        
        log(f"[{transaction_id}] Transaction COMMITTED")
        return True
        
    except Exception as e:
        # ROLLBACK
        log(f"[{transaction_id}] ERROR: {str(e)}")
        log(f"[{transaction_id}] Rolling back...")
        
        try:
            previous_rates = get_previous_rates(location_id)
            pms_api.update_rates(location_id, previous_rates)
            log(f"[{transaction_id}] Rollback SUCCEEDED")
        except:
            log(f"[{transaction_id}] Rollback FAILED - MANUAL INTERVENTION REQUIRED")
            alert_ops(f"PMS rate push FAILED. Manual intervention required. txn_id={transaction_id}")
        
        log_to_rate_history(
            transaction_id=transaction_id,
            location_id=location_id,
            status="ROLLED_BACK",
            error=str(e),
            timestamp=timestamp
        )
        
        return False
```

---

## Section 2: Pre-Flight Validation Checklist

### Validation Steps (5 minutes before push)

```python
def run_preflight_checks(location_id):
    """Execute all pre-flight validations"""
    
    checks = {
        'pms_connectivity': False,
        'rate_sanity': False,
        'inventory_freshness': False,
        'database_writeable': False,
        'no_active_push': False
    }
    
    # Check 1: PMS API Health
    try:
        pms_api.health_check(timeout=5)
        checks['pms_connectivity'] = True
        log("✓ PMS API is reachable")
    except:
        log("✗ PMS API unreachable")
        alert_ops("PMS down; aborting rate push")
        return False
    
    # Check 2: Rate Sanity
    proposed_rates = get_agent_consensus(location_id)
    previous_rates = get_previous_rates(location_id)
    
    for car_class, new_rate in proposed_rates.items():
        prev_rate = previous_rates[car_class]
        
        # No >20% daily spike
        if (new_rate / prev_rate - 1) > 0.20:
            log(f"✗ {car_class}: spike {new_rate}/{prev_rate} > 20%")
            alert_ops(f"Rate spike detected: {car_class} {prev_rate}→{new_rate}. Manual review required.")
            return False
        
        # All rates > margin floor
        margin_floor = get_margin_floor(car_class)
        if new_rate < margin_floor:
            log(f"✗ {car_class}: {new_rate} < margin floor {margin_floor}")
            return False
    
    checks['rate_sanity'] = True
    log("✓ Rates pass sanity checks")
    
    # Check 3: Inventory Freshness
    inventory_age = datetime.now() - get_inventory_timestamp()
    if inventory_age > timedelta(hours=2):
        log(f"✗ Inventory snapshot is {inventory_age} old (>2h threshold)")
        return False
    
    checks['inventory_freshness'] = True
    log("✓ Inventory is fresh")
    
    # Check 4: Database Writeable
    try:
        test_write_to_rate_history()
        checks['database_writeable'] = True
        log("✓ Database is writeable")
    except:
        log("✗ Database is not writeable")
        return False
    
    # Check 5: No Active Push
    if is_push_in_progress(location_id):
        log("✗ Previous push still in progress")
        return False
    
    checks['no_active_push'] = True
    log("✓ No active push in progress")
    
    return all(checks.values())
```

---

## Section 3: Dry-Run Mode Spec

### Enable Dry-Run (Week 3-4, Pre-Production)

```bash
# Run agents, generate rates, but DON'T push to PMS
cron_schedule_dry_run="0 6 * * *"  # 6 AM daily

./fleetyield_agents.py \
  --location LAS_AIRPORT \
  --dry_run true \
  --compare_to_manual true
```

### Output: Comparison Report

```
DRY RUN: 2026-02-18 6:00 AM
Location: LAS_AIRPORT

Agent Recommendations (NOT PUBLISHED):
  E-class: $46.50 (current: $46.00, +1.09%)
  F-class: $48.20 (current: $48.00, +0.42%)
  S-class: $51.80 (current: $51.00, +1.57%)

Manual Rates (Human decision):
  E-class: $46.50 (✓ match)
  F-class: $48.00 (conflict: agent $48.20 vs manual $48.00)
  S-class: $52.00 (conflict: agent $51.80 vs manual $52.00)

Accuracy: 1/3 classes (33%)
Confidence Avg: 78%

Analysis: Agent underprices S-class; consider increasing demand confidence weight
```

### 1-Week Dry-Run Validation

Run dry-run for 7 days (Mon-Sun):
- Compare agent recommendations vs. manual rates
- Calculate agreement rate (target: >80% match within $2)
- Identify systematic biases (agent always over/under-prices certain classes)
- Adjust agent prompts/weights based on feedback
- When accuracy >85% for 3+ consecutive days, proceed to live push

---

## Section 4: Error Recovery Procedures

### Network Timeout Recovery

```python
def handle_timeout(transaction_id, attempt, max_retries=3):
    """Exponential backoff on API timeout"""
    
    if attempt < max_retries:
        wait_seconds = 5 ** attempt  # 5s, 25s, 125s
        log(f"[{transaction_id}] Timeout on attempt {attempt}; retrying in {wait_seconds}s")
        wait(wait_seconds)
        return retry_push(transaction_id, attempt + 1)
    else:
        log(f"[{transaction_id}] Max retries exceeded; rolling back")
        return rollback(transaction_id)
```

### PMS Rejection Recovery

```python
def handle_pms_rejection(transaction_id, error_code):
    """PMS returned validation error"""
    
    if error_code == "RATE_OUT_OF_RANGE":
        alert_ops(f"PMS rejected rates (out of range). Manual override needed. txn={transaction_id}")
        # Hold for manual review; don't rollback
        log_to_rate_history(status="PENDING_MANUAL_REVIEW", error=error)
    
    elif error_code == "LOCATION_NOT_FOUND":
        log(f"[{transaction_id}] PMS location invalid; check configuration")
        alert_ops("PMS location misconfiguration")
        return rollback(transaction_id)
    
    else:
        log(f"[{transaction_id}] Unknown PMS error: {error_code}")
        return rollback(transaction_id)
```

### Partial Write Detection

```python
def verify_partial_write(transaction_id):
    """Detect if some rates updated but others didn't"""
    
    published_rates = get_rates_from_pms()
    expected_rates = get_from_rate_history(transaction_id, status="IN_PROGRESS")
    
    mismatches = []
    for car_class in expected_rates:
        if published_rates[car_class] != expected_rates[car_class]:
            mismatches.append(car_class)
    
    if mismatches:
        log(f"[{transaction_id}] Partial write detected: {mismatches} not updated")
        log(f"[{transaction_id}] Rolling back...")
        rollback_and_revert_all(transaction_id)
        alert_ops(f"CRITICAL: Partial write detected. Rollback executed. Manual check required.")
        return False
    
    return True
```

---

## Section 5: Python Pseudocode

```python
# Complete transaction wrapper

class PMSTransactionManager:
    def __init__(self, pms_client, db_client):
        self.pms = pms_client
        self.db = db_client
    
    def push_rates(self, location_id, rates_dict, dry_run=False):
        txn_id = self._generate_txn_id()
        try:
            self._log(f"Starting transaction {txn_id}")
            
            # Pre-flight
            if not self._preflight_checks(location_id):
                return False
            
            # Dry-run branch
            if dry_run:
                self._log_dry_run(txn_id, rates_dict)
                return True
            
            # Push with retry
            self._push_with_retry(txn_id, location_id, rates_dict)
            
            # Verify
            if not self._verify_write(txn_id, rates_dict):
                self._rollback(txn_id)
                return False
            
            # Commit
            self._log_to_db(txn_id, status="PUBLISHED", rates=rates_dict)
            self._log(f"Transaction {txn_id} COMMITTED")
            return True
        
        except Exception as e:
            self._log(f"Transaction {txn_id} FAILED: {e}")
            self._rollback(txn_id)
            self._alert_ops(f"Rate push failed. txn={txn_id}")
            return False
```

---

**Summary**: Atomic transactions + pre-flight validation + verification + rollback ensure PMS integration is safe and recoverable. Dry-run mode allows validation before production deployment.