# SRE Practices & Incident Management — Complete Knowledge Guide

> **Purpose**: Understand Site Reliability Engineering principles, SLOs/SLIs/Error Budgets, incident management, chaos engineering, and capacity planning for building reliable production systems.

---

## 1. SRE Fundamentals

### What SRE Is (and Isn't)

```
SRE = Software Engineering approach to Operations

Traditional Ops:                    SRE:
├── Manual runbooks                ├── Automate everything
├── Reactive firefighting          ├── Proactive prevention
├── Separate from dev team         ├── Embedded with dev team
├── "Keep it running"              ├── "Make it better"
├── Ops does deploys               ├── Devs own their services
└── Measured by uptime             └── Measured by SLOs

Core Philosophy:
"Reliability is the MOST important feature.
If users can't use it, nothing else matters."

But also:
"100% reliability is the wrong target.
The cost of going from 99.99% to 99.999% is enormous.
The RIGHT target depends on your users' tolerance."
```

### The SRE Hierarchy of Needs

```
┌─────────────────────────────────────────────┐
│              Self-Healing                     │  ← Nirvana
│         (Auto-remediation)                   │
├─────────────────────────────────────────────┤
│           Capacity Planning                  │  ← Predictive
│      (Forecast & provision ahead)            │
├─────────────────────────────────────────────┤
│         Change Management                    │  ← Controlled
│     (Safe deployments, rollbacks)            │
├─────────────────────────────────────────────┤
│            Alerting                          │  ← Aware
│    (Know when things break)                  │
├─────────────────────────────────────────────┤
│           Observability                      │  ← Visible
│   (Metrics, Logs, Traces)                    │
├─────────────────────────────────────────────┤
│      Incident Response                       │  ← Reactive
│  (Handle outages quickly)                    │
├─────────────────────────────────────────────┤
│         Monitoring                           │  ← Basic
│  (Is it up or down?)                         │
└─────────────────────────────────────────────┘

Build from bottom up. You can't do capacity planning
without observability. You can't self-heal without alerting.
```

---

## 2. SLIs, SLOs, SLAs, and Error Budgets

### Definitions (Critical to Get Right)

```
SLI (Service Level INDICATOR):
├── A carefully chosen METRIC that measures user experience
├── Examples:
│   ├── "% of requests returning non-error response"
│   ├── "% of requests completing in < 300ms"
│   ├── "% of time the service is reachable"
│   └── "% of data processed within freshness window"
├── Formula: good events / total events × 100%
└── Key: MUST reflect what USERS care about (not CPU usage!)

SLO (Service Level OBJECTIVE):
├── A TARGET value for an SLI
├── Examples:
│   ├── "99.9% of requests succeed" (availability)
│   ├── "99% of requests complete in < 300ms" (latency)
│   └── "99.99% of data processed within 1 hour" (freshness)
├── Measured over a WINDOW (usually 30 days rolling)
└── Key: Agreed upon by eng team + product + business

SLA (Service Level AGREEMENT):
├── A CONTRACT with consequences if SLO is violated
├── Examples:
│   ├── "99.9% availability or 10% credit refund"
│   └── "If latency P99 > 1s, penalty clause triggered"
├── Typically LOOSER than internal SLO (buffer!)
└── Key: SLO should be tighter than SLA
    (e.g., SLO=99.95%, SLA=99.9% → 0.05% buffer)

ERROR BUDGET:
├── "How much unreliability are we ALLOWED?"
├── Formula: Error Budget = 1 - SLO
├── Example: SLO = 99.9% → Error Budget = 0.1%
│   └── In 30 days: 0.1% × 30d × 24h × 60m = 43.2 minutes of downtime ALLOWED
├── Key: Error budget is SPENT by:
│   ├── Production incidents (unplanned)
│   └── Risky deployments (planned risk)
└── When budget is exhausted → STOP taking risks (feature freeze)
```

### SLI Selection Guide

