# AWS Cost Estimate

## Baseline Monthly Cost (Normal Traffic: 2-4K RPS)

| Service | Instance Type | Quantity | Unit Cost | Hours/Month | Monthly Cost |
|---------|--------------|----------|-----------|-------------|--------------|
| **EC2 Node.js** | t3.medium | 20 | $0.0416/hr | 730 | $607.36 |
| **RDS Primary** | db.r6g.2xlarge | 1 | $0.96/hr | 730 | $700.80 |
| **RDS Replicas** | db.r6g.xlarge | 3 | $0.48/hr | 730 | $1,051.20 |
| **ElastiCache Redis** | cache.r6g.large | 1 | $0.252/hr | 730 | $183.96 |
| **PgBouncer** | t3.small | 2 | $0.0208/hr | 730 | $30.37 |
| **ALB** | - | 1 | $0.0225/hr + LCU | 730 | $16.43 + $50 |
| **CloudFront** | - | - | $0.085/GB | 2TB | $170.00 |
| **SQS** | - | - | $0.40/M req | 50M | $20.00 |
| **Payment Workers** | t3.small | 5 | $0.0208/hr | 730 | $75.92 |
| **Data Transfer** | - | - | $0.09/GB | 1TB | $90.00 |
| **EBS Volumes** | gp3 | 1.5TB | $0.08/GB-mo | - | $120.00 |
| | | | | **Total** | **$3,116.04** |

**Annual Baseline**: ₹3,116 × 12 × 83 = **₹31.0 lakh/year**

---

## Peak Event Cost (4-Hour Surge: 240K RPS)

**Additional Resources**:

| Service | Baseline | Peak | Additional | Unit Cost | Duration | Event Cost |
|---------|----------|------|------------|-----------|----------|-----------|
| **EC2 Node.js** | 20 | 100 | +80 | $0.0416/hr | 4hr | $13.31 |
| **Payment Workers** | 5 | 20 | +15 | $0.0208/hr | 4hr | $1.25 |
| **CloudFront** | 2TB/mo | +500GB | +500GB | $0.085/GB | - | $42.50 |
| **SQS** | 50M/mo | +20M | +20M | $0.40/M | - | $8.00 |
| **Data Transfer** | 1TB/mo | +200GB | +200GB | $0.09/GB | - | $18.00 |
| | | | | | **Total** | **$83.06** |

**Peak Event Cost**: $83 × 83 = **₹6,894 per event**

**Annual Cost** (4 events/year): ₹6,894 × 4 = **₹27,576**

---

## Total Annual Cost

```
Baseline Annual:        ₹31,00,000
Peak Events (4×):       ₹27,576
Reserve Instance (30%): -₹9,30,000
-------------------------------------------
Net Annual Cost:        ₹21,97,576
Monthly Average:        ₹1,83,131
```

---

## Business Justification

### Outage Cost Analysis

**Current Monolith Failure**:
```
Revenue loss rate: ₹4.2 crore/minute
Expected downtime: 45 minutes
Total loss = 45 × ₹4.2 crore = ₹189 crore per incident
```

**With 2 major events/year**:
```
Annual outage cost = 2 × ₹189 crore = ₹378 crore
```

### ROI Calculation

| Scenario | Annual Cost | Outage Risk | Expected Loss | Net Impact |
|----------|-------------|-------------|---------------|------------|
| **Monolith** | ₹0 infra | 100% (2 events) | ₹378 crore | -₹378 crore |
| **Redesign** | ₹22 lakh | 0% (resilient) | ₹0 | -₹22 lakh |
| **Savings** | | | | **₹377.78 crore** |

**ROI**: (₹377.78 crore / ₹22 lakh) × 100 = **171,718% return**

**Payback Period**: 45 minutes of prevented downtime = ₹189 crore saved

```
Infrastructure cost / Outage prevention = ₹22L / ₹189 crore = 0.01%
```

**Justification**: Spending ₹22 lakh to prevent ₹378 crore in annual losses is a **1,718:1 return on investment**. The architecture pays for itself in the first prevented outage.

### Additional Benefits

- **Reputation**: Prevent viral "Swiggy is down" social media damage
- **Customer retention**: 14.4M users don't switch to competitors
- **Brand trust**: Ability to handle major sporting events
- **Revenue**: Enable aggressive marketing campaigns without fear
- **Operational**: Reduced support tickets, on-call stress, manual interventions
