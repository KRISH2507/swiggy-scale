# Production Incident Runbook

## STEP 1: DETECT - CloudWatch Alarms

| Metric | Threshold | Severity | Action |
|--------|-----------|----------|--------|
| `HighRPS` | RPS > 50,000 | WARNING | Alert on-call, monitor |
| `DBConnectionPoolUtil` | >80% | CRITICAL | Page SRE, investigate queries |
| `EventLoopLag` | >1,000ms | CRITICAL | Page SRE, check CPU/memory |
| `5xxErrorRate` | >5% | CRITICAL | Page eng + SRE, investigate logs |
| `PaymentQueueDepth` | >10,000 | WARNING | Scale workers, check Razorpay |
| `CacheHitRate` | <60% | WARNING | Check Redis, verify TTLs |
| `MemoryUtilization` | >85% | CRITICAL | Scale immediately, investigate leaks |
| `P99Latency` | >2,000ms | WARNING | Investigate slow queries, check Redis |

**Alert Routing**:
- CRITICAL → PagerDuty → On-call SRE + Engineering Lead
- WARNING → Slack #incidents channel

---

## STEP 2: TRIAGE - Decision Tree

```
                    ┌─────────────────────┐
                    │   Incident Detected │
                    └──────────┬──────────┘
                               │
                ┌──────────────┴──────────────┐
                ▼                             ▼
        ┌───────────────┐             ┌──────────────┐
        │ 5xx > 20%?    │             │ RPS spiking? │
        └───────┬───────┘             └──────┬───────┘
                │                             │
          YES   │  NO                   YES   │  NO
                │                             │
                ▼                             ▼
    ┌────────────────────┐        ┌────────────────────┐
    │ DB pool >90%?      │        │ Check CloudWatch   │
    └────────┬───────────┘        │ for root cause     │
             │                    └────────────────────┘
       YES   │  NO
             │
             ▼
    ┌────────────────────┐
    │ Payment related?   │
    └────────┬───────────┘
             │
       YES   │  NO
             │
             ▼                     ▼
    ┌────────────────┐    ┌──────────────────┐
    │ Scale payment  │    │ Scale Node.js    │
    │ workers NOW    │    │ fleet NOW        │
    └────────────────┘    └──────────────────┘
```

**Quick Checks**:
1. Check RPS: Is traffic abnormal?
2. Check DB connections: Pool exhausted?
3. Check error rate: What's failing?
4. Check payment queue: Backed up?
5. Check Redis: Cache miss spike?

---

## STEP 3: RESPOND - Incident Actions

### Action A: DB Pool Exhaustion

**Symptoms**: `ECONNREFUSED`, connection timeouts, 500 errors

**Immediate**:
```bash
# Check connection count
aws rds describe-db-instances --db-instance-identifier swifteats-prod \
  --query 'DBInstances[0].DBParameterGroups'

# Scale Node.js fleet (releases some connections)
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name swifteats-app-asg \
  --desired-capacity 40

# Monitor connection pool
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=swifteats-prod \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Maximum
```

**Expected Recovery**: 2-5 minutes after scaling

**Team**: Database SRE + Backend Engineering

---

### Action B: Compute Saturation

**Symptoms**: High CPU, event loop lag >5s, slow responses

**Immediate**:
```bash
# Scale Node.js fleet aggressively
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name swifteats-app-asg \
  --desired-capacity 60

# Check instance health
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names swifteats-app-asg \
  --query 'AutoScalingGroups[0].Instances[*].[InstanceId,HealthStatus,LifecycleState]'

# Verify ALB targets
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:region:account:targetgroup/swifteats-app/xxx
```

**Expected Recovery**: 3-7 minutes (instance boot + warmup)

**Team**: SRE + Platform Engineering

---

### Action C: Payment Queue Backlog

**Symptoms**: Payment queue depth >10K, slow payment confirmations