```
┌──────────────────────────────────────────────────────────────────┐
│  Service Type     │ SLI Category │ Measurement                   │
├───────────────────┼──────────────┼───────────────────────────────┤
│ Request-based     │ Availability │ % requests with status < 500  │
│ (APIs, web apps)  │ Latency      │ % requests < threshold        │
│                   │ Quality      │ % requests with correct data  │
├───────────────────┼──────────────┼───────────────────────────────┤
│ Pipeline/Batch    │ Freshness    │ % time data is < age threshold│
│ (ETL, data proc)  │ Correctness  │ % records processed correctly │
│                   │ Coverage     │ % of expected data present    │
├───────────────────┼──────────────┼───────────────────────────────┤
│ Storage           │ Durability   │ % data retrievable            │
│ (databases, S3)   │ Availability │ % time read/write succeeds   │
│                   │ Latency      │ % operations < threshold      │
└───────────────────┴──────────────┴───────────────────────────────┘

GOOD SLIs:
✅ "99.9% of HTTP requests return 2xx/3xx/4xx (not 5xx)"
✅ "95% of requests complete in < 200ms"
✅ "Data arrives in warehouse within 1 hour of generation, 99.9% of time"

BAD SLIs:
❌ "CPU utilization < 80%" (users don't care about CPU)
❌ "No alerts firing" (absence of alerts ≠ good experience)
❌ "100% availability" (impossible and unmeasurable)
```

### Error Budget Math (Interview Favorite!)

```
SCENARIO: Payment API with 99.95% availability SLO over 30 days

Error Budget:
= (1 - 0.9995) × 30 days × 24 hours × 60 minutes
= 0.0005 × 43,200 minutes
= 21.6 minutes of allowed downtime per month

Budget consumption this month:
├── Incident on June 3: 8 minutes of 500 errors → Budget spent: 8 min
├── Deployment issue June 10: 3 minutes → Budget spent: 3 min
├── AWS AZ issue June 18: 5 minutes → Budget spent: 5 min
└── Total spent: 16 minutes out of 21.6 minutes

Remaining budget: 5.6 minutes (26% remaining)
Status: ⚠️ CAUTION — approaching exhaustion

ACTIONS based on remaining budget:
├── > 50% remaining: Normal development velocity
├── 25-50% remaining: Extra caution, reduce deploy frequency
├── < 25% remaining: Reliability work only, no risky deploys
└── 0% (exhausted): Feature freeze until budget recovers
```

### Multi-Window, Multi-Burn-Rate Alerts

```
PROBLEM with simple SLO alerting:
"Alert when error rate > 0.1%" → fires on every tiny blip (noisy!)
"Alert when error rate > 0.1% for 30 minutes" → too slow for real outages!

SOLUTION: Burn rate alerting (how FAST are you consuming your budget?)

Burn Rate = actual error rate / maximum allowed error rate

Examples with 99.9% SLO (allowed error rate = 0.1%):
├── Burn rate 1x: Consuming budget at normal pace (fine)
├── Burn rate 14.4x: Budget gone in 2 hours (CRITICAL!)
├── Burn rate 6x: Budget gone in 5 hours (WARNING)
└── Burn rate 3x: Budget gone in 10 days (caution)

ALERT CONFIGURATION:
┌────────────────────────────────────────────────────────────────┐
│ Severity   │ Burn Rate │ Short Window │ Long Window │ Action  │
├────────────┼───────────┼──────────────┼─────────────┼─────────┤
│ PAGE (P1)  │ 14.4x     │ 2 minutes    │ 1 hour      │ Page    │
│ PAGE (P2)  │ 6x        │ 5 minutes    │ 6 hours     │ Page    │
│ TICKET     │ 3x        │ 30 minutes   │ 3 days      │ Ticket  │
└────────────┴───────────┴──────────────┴─────────────┴─────────┘

WHY two windows?
├── Short window: Detects fast (recent spike)
├── Long window: Confirms sustained (not just a blip)
└── Both must be true to fire → very low false-positive rate!
```

---

## 3. Incident Management

