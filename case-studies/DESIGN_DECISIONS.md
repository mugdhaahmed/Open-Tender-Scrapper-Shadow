# Design Decisions

Key architectural decisions and their rationale.

---

## Decision 1: Connection Pooling vs Individual Connections

### Decision
Use connection pooling with a small fixed-size pool instead of creating connections on-demand.

### Rationale
- **Constraint:** System operates under external connection quota (e.g., 500/hour)
- **Problem:** Naive approach (create connection per operation) wastes quota
- **Solution:** Maintain 2-3 persistent connections, reuse them
- **Tradeoff:** Slightly more complex code → Massive quota savings (80%)

### Alternative Considered
- Create new connection for each operation (simpler code)
- Would use 80% more connections
- Violates external constraints
- Rejected

### Implementation
- Pool initialized on startup
- Connections checked out/returned explicitly
- Automatic cleanup on errors
- Configurable pool size

---

## Decision 2: Relational Schema vs Denormalized

### Decision
Use normalized relational schema with separate tables for different entities.

### Rationale
- **Goal:** Maintain data integrity and prevent inconsistencies
- **Problem:** Denormalized (single table) design is prone to update anomalies
- **Solution:** Multiple tables with foreign keys enforce relationships
- **Benefit:** Atomic transactions guarantee consistency

### Alternative Considered
- Single denormalized table (simpler queries)
- Prone to orphaned records and inconsistencies
- Hard to maintain data quality
- Rejected for production use

### Design
- Primary entity table (core records)
- 4 related entity tables (supporting details)
- Foreign key constraints
- Referential integrity enforced at DB level

---

## Decision 3: Atomic Transactions vs Partial Inserts

### Decision
All related data updates happen in single atomic transaction. Either all succeed or all fail.

### Rationale
- **Goal:** Never allow partial updates
- **Problem:** If some related inserts succeed and others fail, data becomes inconsistent
- **Solution:** Transaction wraps all inserts; commit only if all succeed
- **Benefit:** No orphaned records, guaranteed consistency

### Alternative Considered
- Insert and handle errors individually
- Simpler per-insert logic
- Results in inconsistent data
- Rejected

### Implementation
- BEGIN TRANSACTION at start
- Multiple INSERT operations
- COMMIT if all succeed
- ROLLBACK if any fails

---

## Decision 4: In-Memory Deduplication vs Database Queries

### Decision
Load existing record IDs into memory set for O(1) deduplication checks.

### Rationale
- **Goal:** Skip records already in database
- **Problem:** Querying database for each record during extraction is slow
- **Solution:** Load all IDs on startup, use in-memory set lookup (very fast)
- **Tradeoff:** Uses memory → Eliminates redundant database queries

### Alternative Considered
- Query database for each extracted record
- Accurate (always queries current state)
- Very slow (N+1 queries)
- Rejected for performance

### Implementation
- Load IDs from database once on startup
- Store in Python set (hash table, O(1) lookup)
- Check membership before processing
- Trade: Memory for speed

---

## Decision 5: Supervised Loop vs Cron Job

### Decision
Use custom supervision script (bash loop) instead of cron for execution management.

### Rationale
- **Goal:** Respect connection quotas and handle rate limits
- **Problem:** Cron runs at fixed intervals; doesn't know about rate limits
- **Solution:** Custom loop monitors exit codes and adjusts timing
- **Benefit:** Intelligent backoff when rate limits hit

### Alternative Considered
- Use cron with fixed intervals
- Simple to set up
- Can't adapt to rate limits
- Would violate quotas
- Rejected

### Implementation
- Bash loop runs main.py
- Monitors exit code
- Adjusts sleep duration:
  - Success: Standard interval
  - Rate limit: Extended interval
  - Failure: Recovery interval
- Provides self-healing recovery

---

## Decision 6: Headless Browser vs HTTP Library

### Decision
Use headless browser automation (Selenium) instead of HTTP library (requests).

### Rationale
- **Problem:** Target portal uses JavaScript to render data
- **HTTP Library:** Can't execute JavaScript, missing data
- **Headless Browser:** Fully renders page, can extract all data
- **Tradeoff:** Slower (render time) → More reliable (actual rendered content)

