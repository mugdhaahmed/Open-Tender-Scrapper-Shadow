# System Architecture

High-level design of the web scraper system.

---

## Table of Contents

1. [Overview](#overview)
2. [System Components](#system-components)
3. [Data Flow](#data-flow)
4. [Design Patterns](#design-patterns)
5. [Scalability](#scalability)

---

## Overview

The system extracts structured data from web portals while operating under external constraints (connection limits, rate limiting). It uses a supervised execution model with adaptive timing to maintain 24/7 availability.

**Key characteristics:**
- Connection-pooled database access
- Multi-entity relational data model
- Automated integrity validation
- Graceful error recovery

---

## System Components

### 1. Execution Supervisor

Manages the overall execution loop:
- Monitors execution outcomes
- Adjusts timing based on results
- Ensures process isolation (no concurrent runs)
- Coordinates system startup/shutdown

**Timing strategy:**
- After success: Standard interval
- After failure: Extended interval  
- After rate-limit: Maximum interval

### 2. Data Extraction Layer

Handles web automation:
- Browser automation (Selenium WebDriver)
- Multi-step navigation through portal
- Element waiting and explicit waits
- Error recovery with retries
- Screenshot capture for failures

### 3. Data Processing Layer

Transforms extracted data:
- Field parsing and validation
- Format normalization
- Deduplication checks
- Missing data detection
- Structure validation

### 4. Database Layer

Manages persistent storage:
- Connection pooling
- Transaction management
- Multi-table inserts
- Data integrity maintenance
- Archive database for history

### 5. Validation System

Ensures data consistency:
- Referential integrity checks
- Orphaned record detection
- Relationship validation
- Automatic repair (dry-run mode)
- Health reporting

---

## Data Flow

```
User Portal (External System)
        ↓ (Extract)
  Extraction Layer
        ↓ (Parse & Validate)
  Processing Layer
        ↓ (Store Atomically)
  Database Layer
        ↓
  Live Database (Active Records)
        ↓ (Archive)
  Archive Database (Historical)
```

### Detailed Flow

1. **Initialization**
   - Load existing record IDs (for deduplication)
   - Verify database connectivity
   - Check integrity of existing data

2. **Extraction**
   - Navigate to portal
   - Iterate through paginated results
   - Extract each record's details
   - Handle multi-step navigation

3. **Validation**
   - Parse extracted fields
   - Validate field values
   - Check for required fields
   - Compare against schema

4. **Storage**
   - Begin atomic transaction
   - Insert into main entity table
   - Insert related entity records
   - Commit (all or nothing)

5. **Post-Processing**
   - Archive expired records
   - Run integrity checks
   - Generate status report
   - Log operation details

---

## Design Patterns

### Pattern 1: Connection Pooling

**Problem:** Creating new database connections is expensive and wasteful.

**Solution:** Maintain small pool of persistent connections, reusing them for all operations.

**Benefits:**
- Reduced connection overhead
- Respects external connection quotas
- Better resource utilization

### Pattern 2: Atomic Transactions

**Problem:** Partial updates could leave related data inconsistent.

**Solution:** Group all related updates into single transaction. Either all succeed or all fail.

**Benefits:**
- No orphaned records
- No inconsistent state
- Data integrity guaranteed

### Pattern 3: Custom Exception Hierarchy

**Problem:** Different errors need different handling (retries, backoff, etc.).

**Solution:** Create specific exception types for different error conditions.

**Benefits:**
- Targeted error handling
- Clear error semantics
- Appropriate response to each failure type

### Pattern 4: Context Managers

**Problem:** Resources might leak if exceptions occur during use.

**Solution:** Use context managers (try-finally) to guarantee cleanup.

**Benefits:**
- Guaranteed resource release
- No connection leaks
- Clean error propagation

### Pattern 5: Adaptive Supervision

**Problem:** Fixed timing can't adapt to changing conditions (rate limits, failures).

**Solution:** Execution supervisor monitors outcomes and adjusts timing accordingly.

**Benefits:**
- Automatic backoff on failure
- Respects rate limits
- Self-healing recovery

---

## Database Design

### Data Model

The system maintains multiple related entities in normalized schema:

**Live Database:**
- Entity A: Core data records
- Entity B: Supporting details (references Entity A)
- Entity C: Additional attributes (references Entity A)
- Entity D: Configuration (references Entity A)
- Entity E: Contact info (references Entity A)

**Archive Database:**
- Same schema as live database
- Holds historical/expired records
- Keeps live database optimized

### Integrity Constraints

- Foreign key constraints between all related entities
- Referential integrity enforced at database level
- Atomic transactions ensure consistency
- Automatic validation layer catches edge cases

### Design Rationale

**Normalization:** Separate tables for different concerns reduces duplication and improves consistency.

**Relationships:** Foreign keys and constraints prevent orphaned data and enforce referential integrity.

**Atomicity:** All related updates in single transaction prevent partial states.

**Archival:** Historical database keeps live database optimized while preserving data.

---

## Error Handling

### Error Categories

1. **Transient Errors** (Connection, timeout)
   - Automatic retry with backoff
   - Supervised loop handles recovery

2. **Rate Limit Errors** (Connection quota exceeded)
   - Recognized explicitly
   - Triggers maximum backoff
   - System waits for quota window to clear

3. **Data Errors** (Invalid format, missing field)
   - Logged with context
   - Record marked for manual review
   - Doesn't stop overall execution

4. **Integrity Errors** (Orphaned data, missing relationships)
   - Detected automatically
   - Can be repaired in dry-run mode
   - Prevents silent corruption

### Recovery Strategy

- Graceful degradation (skip bad records, continue)
- Automatic retry for transient failures
- Backoff timing respects system limits
- Supervisor loop provides recovery mechanism

---

## Performance Considerations

### Connection Efficiency

- Pooling reduces connection count by 70-80%
- Reuse amortizes connection setup cost
- Batch operations reduce roundtrips

### Throughput

- Parallel field extraction where possible
- Batch validation before storage
- Efficient pagination handling

### Data Integrity

- Sub-second validation for consistency checks
- Efficient orphan detection algorithms
- Optimized relationship queries

### Scalability

- Connection pooling scales better than 1:1 connections
- Archive database prevents live DB bloat
- Normalized design minimizes redundancy

---

## Security & Best Practices

- Credentials in environment variables only
- Database users have minimal required privileges
- No sensitive data in logs
- Secure error messages (no data leakage)
- Regular dependency updates

---

## Monitoring & Observability

### Structured Logging

- Operation start/end with timestamps
- Record counts and statistics
- Error details with context
- Integrity check results

### Status Indicators

- Exit codes for automated detection
- Process lock file for concurrent prevention
- Log timestamps for timeline analysis

### Health Checks

- Database connectivity verification
- Integrity check reports
- Archive operation confirmation

---

## Conclusion

The architecture balances reliability, performance, and constraint-awareness through:

1. **Pooled connections** to respect quotas
2. **Atomic transactions** to maintain consistency
3. **Adaptive supervision** for self-healing
4. **Comprehensive validation** to catch errors
5. **Graceful degradation** for robustness

This approach enables reliable, high-volume data extraction under external constraints.