**Immediate**:
```bash
# Check queue depth
aws sqs get-queue-attributes \
  --queue-url https://sqs.region.amazonaws.com/account/swifteats-payments \
  --attribute-names ApproximateNumberOfMessages

# Scale payment workers
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name swifteats-payment-workers \
  --desired-capacity 20

# Check DLQ for failures
aws sqs get-queue-attributes \
  --queue-url https://sqs.region.amazonaws.com/account/swifteats-payments-dlq \
  --attribute-names ApproximateNumberOfMessages
```

**Expected Recovery**: Queue drains at ~500 payments/sec with 20 workers

**Team**: Payments Engineering + SRE

---

### Action D: Cache Miss Spike

**Symptoms**: Redis hit rate <60%, increased DB load

**Immediate**:
```bash
# Check Redis status
aws elasticache describe-cache-clusters \
  --cache-cluster-id swifteats-redis \
  --show-cache-node-info

# Check cache hit rate
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CacheHitRate \
  --dimensions Name=CacheClusterId,Value=swifteats-redis \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# Warm cache (if needed)
curl -X POST https://api.swifteats.com/admin/cache/warm \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

**Expected Recovery**: Cache warms in 5-10 minutes

**Team**: Backend Engineering + SRE

---

## STEP 4: ROLLBACK

### Rollback Criteria

**When to Rollback**:
- Error rate >30% for 5+ minutes
- Deployment within last 30 minutes
- New code identified as cause
- No improvement from scaling

**When NOT to Rollback**:
- Traffic spike (scale instead)
- External dependency failure (circuit break)
- Database issue (fix DB, not app)

### Rollback Commands

```bash
# Check recent deployments
aws deploy list-deployments \
  --application-name swifteats-app \
  --create-time-range begin=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S),end=$(date -u +%Y-%m-%dT%H:%M:%S)

# Rollback to previous version
aws deploy create-deployment \
  --application-name swifteats-app \
  --deployment-group-name production \
  --s3-location bucket=swifteats-deploys,key=app-v1.2.3.zip,bundleType=zip \
  --description "Emergency rollback from v1.2.4"

# Monitor rollback progress
aws deploy get-deployment --deployment-id d-XXXXX
```

**Expected Recovery**: 8-12 minutes for full fleet rollback

**CRITICAL**: **Never rollback database schema changes**. Migrations must be backward-compatible.

---

## STEP 5: POST-MORTEM

### Post-Mortem Template

```markdown
# Incident Post-Mortem: [Title]

**Date**: YYYY-MM-DD
**Duration**: XX minutes
**Severity**: P0 / P1 / P2
**Impact**: XX users, ₹XX revenue loss

---

## Timeline

| Time | Event | Action Taken |
|------|-------|--------------|
| HH:MM | First alert | ... |
| HH:MM | Escalation | ... |
| HH:MM | Mitigation | ... |
| HH:MM | Resolution | ... |

---

## Root Cause

[Technical root cause analysis]

---

## Impact

- **Users Affected**: X million
- **Error Rate**: X%
- **Revenue Loss**: ₹X crore
- **Downtime**: X minutes

---

## What Went Well

- Detection within X minutes
- Response time X minutes
- Communication clear

---

## What Went Wrong

- Late escalation
- Missing monitoring
- Manual intervention required

---

## Action Items

| Action | Owner | Deadline | Status |
|--------|-------|----------|--------|
| Add monitoring for X | SRE | Week 1 | Open |
| Increase Y capacity | Platform | Week 2 | Open |
| Update runbook | Eng | Week 1 | Open |

---

## Lessons Learned

[Key takeaways for future prevention]
```

**Distribution**: Engineering All-Hands, Leadership, Customer Support

**Follow-up**: Review action items in weekly incident review meeting

---

## Emergency Contacts

| Role | Primary | Backup | PagerDuty |
|------|---------|--------|-----------|
| **SRE Lead** | Name 1 | Name 2 | @sre-oncall |
| **Backend Lead** | Name 3 | Name 4 | @backend-oncall |
| **Database SRE** | Name 5 | Name 6 | @db-oncall |
| **VP Engineering** | Name 7 | - | @exec-escalation |

**War Room**: Zoom link in #incidents channel pinned message