### Incident Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│  INCIDENT LIFECYCLE                                              │
│                                                                   │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│  │ DETECT   │──▶│ RESPOND  │──▶│ RESOLVE  │──▶│ LEARN    │   │
│  │          │   │          │   │          │   │          │   │
│  │ Monitor  │   │ Assemble │   │ Mitigate │   │ Postmortem│   │
│  │ Alert    │   │ Classify │   │ Fix root │   │ Action   │   │
│  │ Report   │   │ Communicate│  │ Verify   │   │ items    │   │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   │
│                                                                   │
│  Target Times:                                                    │
│  ├── Detection → Response: < 5 minutes (P1)                    │
│  ├── Response → Mitigation: < 30 minutes (P1)                  │
│  ├── Mitigation → Full Resolution: < 4 hours (P1)              │
│  └── Resolution → Postmortem: < 3 business days                 │
└─────────────────────────────────────────────────────────────────┘
```

### Incident Severity Levels

```
┌──────────────────────────────────────────────────────────────────┐
│ SEV │ Impact                    │ Response         │ Examples     │
├─────┼───────────────────────────┼──────────────────┼──────────────┤
│ P1  │ Critical: Major feature   │ All hands        │ Payment API  │
│     │ completely unavailable     │ Page immediately │ down, data   │
│     │ for majority of users     │ War room         │ loss, breach │
├─────┼───────────────────────────┼──────────────────┼──────────────┤
│ P2  │ High: Feature degraded,   │ On-call responds │ High latency,│
│     │ significant user impact   │ within 15 min    │ partial      │
│     │                           │                  │ outage       │
├─────┼───────────────────────────┼──────────────────┼──────────────┤
│ P3  │ Medium: Minor feature     │ Business hours   │ Non-critical │
│     │ impacted, workaround      │ Next day         │ endpoint     │
│     │ available                 │                  │ errors       │
├─────┼───────────────────────────┼──────────────────┼──────────────┤
│ P4  │ Low: Cosmetic or          │ Sprint backlog   │ Typo, minor  │
│     │ non-user-facing issue     │ When convenient  │ UI glitch    │
└─────┴───────────────────────────┴──────────────────┴──────────────┘
```

### Incident Commander (IC) Role

```
The IC is NOT the person fixing the problem.
The IC COORDINATES the response.

IC Responsibilities:
├── Declare incident severity
├── Assemble the response team
├── Assign roles (communication, investigation, mitigation)
├── Make decisions (escalate? rollback? failover?)
├── Keep stakeholders informed (status updates every 15-30 min)
├── Ensure handoffs if IC rotation needed
└── Call the incident resolved when appropriate

IC Does NOT:
├── Debug the code
├── SSH into servers
├── Write hotfixes
└── Get lost in technical details

