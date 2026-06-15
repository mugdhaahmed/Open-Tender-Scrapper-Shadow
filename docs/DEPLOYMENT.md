# Deployment Guide

Deployment of the system on Hostinger VPS with Docker containerization.

---

## Overview

The system is deployed as a containerized application on Hostinger VPS:

- **Container Runtime:** Docker
- **Host:** Hostinger Virtual Private Server (VPS)
- **Database:** MySQL on Hostinger
- **Execution:** Supervised background container with auto-restart

---

## Architecture

```
Hostinger VPS
├── Docker Engine
│   └── App Container
│       ├── Python 3.11+
│       ├── Selenium WebDriver
│       ├── Data volumes (/data)
│       └── .env configuration
│
└── Hostinger MySQL
    ├── Live Database
    └── Archive Database
```

---

## Prerequisites

### VPS Requirements
- [ ] Hostinger VPS account
- [ ] SSH access to VPS
- [ ] Docker installed on VPS
- [ ] Sufficient disk space (10GB+)

### Database Setup
- [ ] MySQL database created on Hostinger
- [ ] Database user with appropriate privileges
- [ ] Database connection tested from VPS
- [ ] Network firewall rules configured (if needed)

---

## Production Checklist

Before going live on Hostinger:

- [ ] Hostinger VPS created and SSH access configured
- [ ] Docker installed on VPS
- [ ] MySQL database created on Hostinger
- [ ] Database credentials verified
- [ ] .env file configured with Hostinger credentials
- [ ] Docker image built and tested
- [ ] Container runs successfully once (test run)
- [ ] Container deployed as background service
- [ ] Logs verified for normal operation
- [ ] Database integrity verified
- [ ] Auto-restart policy configured

---

## System Configuration Summary

| Component | Location | Status |
|-----------|----------|--------|
| **VPS** | Hostinger | ✅ Deployed |
| **Docker** | Hostinger VPS | ✅ Running |
| **Container** | Hostinger VPS | ✅ app-scraper (daemon) |
| **Database** | Hostinger MySQL | ✅ app_db + app_archive |
| **Data** | /home/app/scraper/data | ✅ Logs, screenshots, backups |
| **Config** | /home/app/scraper/.env | ✅ Hostinger credentials |
| **Execution** | 24/7 continuous | ✅ Auto-restart unless-stopped |

---

## Conclusion

The system is deployed on Hostinger VPS:

1. **Containerized** - Docker image simplifies deployment and scaling
2. **Database on Hostinger** - MySQL configured with live and archive databases
3. **Automated** - Container runs 24/7 with automatic restart
4. **Monitored** - Logs available for troubleshooting
5. **Scalable** - Can adjust container resources as needed

This setup provides reliable, production-grade operation on Hostinger infrastructure.
