# Technical Specification

Complete technical reference for system configuration and deployment.

---

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Configuration](#configuration)
3. [Database Schema](#database-schema)
4. [API & Interfaces](#api--interfaces)
5. [Error Codes](#error-codes)

---

## System Requirements

### Software Requirements

- Python 3.11+
- MySQL 8.0+ (InnoDB engine)
- Docker (for containerized deployment)
- Chrome/Chromium browser
- Git

### Hardware Requirements

- **CPU:** 2+ cores (for browser automation)
- **Memory:** 2GB minimum (4GB recommended)
- **Disk Space:** 10GB+ for logs and data
- **Network:** Stable connection to target portal

### External Requirements

- MySQL database server (local or remote)
- Access to data extraction portal
- Portal credentials
- Outbound internet access

---

## Configuration

### Environment Variables

All configuration via `.env` file (12-factor principles):

```
# Database Configuration

# Archive Database

# Portal Credentials
PORTAL_USERNAME=<your_username>
PORTAL_PASSWORD=<your_password>

# System Settings
HEADLESS=true
LOG_LEVEL=INFO
TIMEZONE=UTC
```

### Configuration Principles

- Never hardcode secrets
- Use environment variables for all credentials
- Load configuration on startup
- Validate configuration before execution
- Document all available options

---

## Database Schema

### Overview

Normalized relational schema with N tables. Each table represents a logical entity with foreign key relationships maintaining referential integrity.

### Design Principles

**Normalization:** Data is organized to minimize duplication and improve consistency.

**Relationships:** Foreign keys enforce relationships between entities and prevent orphaned records.

**Constraints:** Check constraints and data types validate data quality at database level.

**Transactions:** All related updates happen atomically or not at all.

### Entity Structure

The schema contains multiple interrelated entities:

- **Primary Entity:** Core records (e.g., document headers)
- **Related Entity 1:** Supporting information (references primary)
- **Related Entity 2:** Additional details (references primary)
- **Related Entity 3:** Configuration data (references primary)
- **Related Entity 4:** Contact information (references primary)

Each entity includes:
- Unique identifier (primary key)
- Timestamp fields (created_at, updated_at)
- Status fields (active, processed, etc.)
- Foreign key references to related entities

### Character Set

- **Collation:** UTF8MB4 (supports multi-language text)
- **Encoding:** UTF-8 (Unicode compatibility)

### Indexes

Created on:
- Primary keys (implicit)
- Foreign keys (performance)
- Status fields (common queries)
- Timestamp fields (time-range queries)

### Archive Schema

Archive database maintains identical schema to live database:
- Stores historical/expired records
- Keeps live database optimized
- Enables time-based queries
- Provides data retention

### Data Integrity

**Referential Integrity:** Foreign key constraints prevent orphaned records.

**Atomic Transactions:** Multi-table inserts succeed completely or fail completely.

**Validation Layer:** Application-level checks catch edge cases.

**Integrity Checker:** Automatic detection and repair of inconsistencies.

---

## API & Interfaces

### Main Entry Point

**File:** `main.py`

**Purpose:** Primary execution entry point

**Behavior:**
1. Load environment configuration
2. Initialize browser automation
3. Connect to databases
4. Execute data extraction cycle
5. Run post-processing (archival, validation)
6. Exit with appropriate status code

**Exit Codes:**
- `0` - Successful execution
- `1` - General error
- `2` - Configuration error
- `88` - Rate limit hit (special handling)
- Other - Specific error conditions

### Supervisor Script

**File:** `run_loop.sh`

**Purpose:** Manage continuous execution

**Behavior:**
- Runs main.py in loop
- Monitors exit codes
- Adjusts sleep duration based on result
- Handles process locking
- Manages logging

**Timing Strategy:**
```
Exit 0:  sleep standard_interval
Exit 88: sleep maximum_interval
Exit 1+: sleep extended_interval
```

### Database Interface

**Connection Management:**
- Connection pooling with configurable pool size
- Automatic connection cleanup
- Query timeout handling
- Connection validation

**Transaction Interface:**
- Begin transaction
- Commit (success) / Rollback (failure)
- Nested transaction support where available
- Deadlock detection and retry

### Browser Automation Interface

**Initialization:**
- Launch headless browser
- Load initial page
- Verify page loaded successfully

**Navigation:**
- Click elements
- Fill forms
- Wait for conditions
- Handle popups/alerts

**Data Extraction:**
- Find elements by locator
- Extract text/attributes
- Handle dynamic content
- Screenshot on failure

### Data Validation Interface

**Input Validation:**
- Required field checking
- Data type validation
- Format validation
- Range validation

**Integrity Checking:**
- Referential integrity checks
- Orphan record detection
- Consistency validation
- Relationship verification

**Repair Operations:**
- Dry-run detection
- Suggested fixes
- Rollback capability
- Audit trail

---

## Error Codes

### Standard Exit Codes

| Code | Category | Meaning | Action |
|------|----------|---------|--------|
| 0 | Success | Normal completion | Continue normally |
| 1 | Error | General failure | Investigate logs |
| 2 | Config | Configuration problem | Fix configuration |
| 10 | DB | Database connection failed | Check connectivity |
| 20 | Auth | Authentication failed | Verify credentials |
| 30 | Portal | Portal access error | Check portal status |
| 40 | Validation | Data validation failed | Review data |
| 50 | Processing | Processing error | Check system state |
| 88 | RateLimit | Connection limit hit | Wait for recovery |

### Error Handling Strategy

**Transient Errors (connection, timeout):**
- Automatic retry with exponential backoff
- Logged with context
- Supervisor loop handles recovery

**Permanent Errors (authentication, invalid config):**
- Logged with details
- Exit immediately
- Manual intervention required

**Rate Limit Errors:**
- Recognized and flagged
- Supervisor adjusts wait time
- Automatic recovery when quota resets

**Data Errors:**
- Logged but don't stop execution
- Record marked for review
- Allows partial success

---

## Logging

### Log Format

Structured logging with timestamp, level, and message:

```
[TIMESTAMP] [LEVEL] [COMPONENT] Message
```

### Log Levels

- **DEBUG:** Detailed execution trace
- **INFO:** Normal operation milestones
- **WARN:** Recoverable issues
- **ERROR:** Failures requiring attention

### Log Rotation

- Daily rotation (prevents huge files)
- Timestamped filenames
- Retention policy (configurable)
- Compression of old logs

---

## Performance Tuning

### Connection Pool

Adjust `DB_POOL_SIZE` based on:
- Connection limits (external quota)
- Concurrent operations
- Memory constraints
- Throughput requirements

### Batch Size

Tune batch operations for:
- Memory usage
- Transaction duration
- Error recovery capability
- Throughput optimization

### Timeout Values

Configure based on:
- Network latency
- Portal responsiveness
- System load
- Acceptable retry count

---

## Troubleshooting

### Common Issues

**Issue:** Frequent timeouts
- **Check:** Network connectivity, portal status, timeout configuration

**Issue:** Memory usage growing
- **Check:** Connection leaks, browser process cleanup, batch size

**Issue:** Database errors
- **Check:** Credentials, connectivity, pool size, query syntax

**Issue:** No data extracted
- **Check:** Portal login, page selectors, element availability

**Issue:** Orphaned records
- **Check:** Transaction handling, error recovery, integrity constraints

---

## Conclusion

This specification documents:
- System requirements and configuration
- Database schema design
- API interfaces
- Error handling
- Performance tuning

See ARCHITECTURE.md for design rationale and DEPLOYMENT.md for operations guide.