Communication Template (every 15-30 min):
"[INCIDENT UPDATE - P1 - Payment API]
Status: Investigating
Impact: 15% of payment attempts failing
Current action: Team is investigating database connection issues
ETA: Next update in 15 minutes
Lead: @jane (IC)"
```

### Incident Communication

```
INTERNAL (Slack #incident-active):
├── Regular updates every 15 min (P1) or 30 min (P2)
├── Technical details for responders
├── Decision log (who decided what and why)
└── Timeline tracking

EXTERNAL (Status page):
├── Initial: "Investigating increased error rates"
├── Update: "Identified issue with payment processing"
├── Update: "Fix deployed, monitoring recovery"
├── Resolved: "Issue resolved, service operating normally"
└── Postmortem: "Details available at [link]"

RULES:
├── Never say "it's fixed" until you've VERIFIED (false resolution = worse)
├── Be honest about impact (don't downplay)
├── Give ETAs only when you're confident
├── "We're investigating" is OK (better than silence)
└── Update even if nothing changed ("Still investigating, no new info")
```

---

## 4. Postmortems (Blameless Retrospectives)

### The Blameless Culture

```
BLAME CULTURE:                      BLAMELESS CULTURE:
"Who caused this?"                  "What caused this?"
"Person X made a mistake"           "The SYSTEM allowed this mistake"
"Person X should be punished"       "How do we prevent this for ANYONE?"
"People hide mistakes"              "People report mistakes openly"
"Same mistakes repeat"              "Systemic improvements"

KEY PRINCIPLE:
"If someone can make a mistake that causes an outage,
the SYSTEM is at fault, not the person.
A well-designed system prevents or mitigates human error."

Example:
BAD: "John deployed bad code to production"
GOOD: "Our deployment pipeline lacked automated rollback.
       The pre-production test environment didn't match production.
       There was no canary stage to catch errors early."
```

### Postmortem Template

```markdown
# Incident Postmortem: [Title]

## Summary
- **Date**: 2024-06-15
- **Duration**: 47 minutes
- **Severity**: P1
- **Impact**: 25% of payment transactions failed
- **Root Cause**: Database connection pool exhausted due to connection leak
- **Detection**: CloudWatch alarm on 5xx error rate

## Timeline
| Time (UTC) | Event |
|------------|-------|
| 10:00 | Deployment of payment-api v2.3.1 |
| 10:15 | Connection pool starts filling (not detected yet) |
| 10:32 | CloudWatch alarm fires: 5xx > 1% |
| 10:35 | On-call acknowledges, begins investigation |
| 10:42 | Root cause identified: connection leak in v2.3.1 |
| 10:45 | Decision: rollback to v2.3.0 |
| 10:52 | Rollback complete, connections draining |
| 11:19 | All connections released, service fully recovered |
| 11:19 | Incident declared resolved |

## Root Cause
The v2.3.1 release introduced a database connection leak in the
retry logic path. When a database query timed out, the connection
was not returned to the pool. Under load, this gradually exhausted
the pool (max 50 connections).

## Contributing Factors
1. Unit tests didn't cover the timeout-retry code path
2. Load testing in staging used a smaller dataset (didn't trigger timeouts)
3. Connection pool monitoring existed but alarm was set too high (45/50)
4. Canary deployment was skipped for this "minor" release

## What Went Well
- CloudWatch alarm detected the issue within 2 minutes of user impact
- On-call response was fast (3 minutes to acknowledge)
- Rollback procedure worked correctly
- Communication was clear and timely

## What Went Wrong
- 15 minutes between deploy and detection (slow leak)
- No automated rollback on error rate increase
- Connection pool alarm threshold too high

## Action Items
| Priority | Action | Owner | Due |
|----------|--------|-------|-----|
| P1 | Add connection pool size to deployment health check | @alice | June 20 |
| P1 | Enable CodeDeploy auto-rollback on 5xx alarm | @bob | June 18 |
| P2 | Add integration test for DB timeout retry path | @charlie | June 25 |
| P2 | Lower connection pool alarm to 35/50 (70%) | @alice | June 17 |
| P3 | Add load test scenario with DB timeouts | @dave | July 1 |
| P3 | Require canary stage for ALL deployments | @platform | July 5 |

## Lessons Learned
- "Minor" releases can have major impact
- Connection pool health should be a deployment gate
- Canary deployments should never be skipped
```

---

## 5. Chaos Engineering

### What Chaos Engineering Is

```
"The discipline of experimenting on a system to build confidence
in the system's capability to withstand turbulent conditions
in production." — Principles of Chaos Engineering

NOT: "Let's break things for fun"
IS:  "Let's PROVE our systems handle failures correctly"

The Scientific Method Applied to Reliability:
1. HYPOTHESIZE: "If AZ-a fails, traffic shifts to AZ-b with no errors"
2. EXPERIMENT: Terminate all instances in AZ-a
3. OBSERVE: Did traffic shift? Were there errors? How long?
4. CONCLUDE: Hypothesis confirmed/denied
5. IMPROVE: Fix any gaps discovered
```

### AWS Fault Injection Simulator (FIS)

```
FIS lets you inject failures safely:

Available Fault Types:
├── EC2:
│   ├── Terminate instances
│   ├── Stop instances
│   └── CPU/Memory stress
├── ECS:
│   ├── Stop tasks
│   └── Container crash
├── RDS:
│   ├── Failover
│   └── Reboot
├── Network:
│   ├── Packet loss (10%, 50%, 100%)
│   ├── Latency injection (100ms, 500ms, 2s)
│   └── DNS failure
├── SSM:
│   ├── Run stress commands on instances
│   └── Kill processes
└── EKS:
    ├── Delete pods
    └── Node drain

SAFETY CONTROLS (Guardrails):
├── Stop conditions: "If error rate > 5%, abort experiment"
├── Blast radius limits: "Only affect 10% of instances"
├── Duration limits: "Maximum 10 minutes"
├── Rollback: Automatic when experiment ends
└── Scheduling: Only during business hours with team present
```

### Chaos Engineering Maturity

```
Level 1: "Can we survive ONE instance failure?"
├── Terminate 1 EC2 instance
├── Expected: Auto Scaling replaces, no user impact
├── Frequency: Weekly
└── Confidence gained: Basic redundancy works

Level 2: "Can we survive ONE AZ failure?"
├── Terminate all instances in 1 AZ
├── Expected: Multi-AZ architecture handles, SLO maintained
├── Frequency: Monthly
└── Confidence gained: AZ-level redundancy works

Level 3: "Can we survive a dependency failure?"
├── Block network to database / external API
├── Expected: Circuit breaker activates, graceful degradation
├── Frequency: Monthly
└── Confidence gained: Failure isolation works

Level 4: "Can we survive a regional failure?"
├── Simulate complete region unavailability
├── Expected: Failover to DR region within RTO
├── Frequency: Quarterly
└── Confidence gained: DR actually works

Level 5: "Can we survive during peak traffic + failure?"
├── Inject failure during highest traffic period
├── Expected: System maintains SLO even under combined stress
├── Frequency: Semi-annually
└── Confidence gained: Ready for real-world scenarios
```

---

## 6. Capacity Planning

### The Capacity Planning Process

```
1. MEASURE current capacity usage
   ├── CPU utilization trends (average and peak)
   ├── Memory usage patterns
   ├── Network throughput
   ├── Storage growth rate
   └── Request rate and queue depth

2. FORECAST future demand
   ├── Historical growth rate (linear or exponential?)
   ├── Planned events (product launch, marketing campaign)
   ├── Seasonal patterns (Black Friday, month-end)
   └── Business projections (new features, new markets)

3. MODEL capacity needs
   ├── Current capacity: 10,000 requests/sec
   ├── Current utilization: 60% at peak
   ├── Growth rate: 20% per quarter
   ├── Next quarter peak: 10,000 × 1.2 = 12,000 req/sec
   ├── Required headroom: 30% for unexpected spikes
   ├── Target capacity: 12,000 / 0.7 = 17,142 req/sec
   └── Action: Scale from 10K to 17K capacity (add 70% more compute)

4. PROVISION with headroom
   ├── Don't provision at 100% of projected need
   ├── Rule of thumb: Stay below 70% utilization at peak
   ├── Auto-scaling handles short bursts (but has lag!)
   └── Pre-provision for known events (don't rely on auto-scale alone)
```

### Auto-Scaling Best Practices

```
REACTIVE Auto-Scaling (most common):
├── Scale OUT when CPU > 70% for 3 minutes
├── Scale IN when CPU < 30% for 10 minutes
├── Cooldown: 300 seconds (prevent thrashing)
├── Problem: 3+ minutes to detect + provision = LAG
└── Not sufficient for sudden traffic spikes!

PREDICTIVE Auto-Scaling (better for known patterns):
├── ML analyzes historical traffic patterns
├── Pre-scales BEFORE traffic arrives
├── Example: Scale up at 8:50 AM (before 9 AM rush)
└── Available for: EC2 Auto Scaling, ECS Service Auto Scaling

SCHEDULED Auto-Scaling (best for known events):
├── Scale to specific capacity at specific time
├── Example: 2x capacity every Monday 9 AM
├── Example: 5x capacity on Black Friday
└── Most reliable for known traffic patterns

COMBINATION (production best practice):
├── Base: Scheduled scaling for known patterns
├── Mid: Predictive scaling for daily variations
└── Safety: Reactive scaling for unexpected spikes
```

---

## 7. On-Call Best Practices

### Sustainable On-Call

```
ON-CALL HEALTH INDICATORS:

Healthy on-call:
├── < 2 pages per on-call shift (8 hours)
├── > 80% of alerts are actionable
├── Average incident resolution < 30 minutes
├── On-call engineer gets adequate sleep
└── Rotation: minimum 1 week on, 3 weeks off

Unhealthy on-call (burnout risk!):
├── > 5 pages per shift
├── False alarm rate > 50%
├── Engineer firefighting entire shift
├── Same issues recurring (not being fixed)
└── Engineers dread on-call rotation

FIXES for unhealthy on-call:
├── Review and silence noisy alerts (monthly alert hygiene)
├── Fix recurring issues permanently (not just patches)
├── Automate common remediations (Lambda auto-fix)
├── Improve detection speed (reduce incident duration)
├── Add team members to rotation (spread the load)
└── Implement error budgets (pause features when unstable)
```

### On-Call Runbook Template

```markdown
# Runbook: [Alert Name]

## Alert Meaning
What this alert indicates in plain language.

## Severity & Impact
- Severity: P1/P2/P3
- User impact: [What users experience]
- Business impact: [Revenue/reputation impact]

## First Response (< 5 minutes)
1. Check dashboard: [link]
2. Check recent deployments: [link]
3. Check dependency status: [link]

## Diagnosis Steps
1. [Specific command to run]
2. [What to look for in logs]
3. [Metrics to check]

## Mitigation Options
### Option A: Rollback (if recent deployment)
```bash
aws ecs update-service --cluster prod --service app --task-definition app:PREV_VERSION
```

### Option B: Scale up (if capacity issue)
```bash
aws ecs update-service --cluster prod --service app --desired-count 10
```

### Option C: Failover (if regional issue)
[Link to DR failover runbook]

## Escalation
- If not resolved in 15 min → Escalate to [team lead]
- If not resolved in 30 min → Escalate to [VP Eng]
- If data integrity risk → Page [DBA + Security]

## Previous Incidents
- 2024-06-01: Similar issue caused by [X]. Resolution: [Y]
- 2024-05-15: Related alert due to [Z]. Resolution: [W]
```

---

## 8. Reliability Patterns

### Circuit Breaker

```
PURPOSE: Prevent cascading failures when a downstream service is failing.

STATES:
┌──────────┐         ┌──────────┐         ┌──────────┐
│  CLOSED  │──fail──▶│  OPEN    │──timer──▶│HALF-OPEN │
│ (normal) │         │ (reject) │         │ (testing) │
└──────────┘         └──────────┘         └──────────┘
     ▲                                         │
     └────────── success ──────────────────────┘

CLOSED: All requests pass through normally
├── Track failure count
├── When failures > threshold → Switch to OPEN

OPEN: All requests IMMEDIATELY FAIL (no calling downstream)
├── Return fallback response or cached data
├── After timeout (30s) → Switch to HALF-OPEN

HALF-OPEN: Let ONE request through to test
├── If succeeds → Switch to CLOSED (recovered!)
├── If fails → Switch back to OPEN (still broken)

WHY it helps:
├── Downstream service gets time to recover (no flood of retries)
├── YOUR service stays responsive (fails fast instead of waiting)
├── Users get degraded but FAST response (vs timeout)
└── System recovers automatically when downstream is healthy
```

### Bulkhead Pattern

```
PURPOSE: Isolate failures to prevent one bad component from
taking down everything.

Like a ship's bulkhead (compartments prevent total flooding):

WITHOUT Bulkhead:
┌────────────────────────────────────────────┐
│  One shared thread pool: 100 threads       │
│                                            │
│  Service A calls (normal): 20 threads      │
│  Service B calls (SLOW!): 80 threads ← Consuming all resources!
│  Service C calls: 0 threads ← STARVED! Can't process!
│                                            │
│  Result: One slow dependency kills everything
└────────────────────────────────────────────┘

WITH Bulkhead:
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ Pool A: 30    │  │ Pool B: 30    │  │ Pool C: 30    │
│ Service A     │  │ Service B     │  │ Service C     │
│ calls         │  │ calls (SLOW!) │  │ calls         │
│ 20/30 used    │  │ 30/30 FULL    │  │ 10/30 used    │
└───────────────┘  └───────────────┘  └───────────────┘

Service B is slow → Only its pool is exhausted
Services A and C continue normally! Failure is CONTAINED.
```

### Retry with Exponential Backoff

```
PURPOSE: Handle transient failures without overwhelming the system.

BAD retry (hammers the service):
Fail → Retry immediately → Fail → Retry immediately → Fail...
(Makes things WORSE by adding load to struggling service!)

GOOD retry (exponential backoff + jitter):
Fail → Wait 1s → Retry → Wait 2s → Retry → Wait 4s → Retry → Give up

With JITTER (randomness prevents thundering herd):
Client A: Wait 0.8s → 2.3s → 3.7s
Client B: Wait 1.2s → 1.8s → 4.5s
Client C: Wait 0.5s → 2.9s → 5.1s
(All retry at DIFFERENT times → distribute load)

IMPLEMENTATION:
delay = min(base_delay * 2^attempt + random(0, 1000ms), max_delay)

attempt 0: 1000ms + random → ~1-2 seconds
attempt 1: 2000ms + random → ~2-3 seconds  
attempt 2: 4000ms + random → ~4-5 seconds
attempt 3: GIVE UP (or circuit breaker opens)
```

### Load Shedding

```
PURPOSE: When overloaded, REJECT some requests to keep the
service responsive for remaining requests.

WITHOUT load shedding:
1000 requests arrive, server can handle 500
→ ALL 1000 requests queue up
→ ALL responses are slow (2+ seconds)
→ ALL users unhappy

WITH load shedding:
1000 requests arrive, server can handle 500
→ 500 are processed normally (200ms response)
→ 500 are REJECTED immediately (503 with Retry-After)
→ 50% of users get fast response, 50% retry later
→ NET: Better user experience overall

IMPLEMENTATION:
├── Request queue length > threshold → reject new requests
├── Response time > threshold → start shedding
├── CPU > threshold → activate load shedding
└── Prioritize: Critical paths served first, non-essential shed first
```

---

## 9. Disaster Recovery

### DR Strategies Summary

```
┌────────────────────────────────────────────────────────────┐
│ Strategy          │ RTO        │ RPO       │ Cost    │     │
├───────────────────┼────────────┼───────────┼─────────┼─────┤
│ Backup & Restore  │ 24 hours   │ Hours     │ $       │     │
│ Pilot Light       │ 1-4 hours  │ Minutes   │ $$      │     │
│ Warm Standby      │ 15-60 min  │ Minutes   │ $$$     │     │
│ Active-Active     │ Seconds    │ Near-zero │ $$$$    │     │
└───────────────────┴────────────┴───────────┴─────────┴─────┘

RTO = Recovery Time Objective (how long to restore service)
RPO = Recovery Point Objective (how much data you can lose)

Choose based on BUSINESS IMPACT of downtime:
├── Revenue per minute of outage
├── Regulatory requirements
├── Customer trust impact
└── Competitive pressure

Example:
├── Payment processing: Active-Active (every minute = lost revenue)
├── Customer portal: Warm Standby (15 min downtime tolerable)
├── Internal tools: Pilot Light (4 hour RTO acceptable)
└── Dev environments: Backup & Restore (rebuild from code)
```

---

## 10. Key Formulas and Numbers

```
AVAILABILITY MATH:
99% = 7.3 hours downtime/month (3.6 days/year)
99.9% = 43.8 minutes/month (8.7 hours/year)
99.95% = 21.9 minutes/month (4.4 hours/year)
99.99% = 4.38 minutes/month (52.6 minutes/year)
99.999% = 26 seconds/month (5.26 minutes/year)

COMPOSITE AVAILABILITY:
Serial: A_total = A1 × A2 × A3
(Service A → Service B → Service C)
Example: 99.9% × 99.9% × 99.9% = 99.7%
(Three 3-nines services in series = less than 3 nines!)

Parallel: A_total = 1 - (1-A1) × (1-A2)
(Service A OR Service B, either works)
Example: 1 - (1-0.999) × (1-0.999) = 99.9999%
(Two 3-nines services in parallel = almost 6 nines!)

LESSON: Redundancy (parallel) dramatically improves availability.
        More dependencies (serial) dramatically reduces it.

MTTR (Mean Time To Recovery):
= Sum(incident durations) / Number of incidents
Goal: < 30 minutes for P1

MTBF (Mean Time Between Failures):
= Total uptime / Number of failures
Goal: Increasing over time (fewer incidents)

MTTD (Mean Time To Detect):
= Average time from failure start to detection
Goal: < 5 minutes for P1
```

---

## 11. Learning Path

```
Beginner (Week 1-2):
├── Understand SLI/SLO/SLA definitions
├── Calculate error budgets
├── Write your first runbook
├── Set up basic monitoring and alerting
├── Join an on-call rotation (shadow first!)
└── Read: "Site Reliability Engineering" (Google SRE Book, free online)

Intermediate (Week 3-6):
├── Implement SLO-based alerting (burn rate)
├── Run a postmortem (from real or simulated incident)
├── Implement circuit breaker pattern in code
├── Run basic chaos experiments (kill one instance)
├── Design deployment strategy with auto-rollback
├── Practice incident response (tabletop exercise)
└── Read: "The Site Reliability Workbook" (practical exercises)

Advanced (Month 2-4):
├── Design SLO framework for 10+ services
├── Implement error budget policies with stakeholders
├── Run chaos experiments at AZ level
├── Build auto-remediation (detect + fix without human)
├── Capacity planning with forecasting
├── Design DR strategy and test via Game Day
├── Implement progressive delivery (canary + automated analysis)
└── Read: "Implementing Service Level Objectives" (Alex Hidalgo)

Expert (Month 5+):
├── Build SRE team and define responsibilities
├── Implement platform-wide reliability standards
├── Design for 99.99%+ availability
├── Multi-region active-active architecture
├── Lead Game Days with executive participation
├── Build self-healing systems (auto-remediation at scale)
├── Define and enforce error budget policies across org
└── Mentor junior SREs, build SRE culture
```
