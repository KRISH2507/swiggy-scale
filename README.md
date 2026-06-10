# SwiftEats Production Scale Architecture Review

Production-scale architecture analysis for handling 14.4M concurrent users during India vs Pakistan World Cup Final with 50% discount campaign.

## Scenario Summary

**Event**: India vs Pakistan World Cup Final
- Push notification: 180M users
- CTR: 8% (14.4M active users)
- Traffic spike: 60-second window
- Peak RPS: 240,000 (conservative estimate)
- Discount: 50% off

**Current Architecture**: Node.js monolith, single server, 1 CPU, 4GB RAM, PostgreSQL max_connections=100

**Outcome**: Catastrophic failure at 800 RPS (0.3% of expected load), 45-minute outage, ₹189 crore revenue loss

---

## Document Summary

| Document | Purpose | Key Findings |
|----------|---------|--------------|
| [FAILURE-CASCADE.md](docs/FAILURE-CASCADE.md) | Traffic simulation and failure analysis | System fails at 800 RPS due to DB pool exhaustion; payment calls amplify resource usage 50×; 5 cascading failures identified |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Current vs redesigned architecture | Redesigned system handles 150K+ RPS with CloudFront, ALB, auto-scaling, Redis, PgBouncer, read replicas, async payments |
| [COST-ESTIMATE.md](docs/COST-ESTIMATE.md) | AWS infrastructure costs | ₹22 lakh/year prevents ₹378 crore annual losses; 1,718:1 ROI; infrastructure pays for itself in first prevented outage |
| [RUNBOOK.md](docs/RUNBOOK.md) | Incident response procedures | 5-step incident response: Detect (8 CloudWatch alarms), Triage (decision tree), Respond (4 action plans), Rollback, Post-mortem |

---

## Key Findings

### Critical Bottlenecks

1. **DB Pool Exhaustion** @ 800 RPS
   - Root cause: Synchronous payments hold connections 1.5s (50× amplification)
   - Impact: Complete service failure
   - Solution: PgBouncer + async payment workers

2. **Event Loop Saturation** @ 2,500 RPS
   - Root cause: Single CPU, synchronous operations
   - Impact: 30s timeouts, frozen app
   - Solution: Auto-scaled fleet (20-100 instances)

3. **Promo Race Condition** @ 1,500 RPS
   - Root cause: No distributed locking
   - Impact: ₹3L/min revenue leakage
   - Solution: Redis-based distributed locks

4. **Memory OOM** @ 8,000 RPS
   - Root cause: 4GB limit, request queue buildup
   - Impact: Process crash, restart loop
   - Solution: Horizontal scaling + request shedding

5. **NIC Saturation** @ 4,000 RPS
   - Root cause: Serving static assets, no CDN
   - Impact: Slow images, broken UI
   - Solution: CloudFront offloads 60% traffic

### Incident Timeline

| Time | Event | Impact |
|------|-------|--------|
| T+12s | DB pool exhausted @ 142K RPS | 40% failure rate |
| T+20s | Event loop collapsed @ 198K RPS | 85% failure rate |
| T+50s | OOM crash @ 240K RPS | 100% downtime |
| T+2m | Restart fails (thundering herd) | Crash loop |
| T+5m | Emergency traffic block | 99.75% rejected |
| T+45m | Full recovery (manual scaling) | Service restored |

**Total Impact**: 45 min downtime, ₹189 crore loss, 14.4M users affected, 1.8M orders lost

---

## Architecture Overview

### Current (Monolith)
```
Users → Node.js (1 CPU) → PostgreSQL (max_conn=100) → Razorpay (sync)
         ↓ Fails @ 800 RPS
```

### Redesigned (Multi-Tier)
```
Users → CloudFront CDN (static) → ALB → Node.js Fleet (20-100) → Redis Cache
                                   ↓                              ↓
                            PgBouncer (pool multiplexer)    Read Replicas (3)
                                   ↓                              ↓
                            PostgreSQL Primary              SQS → Workers → Razorpay (async)
```

**Capacity Improvement**: 800 RPS → 150K+ RPS (188× increase)

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **CDN** | AWS CloudFront | Static asset delivery, 60% traffic offload |
| **Load Balancer** | Application Load Balancer | Multi-AZ, health checks, SSL termination |
| **Compute** | EC2 t3.medium (auto-scaled) | API processing, 20-100 instances |
| **Cache** | ElastiCache Redis | Menu, restaurants, sessions (70% reads cached) |
| **Connection Pool** | PgBouncer | 10K clients → 100 DB connections |
| **Database** | RDS PostgreSQL + 3 read replicas | Writes to primary, 80% reads to replicas |
| **Queue** | SQS FIFO | Async payment processing |
| **Workers** | EC2 t3.small (auto-scaled) | Payment processing, 5-20 instances |
| **Monitoring** | CloudWatch | 8 alarms, triage decision tree |

---

## Business Case

**Investment**: ₹22 lakh/year infrastructure cost

**Returns**:
- Prevents ₹378 crore annual outage losses (2 events/year)
- ROI: 1,718:1 (171,718% return)
- Payback: First prevented outage (45 minutes)
- Additional: Brand protection, customer retention, operational stability

**Risk Mitigation**: Current system **guaranteed to fail** at 0.3% of expected traffic

---

## Quick Start

Review documents in order:
1. [FAILURE-CASCADE.md](docs/FAILURE-CASCADE.md) - Understand what breaks and when
2. [ARCHITECTURE.md](docs/ARCHITECTURE.md) - Learn the redesigned system
3. [COST-ESTIMATE.md](docs/COST-ESTIMATE.md) - Review financial justification
4. [RUNBOOK.md](docs/RUNBOOK.md) - Prepare for incident response

---

## Author

Senior Staff Engineer  
Production Architecture Review  
Date: June 10, 2026
