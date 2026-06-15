# Code Examples & Patterns

Common patterns and implementation strategies used in the system.

---

## Table of Contents

1. [Connection Pooling](#connection-pooling)
2. [Atomic Transactions](#atomic-transactions)
3. [Error Handling](#error-handling)
4. [Resource Management](#resource-management)
5. [Data Validation](#data-validation)
6. [Browser Automation](#browser-automation)
7. [Integrity Checking](#integrity-checking)

---

## Connection Pooling

### Pattern: Persistent Connection Pool

**Problem:** Creating new database connections for each operation is expensive.

**Solution:** Maintain a small pool of persistent connections that are reused.

**Conceptual Approach:**

```
Initialize pool with N connections (e.g., 2-3)
┌─────────────────┐
│ Connection 1    │ (idle or in-use)
│ Connection 2    │ (idle or in-use)
│ Connection 3    │ (idle or in-use)
└─────────────────┘

For each operation:
1. Get connection from pool
2. Execute query
3. Return connection to pool (still open, ready for reuse)
```

**Benefits:**
- Reduced connection overhead (no setup/teardown per operation)
- Respects external connection quotas
- Better resource utilization
- Improved throughput

**Configuration:**
```
Pool size depends on:
- External quota (e.g., 500 connections/hour)
- Concurrent operations
- Memory constraints
```

---

## Atomic Transactions

### Pattern: All-or-Nothing Updates

**Problem:** Partial updates could leave related data inconsistent.

**Solution:** Group all related database operations into single transaction. Either all succeed or all fail.

**Conceptual Flow:**

```
BEGIN TRANSACTION
  │
  ├─ Insert into Entity 1
  ├─ Insert into Entity 2
  ├─ Insert into Entity 3
  ├─ Insert into Entity 4
  └─ Insert into Entity 5
  │
  If all succeed: COMMIT (all visible)
  If any fail: ROLLBACK (none visible)
```

**Benefit:** No orphaned records or inconsistent state.

**Pseudocode Pattern:**

```
try {
    start_transaction()
    insert_primary_entity(data)
    insert_related_entity_1(data)
    insert_related_entity_2(data)
    insert_related_entity_3(data)
    insert_related_entity_4(data)
    commit()
} catch (error) {
    rollback()
    log_error(error)
    raise error
}
```

---

## Error Handling

### Pattern 1: Exception Hierarchy

**Concept:** Different error types get different handling.

```
Exception
  ├─ ConnectionError
  │   └─ (retry with backoff)
  ├─ RateLimitError
  │   └─ (trigger maximum backoff)
  ├─ ValidationError
  │   └─ (log and continue)
  ├─ DataIntegrityError
  │   └─ (alert and halt)
  └─ ConfigurationError
      └─ (exit immediately)
```

### Pattern 2: Retry with Backoff

**Concept:** Transient errors are retried with increasing delays.

```
Attempt 1: Try immediately
  │ Fails
Attempt 2: Wait 1 second, retry
  │ Fails
Attempt 3: Wait 2 seconds, retry
  │ Fails
Attempt 4: Wait 4 seconds, retry
  │ Fails
Give up: Log failure, move on
```

**Pseudocode:**

```
def retry_with_backoff(operation, max_attempts=4):
    for attempt in range(1, max_attempts + 1):
        try:
            return operation()
        except TransientError as e:
            if attempt == max_attempts:
                raise
            wait_time = 2 ** (attempt - 1)
            sleep(wait_time)
```

---

## Resource Management

### Pattern: Context Manager (Try-Finally)

**Problem:** Resources might leak if exceptions occur during use.

**Solution:** Guarantee cleanup regardless of how the function exits.

**Conceptual Pattern:**

```
try {
    acquire_resource() (e.g., database connection)
    do_work()         (might throw exception)
    return result
} finally {
    release_resource() (always executed, even if exception)
}
```

**Benefit:** No resource leaks, clean error propagation.

**Pseudocode:**

```
def with_database_connection(operation):
    connection = get_connection_from_pool()
    try:
        result = operation(connection)
        return result
    finally:
        connection.close()  # Always runs
```

---

## Data Validation

### Pattern 1: Input Validation

**Concept:** Validate data immediately after extraction.

```
Raw data from portal
    ↓ Validate
1. Required fields present?
2. Field types correct?
3. Field values in valid range?
4. Format matches expected pattern?
    ↓
Valid data or Validation error
```

**Pseudocode:**

```
def validate_record(data):
    errors = []
    
    if not data.get("field1"):
        errors.append("field1 required")
    
    if not isinstance(data.get("field2"), int):
        errors.append("field2 must be integer")
    
    if data.get("field2", 0) < 0:
        errors.append("field2 must be positive")
    
    if errors:
        raise ValidationError(errors)
    
    return data
```

### Pattern 2: Deduplication

**Concept:** Skip records already in database.

```
Load existing IDs into memory (fast set/dict lookup)
    ↓
For each extracted record:
    ├─ Check if ID exists in set? (O(1) lookup)
    ├─ Yes: Skip (already have it)
    └─ No: Process and store
```

**Benefit:** Avoid duplicate data and reduce database operations.

---

## Browser Automation

### Pattern: Explicit Waits

**Problem:** Elements might not be immediately available (JavaScript might still be running).

**Solution:** Wait for specific conditions before interacting with elements.

**Concept:**

```
Click element
    ↓ Wait for expected condition
    ├─ Element visible?
    ├─ Element clickable?
    ├─ Page loaded?
    └─ (Timeout after N seconds)
    ↓
Proceed or Timeout error
```

**Pseudocode:**

```
def safe_click(browser, element_locator, timeout=10):
    # Wait for element to be clickable (not just visible)
    wait = WebDriverWait(browser, timeout)
    element = wait.until(
        expected_condition_clickable(element_locator)
    )
    element.click()
```

### Pattern: Error Recovery

**Concept:** Screenshot failures for debugging, then recover.

```
Try operation
    ↓ Fails
Take screenshot (preserve failure state)
Log error details
Attempt recovery (retry, fallback, or skip)
```

---

## Integrity Checking

### Pattern: Referential Integrity Validation

**Concept:** Ensure all relationships are consistent.

```
For each record in Entity 1:
    ├─ Does corresponding record exist in Entity 2? YES/NO
    ├─ Does corresponding record exist in Entity 3? YES/NO
    ├─ Does corresponding record exist in Entity 4? YES/NO
    └─ Does corresponding record exist in Entity 5? YES/NO

If any NO: Inconsistency detected
    → Log issue
    → Mark for repair
    → Report statistics
```

### Pattern: Integrity Repair

**Concept:** Dry-run capability before making changes.

```
Dry Run Mode (Detect only):
    ├─ Find inconsistencies
    ├─ Report what would be fixed
    └─ Don't change anything

Production Mode (Fix):
    ├─ Find inconsistencies
    ├─ Apply repairs
    └─ Audit trail for changes
```

---

## Common Implementation Tips

### 1. Configuration
- Load from environment variables
- Validate on startup
- Document all options
- Provide sensible defaults

### 2. Logging
- Use structured logging (timestamp, level, message)
- Include context (record ID, operation type)
- Don't log sensitive data (passwords, tokens)
- Rotate logs daily

### 3. Error Messages
- Be specific (what failed, why)
- Include context (record, operation)
- Suggest remediation
- Don't expose system details

### 4. Testing
- Unit tests for individual components
- Integration tests for workflows
- Error scenario testing
- Performance/load testing

### 5. Monitoring
- Exit codes for automation detection
- Log level for visibility control
- Performance metrics
- Integrity reports

---

## Summary

These patterns address common challenges:

| Pattern | Problem | Solution |
|---------|---------|----------|
| Connection Pooling | Connection overhead | Reuse persistent connections |
| Atomic Transactions | Partial updates | All-or-nothing updates |
| Exception Hierarchy | Mixed error types | Targeted error handling |
| Resource Management | Resource leaks | Guaranteed cleanup |
| Input Validation | Bad data | Immediate validation |
| Deduplication | Duplicate records | In-memory set lookup |
| Explicit Waits | Race conditions | Wait for conditions |
| Integrity Checking | Inconsistent data | Continuous validation |

These patterns, applied together, create a reliable, robust system.
