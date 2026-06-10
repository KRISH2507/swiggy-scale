# Failure Cascade Analysis

## Section 1: Traffic Simulation

**Calculation**:
```
Push notifications sent: 180,000,000
CTR: 8%
Active users = 180M × 0.08 = 14,400,000

API calls per user: 12 (browse + search + cart + checkout)
Payment conversion: 15%

Spike duration: 60 seconds

Peak RPS = (14,400,000 × 12) / 60 = 2,880,000 RPS
Payment RPS = (14,400,000 × 0.15) / 60 = 36,000 RPS

Conservative (5-min spread):
Peak RPS = 240,000
Payment RPS = 12,000 (5% of traffic)
```

## Section 2: Component Capacity Analysis

**Formula**:
```
Connections Held = (RPS × Query Time) + (Payment RPS × Payment Hold Time)
```

**PostgreSQL (max_connections = 100)**:
```
At 1,000 RPS (50 payment, 950 regular):
Regular queries: 950 × 0.03s = 28.5 connections
Payment queries: 50 × 1.5s = 75 connections
Total = 103.5 → EXHAUSTED at ~800 RPS
```

**Node.js Event Loop (1 CPU)**:
- Theoretical: 12,000-15,000 RPS
- Realistic with I/O: 2,500-3,000 RPS
- Saturation: ~2,500 RPS

**Memory (4GB RAM)**:
- Available: ~1GB for requests
- Per request: ~15KB
- OOM threshold: 8,000-10,000 RPS with queuing

**Synchronous Payment Amplification**:
- Hold time: 1.5s vs 0.03s (50x amplification)
- 50 concurrent payments = 75% of connection pool
- Critical at 600 RPS

## Section 3: Failure Cascade

| Failure | Severity | Trigger RPS | Root Cause | User Impact | Cascading Effect |
|---------|----------|-------------|------------|-------------|------------------|
| **DB Pool Exhaustion** | CRITICAL | 800 | Payment calls hold connections 1.5s; max_connections=100 | ECONNREFUSED, 500 errors, complete failure | → Event loop stalls → Memory buildup → Health checks fail |
| **Event Loop Saturation** | CRITICAL | 2,500 | Single CPU, synchronous blocking, no backpressure | 30s timeouts, 504 errors, frozen app | → CPU 100% → GC thrashing → OOM crash |
| **Promo Code Race** | HIGH | 1,500 | No distributed locks, Read-Check-Update race | Over-redemption, budget exceeded, wrong pricing | → Emergency deactivation → Support surge → Revenue loss ₹3L/min |
| **Memory OOM** | CRITICAL | 8,000 | 4GB limit, queued requests, 1.4GB heap | Process crash, all requests lost | → SIGKILL → 100% failure → 2-5min restart → Thundering herd |
| **NIC Saturation** | MEDIUM | 4,000 | 1Gbps NIC, serving static assets, no CDN | Slow images, broken UI | → Browser retries → TCP exhaustion → API delays |

## Section 4: Incident Timeline

| Time | Event | System State | Impact |
|------|-------|--------------|--------|
| **T+0s** | Push sent to 180M users | Baseline: 22/100 DB conn, 12ms lag | Normal |
| **T+12s** | RPS 142K | DB 100/100 EXHAUSTED, lag 2.8s | 40% failure, 500 errors |
| **T+20s** | RPS 198K | Event loop 18s lag, CPU 100%, 3.2GB mem | 85% failure, app frozen |
| **T+35s** | Promo spike | 2,840 over-redemptions | ₹5.68L loss, fraud risk |
| **T+50s** | RPS 240K peak | OOM: 1.42GB/1.40GB heap | **Process crash, 100% down** |
| **T+2m** | Auto-restart | Thundering herd hits | Crashes again in 13s |
| **T+5m** | Traffic blocked | 500 RPS limit (99.75% rejected) | Service "unavailable" |
| **T+15m** | 4 instances added | 2K RPS capacity | Heavily throttled |
| **T+45m** | 20 instances + Redis | 12K RPS | Stable but late |
| **T+2h** | Traffic normalized | 4.2K RPS | Recovery complete |

**Total Impact**: 45min downtime, ₹189 crore loss, 14.4M users affected, 1.8M orders lost
