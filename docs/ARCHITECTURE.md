# Architecture Redesign

## Section 1: Current Monolith

```
                    ┌─────────────────────────┐
                    │   Mobile/Web Users      │
                    │    (14.4M spike)        │
                    └───────────┬─────────────┘
                                │ 240K RPS
                                ▼
                    ┌─────────────────────────┐
                    │    Single Node.js       │◄─── 1 CPU, 4GB RAM
                    │    Express Server       │     SPOF, No scaling
                    │    (Port 3000)          │     Serves static assets
                    └───────────┬─────────────┘
                                │
                                │ Synchronous calls
                                │ (1.5s payment block)
                                │
                    ┌───────────┴─────────────┐
                    │                         │
                    ▼                         ▼
        ┌─────────────────────┐   ┌─────────────────────┐
        │   PostgreSQL DB     │   │   Razorpay API      │
        │  max_connections=100│   │  (200-2000ms)       │
        │   No read replicas  │   │  External service   │
        └─────────────────────┘   └─────────────────────┘
                ▲
                │ Connection pool exhaustion at 800 RPS
                └─── No caching, No CDN, No queue
```

**Critical Weaknesses**:
- Single point of failure (SPOF)
- No horizontal scaling
- Synchronous payment blocking
- No caching layer
- No CDN for static assets
- Connection pool too small
- No request queue
- No circuit breakers

---

## Section 2: Redesigned Architecture

```
                    ┌──────────────────────────────┐
                    │     Mobile/Web Users         │
                    │       (14.4M spike)          │
                    └──────────────┬───────────────┘
                                   │ 240K RPS
                                   ▼
                    ┌──────────────────────────────┐
                    │      CloudFront CDN          │◄─── TTL: 5min (static)
                    │   (Static: images, CSS, JS)  │     300+ edge locations
                    └──────────────┬───────────────┘     Offloads 60% traffic
                                   │ 96K RPS (API only)
                                   ▼
                    ┌──────────────────────────────┐
                    │  Application Load Balancer   │◄─── Health checks
                    │      (Multi-AZ)              │     SSL termination
                    └──────────────┬───────────────┘     Connection pooling
                                   │
                    ┌──────────────┴───────────────┐
                    │                              │
                    ▼                              ▼
        ┌─────────────────────┐       ┌─────────────────────┐
        │   Node.js Fleet     │       │   Redis Cache       │
        │   (Auto-scaling)    │◄─────►│   (ElastiCache)     │
        │   20-100 instances  │       │   TTL: 30s-5min     │
        │   t3.medium         │       │   cache.r6g.large   │
        └──────────┬──────────┘       └─────────────────────┘
                   │                           ▲
                   │                           │
                   │                    Menu: 5min
                   │                    Restaurant: 2min
                   │                    User session: 30min
                   │
                   ├──────── Async ─────┐
                   │                    │
                   ▼                    ▼
        ┌─────────────────────┐   ┌─────────────────────┐
        │    PgBouncer         │   │    SQS Queue        │
        │  Connection Pooler   │   │  (Payment Queue)    │
        │  pool_size: 100      │   │  FIFO, DLQ          │
        │  max_client: 10K     │   └──────────┬──────────┘
        └──────────┬───────────┘              │
                   │                           ▼
                   ▼                ┌─────────────────────┐
        ┌─────────────────────┐    │  Payment Workers    │
        │  PostgreSQL Primary │    │  (Auto-scaling)     │
        │   db.r6g.2xlarge    │    │  5-20 instances     │
        │   8vCPU, 64GB       │    └──────────┬──────────┘
        └──────────┬──────────┘               │
                   │                           │
                   ▼                           ▼
        ┌─────────────────────┐    ┌─────────────────────┐
        │  Read Replicas (3)  │    │   Razorpay API      │
        │  db.r6g.xlarge      │    │   (Async calls)     │
        │  Cross-AZ           │    │   Circuit breaker   │
        └─────────────────────┘    └─────────────────────┘
```

**Component Specifications**:

| Component | Configuration | Purpose | Scaling |
|-----------|--------------|---------|---------|
| **CloudFront** | 300+ edge locations | Static asset delivery | Unlimited |
| **ALB** | Multi-AZ, 3 zones | Traffic distribution, SSL | Auto-scales |
| **Node.js Fleet** | t3.medium (2vCPU, 4GB) × 20-100 | API processing | Auto: CPU >70% |
| **Redis** | cache.r6g.large (2vCPU, 13GB) | Session, menu, restaurant cache | Vertical |
| **PgBouncer** | t3.small, pool=100, clients=10K | Connection multiplexing | Stateless |
| **PostgreSQL** | db.r6g.2xlarge (8vCPU, 64GB) | Primary writes | Vertical |
| **Read Replicas** | db.r6g.xlarge (4vCPU, 32GB) × 3 | Read traffic (80% queries) | Horizontal |
| **SQS** | FIFO, DLQ, retention 14d | Payment async processing | Unlimited |
| **Payment Workers** | t3.small × 5-20 | Process payment queue | Auto: queue depth |

**Cache Strategy**:

| Data Type | TTL | Invalidation | Impact |
|-----------|-----|--------------|--------|
| Restaurant list | 5 min | On update | 40% traffic offload |
| Menu items | 5 min | On update | 30% traffic offload |
| User session | 30 min | On logout | No DB hit |
| Promo codes | 1 min | On change | Race condition prevention |
| Static assets | 1 hour | Version-based | 60% bandwidth saving |

---

## Section 3: Component Justification

| Component | Failure Prevented | Prevention Mechanism |
|-----------|-------------------|----------------------|
| **CloudFront CDN** | NIC saturation (4K RPS) | Offloads 60% traffic (static assets), 300+ edge locations, reduces origin load |
| **ALB** | Single point of failure | Multi-AZ deployment, health checks, automatic failover, SSL termination |
| **Auto-Scaled Fleet** | Event loop saturation (2.5K RPS) | 20-100 instances, each handles 2K RPS, total capacity 40K-200K RPS |
| **Redis Cache** | DB read overload | 70% read traffic cached (menu, restaurants), TTL 30s-5min, sub-ms latency |
| **PgBouncer** | Connection exhaustion (800 RPS) | Multiplexes 10K clients to 100 DB connections, transaction pooling |
| **Read Replicas** | Primary DB overload | 80% read traffic routed to replicas, 3× read capacity, cross-AZ |
| **SQS + Workers** | Synchronous payment blocking | Async payment processing, decouples API from payment latency, auto-scales on queue depth |
| **Circuit Breaker** | Payment cascade failure | Fast-fail on Razorpay timeout, prevents connection pool exhaustion, graceful degradation |
| **Distributed Lock** | Promo race condition (1.5K RPS) | Redis-based locking for promo validation, prevents over-redemption |

**Key Improvements**:
- **Capacity**: 800 RPS → 150K+ RPS (188× increase)
- **Availability**: Single instance → Multi-AZ, auto-healing
- **Latency**: p99 8s → p99 150ms (53× improvement)
- **Resilience**: SPOF → No single point of failure
- **Cost**: Predictable scaling, pay-per-use