### Alternative Considered
- HTTP library (requests, httpx)
- Much faster
- Can't handle dynamic/JS-rendered content
- Works for static sites only
- Rejected for this use case

### Implementation
- Selenium WebDriver with headless Chromium
- Explicit waits for elements
- JavaScript execution capability
- Screenshot capability for debugging

---

## Decision 7: Separate Archive Database vs Single Database

### Decision
Maintain separate archive database for historical records.

### Rationale
- **Goal:** Keep live database optimized for current operations
- **Problem:** Old records bloat live database, slow down queries
- **Solution:** Move expired records to archive database
- **Benefit:** Live DB stays small and fast

### Alternative Considered
- Single database with all records (active + archived)
- Simpler operational model
- Live DB grows over time
- Queries slow down
- Rejected

### Implementation
- Archive database with identical schema
- Periodic archival job moves old records
- Live database stays optimized
- Archive database available for historical queries

---

## Decision 8: UTF8MB4 vs UTF8 Encoding

### Decision
Use UTF8MB4 (4-byte UTF-8) instead of UTF8 (3-byte).

### Rationale
- **Data:** Contains multi-language text
- **Problem:** UTF8 (MySQL) only supports 3-byte characters
- **Solution:** UTF8MB4 supports full 4-byte Unicode
- **Benefit:** Supports all Unicode characters including emoji, rare languages

### Alternative Considered
- UTF8 (MySQL's limited 3-byte version)
- Works for basic Latin/European languages
- Fails for some CJK, Arabic, etc.
- Data truncation or errors
- Rejected

### Implementation
- Database: DEFAULT CHARACTER SET utf8mb4
- Collation: utf8mb4_general_ci
- Connection: UTF8MB4 charset

---

## Decision 9: FK Constraints vs Application Validation

### Decision
Enforce referential integrity at database level using foreign key constraints.

### Rationale
- **Goal:** Prevent orphaned records
- **Database-Level:** Can't violate constraint, guaranteed consistency
- **Application-Level:** Can be bypassed (accidental queries, bugs)
- **Benefit:** Integrity guaranteed by database, not dependent on code correctness

### Alternative Considered
- Validate relationships in application code
- Flexibility to allow temporary inconsistencies
- Can accidentally bypass validation
- Data can become corrupted
- Rejected for production

### Implementation
- FK constraints declared in schema
- Database rejects invalid operations
- Application respects constraints
- Cascade delete on related records

---

## Decision 10: Archive Database vs Delete & Compression

### Decision
Move old records to archive database instead of deleting or compressing.

### Rationale
- **Data Retention:** May need historical records for audits, reports
- **Delete:** Loses data permanently
- **Compression:** Makes data hard to query
- **Archive DB:** Full queryable copy, enables historical analysis
- **Benefit:** Keep history without compromising live DB performance

### Alternative Considered
- Delete old records
  - Simple
  - Permanent data loss
  - Can't recover for audits
  - Rejected

- Compress in live database
  - Saves space
  - Hard to query compressed data
  - Rejected

### Implementation
- Archive database (identical schema to live)
- Periodic job moves expired records
- Archive available for queries
- Maintains full data history

---

## Summary

These 10 decisions balance competing concerns:

| Decision | Tradeoff | Winner |
|----------|----------|--------|
| Pooling | Complexity vs Quota savings | Pooling |
| Relational | Query complexity vs Data integrity | Relational |
| Atomic | Code simplicity vs Consistency | Atomic |
| In-Memory | Memory vs Query speed | In-Memory |
| Supervision | Code complexity vs Rate limit handling | Supervision |
| Browser | Speed vs Dynamic content | Browser |
| Archive | Operational complexity vs Live DB performance | Archive |
| UTF8MB4 | Compatibility vs Character support | UTF8MB4 |
| FK Constraints | Flexibility vs Guaranteed integrity | FK Constraints |
| Archive DB | Data loss vs History retention | Archive DB |

Each decision prioritizes **data integrity** and **operational reliability** over simplicity or speed.
