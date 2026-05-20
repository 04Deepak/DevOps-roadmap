# 13. SRE Practices & Chaos Engineering

## Table of Contents
- [SRE Principles & Philosophy](#sre-principles--philosophy)
- [SLIs, SLOs, and SLAs](#slis-slos-and-slas)
- [Error Budgets](#error-budgets)
- [Incident Management](#incident-management)
- [Postmortems & Blameless Retrospectives](#postmortems--blameless-retrospectives)
- [Chaos Engineering Principles](#chaos-engineering-principles)
- [Litmus ChaosCenter](#litmus-chaoscenter)
- [Chaos Monkey](#chaos-monkey)
- [Load Testing with k6](#load-testing-with-k6)
- [Projects](#projects)

---

## SRE Principles & Philosophy

Site Reliability Engineering (SRE) is a discipline that applies software engineering practices to infrastructure and operations problems. It was pioneered at Google and is defined by treating operations as a software problem.

### Core SRE Principles

```
1. EMBRACE RISK
   - 100% reliability is the wrong target (too expensive, slows innovation)
   - Define an acceptable level of unreliability (error budget)
   - Use the error budget to balance reliability vs feature velocity

2. SERVICE LEVEL OBJECTIVES (SLOs)
   - Every service needs clearly defined reliability targets
   - SLOs are the primary tool for making data-driven reliability decisions
   - If the error budget is exhausted, freeze feature releases and fix reliability

3. ELIMINATE TOIL
   - Toil = manual, repetitive, automatable, reactive work with no lasting value
   - SREs should spend no more than 50% of time on toil
   - The remaining 50%+ goes to engineering work (automation, tooling, design)

4. MONITORING & ALERTING
   - Monitor symptoms (user-facing), not just causes (CPU, memory)
   - Alerts should be actionable -- every page should require human intervention
   - Reduce alert fatigue: eliminate noisy, non-actionable alerts

5. AUTOMATION
   - Automate yourself out of a job (then find harder problems)
   - Prefer consistent automated responses over manual intervention
   - Runbooks for anything that cannot yet be automated

6. RELEASE ENGINEERING
   - Frequent, small releases are safer than rare, large releases
   - Canary releases, feature flags, and progressive rollouts reduce risk
   - Rollback must always be faster than rolling forward

7. SIMPLICITY
   - Simple systems are more reliable, easier to understand, and easier to fix
   - Resist unnecessary complexity in both software and processes
```

### SRE vs Traditional Ops

```
Aspect              | Traditional Ops           | SRE
--------------------|---------------------------|---------------------------
Mindset             | Keep things running       | Engineer reliability
Change Approach     | Resist change (risk)      | Embrace change (with SLOs)
Manual Work         | Expected                  | Called "toil", eliminated
Reliability Target  | "Five nines" always       | Just enough (error budget)
On-Call             | Operations team           | Developers on rotation
Incident Response   | Blame individuals         | Blameless postmortems
Success Metric      | Uptime                    | SLO compliance + velocity
```

---

## SLIs, SLOs, and SLAs

These three concepts form the foundation of SRE reliability management.

### Definitions

```
SLI (Service Level Indicator)
  = A quantitative measure of a service's behavior
  = The METRIC you measure
  Examples: request latency, error rate, throughput, availability

SLO (Service Level Objective)
  = A target value or range for an SLI
  = The TARGET you aim for
  Examples: "99.9% of requests complete in < 200ms"

SLA (Service Level Agreement)
  = A contract with consequences if the SLO is not met
  = The PROMISE you make to customers (with penalties)
  Examples: "99.9% uptime or customer gets service credits"

Relationship:
  SLI (what you measure) --> SLO (what you aim for) --> SLA (what you promise)
  
  SLOs should be STRICTER than SLAs
  (so you catch problems before breaching the contract)
```

### Common SLI Types

```
1. AVAILABILITY
   SLI: Proportion of successful requests
   Formula: successful_requests / total_requests
   
   Example:
     Total requests in 30 days: 10,000,000
     Failed requests (5xx):     5,000
     Availability SLI = (10,000,000 - 5,000) / 10,000,000 = 99.95%

2. LATENCY
   SLI: Proportion of requests faster than a threshold
   Formula: requests_below_threshold / total_requests
   
   Example:
     Total requests: 10,000,000
     Requests under 200ms: 9,850,000
     Latency SLI (p50): 45ms
     Latency SLI (p99): 180ms
     Proportion under 200ms = 9,850,000 / 10,000,000 = 98.5%

3. THROUGHPUT
   SLI: Requests processed per second
   Formula: total_requests / time_period
   
   Example:
     Requests in 1 hour: 360,000
     Throughput SLI = 360,000 / 3600 = 100 req/s

4. FRESHNESS (for data pipelines)
   SLI: Proportion of data updated within a threshold
   Formula: records_updated_within_threshold / total_records
   
   Example:
     Total records: 1,000,000
     Updated within 5 minutes: 990,000
     Freshness SLI = 990,000 / 1,000,000 = 99.0%
```

### Defining SLOs -- Concrete Examples

```yaml
# slo-definition.yaml (documentation format)
service: payment-api
owner: payments-team
slos:
  - name: Availability
    sli: "Proportion of non-5xx responses"
    target: 99.95%
    window: 30 days (rolling)
    measurement: |
      sum(rate(http_requests_total{status!~"5.."}[30d]))
      /
      sum(rate(http_requests_total[30d]))

  - name: Latency (p99)
    sli: "99th percentile request duration"
    target: "99% of requests < 500ms"
    window: 30 days (rolling)
    measurement: |
      histogram_quantile(0.99,
        sum(rate(http_request_duration_seconds_bucket[30d])) by (le)
      ) < 0.5

  - name: Latency (p50)
    sli: "50th percentile request duration"
    target: "50% of requests < 100ms"
    window: 30 days (rolling)
    measurement: |
      histogram_quantile(0.50,
        sum(rate(http_request_duration_seconds_bucket[30d])) by (le)
      ) < 0.1
```

### SLO Calculation Example

```
Service: E-commerce Checkout API
SLO: 99.9% availability over 30 days

Allowed downtime in 30 days:
  30 days * 24 hours * 60 minutes = 43,200 minutes total
  43,200 * (1 - 0.999) = 43,200 * 0.001 = 43.2 minutes of downtime allowed

In terms of failed requests:
  If we handle 1,000,000 requests/day over 30 days = 30,000,000 requests
  Allowed failures = 30,000,000 * 0.001 = 30,000 failed requests

Nines Table (per 30 days):
  99%    (two nines)   = 432 minutes  = 7.2 hours downtime
  99.9%  (three nines) = 43.2 minutes downtime
  99.95%               = 21.6 minutes downtime
  99.99% (four nines)  = 4.32 minutes downtime
  99.999%(five nines)  = 0.43 minutes = 26 seconds downtime
```

### Prometheus Alerting Rules for SLOs

```yaml
# slo-alerts.yaml
# Prometheus alerting rules based on SLO burn rate
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-alerts
  namespace: monitoring
spec:
  groups:
    - name: slo.availability
      rules:
        # Fast burn alert: detect issues burning error budget quickly
        # (consuming 2% of 30-day budget in 1 hour)
        - alert: HighErrorBurnRate_Fast
          expr: |
            (
              sum(rate(http_requests_total{status=~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            ) > 14.4 * 0.001
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "High error burn rate (fast) for {{ $labels.service }}"
            description: >
              Error rate is burning through the error budget at 14.4x the
              allowed rate. At this pace, the 30-day error budget will be
              exhausted in approximately 2 days.

        # Slow burn alert: detect sustained elevated error rates
        # (consuming 5% of 30-day budget in 6 hours)
        - alert: HighErrorBurnRate_Slow
          expr: |
            (
              sum(rate(http_requests_total{status=~"5.."}[6h]))
              /
              sum(rate(http_requests_total[6h]))
            ) > 6 * 0.001
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Elevated error burn rate (slow) for {{ $labels.service }}"
            description: >
              Error rate is burning through the error budget at 6x the
              allowed rate over the last 6 hours.

    - name: slo.latency
      rules:
        - alert: HighLatency_P99
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
            ) > 0.5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "P99 latency exceeds 500ms SLO"
```

---

## Error Budgets

An error budget is the maximum amount of unreliability your SLO allows. It represents the difference between perfect reliability (100%) and your SLO target.

### Error Budget Calculation

```
SLO: 99.9% availability

Error Budget = 100% - 99.9% = 0.1%

Over a 30-day window:
  Total minutes: 30 * 24 * 60 = 43,200 minutes
  Error budget:  43,200 * 0.001 = 43.2 minutes

This means:
  - You can have 43.2 minutes of downtime per 30-day window
  - OR ~30,000 failed requests out of 30,000,000
  - Once the budget is spent, you must freeze feature deployments
    and focus entirely on reliability improvements
```

### Error Budget Policy

```
Error Budget Policy for Payment Service
========================================

SLO: 99.95% availability (30-day rolling window)
Error Budget: 0.05% = 21.6 minutes of downtime per 30 days

Budget Remaining    | Actions
--------------------|--------------------------------------------------
> 50% remaining     | Normal feature development and deployment pace
                    | Standard deployment procedures
25% - 50% remaining| Increase monitoring and alerting sensitivity
                    | Require extra review for risky deployments
                    | Begin investigating top reliability issues
10% - 25% remaining| Reduce deployment frequency
                    | Mandatory canary deployments for all changes
                    | SRE team reviews all production changes
< 10% remaining    | FREEZE all non-critical feature deployments
                    | Engineering effort redirected to reliability
                    | Daily reliability standup meetings
0% (exhausted)     | COMPLETE feature freeze
                    | All engineering works on reliability
                    | Postmortem required before resuming deployments
                    | Freeze lifted only after buffer is restored
```

### Error Budget Tracking Dashboard (Prometheus + Grafana)

```yaml
# Grafana dashboard query examples for error budget tracking

# Current availability (last 30 days)
# Panel: Stat showing "99.97%"
query: |
  sum(rate(http_requests_total{status!~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
  * 100

# Error budget remaining (percentage)
# Panel: Gauge from 0% to 100%
query: |
  (
    1 - (
      (1 - (
        sum(rate(http_requests_total{status!~"5.."}[30d]))
        /
        sum(rate(http_requests_total[30d]))
      ))
      /
      (1 - 0.9995)
    )
  ) * 100

# Error budget remaining (minutes)
# Panel: Stat showing "15.3 minutes"
query: |
  (
    1 - (
      (1 - (
        sum(rate(http_requests_total{status!~"5.."}[30d]))
        /
        sum(rate(http_requests_total[30d]))
      ))
      /
      (1 - 0.9995)
    )
  ) * 43200 / 60

# Error budget burn rate (should be <= 1.0)
# Panel: Time series graph
query: |
  (
    sum(rate(http_requests_total{status=~"5.."}[1h]))
    /
    sum(rate(http_requests_total[1h]))
  )
  /
  (1 - 0.9995)
```

---

## Incident Management

Incident management is the process of detecting, responding to, and resolving service disruptions.

### Incident Severity Levels

```
SEV1 (Critical)
  Impact: Complete service outage for all users
  Response Time: < 5 minutes
  Communication: Every 15 minutes to stakeholders
  Example: Payment processing completely down
  Who is paged: On-call SRE + engineering lead + VP Engineering

SEV2 (Major)
  Impact: Significant degradation affecting many users
  Response Time: < 15 minutes
  Communication: Every 30 minutes
  Example: Checkout latency increased 10x, many timeouts
  Who is paged: On-call SRE + team lead

SEV3 (Minor)
  Impact: Partial degradation, workaround available
  Response Time: < 1 hour
  Communication: At start and resolution
  Example: Image thumbnails not loading, text content works
  Who is paged: On-call SRE (notification, not page)

SEV4 (Low)
  Impact: Cosmetic issue or minor bug
  Response Time: Next business day
  Communication: Tracked in issue tracker
  Example: Footer link broken on one page
  Who is paged: Nobody -- ticket created
```

### Incident Response Process

```
1. DETECT
   - Automated monitoring alerts (Prometheus, PagerDuty)
   - Customer reports
   - Internal team reports
   
2. TRIAGE
   - Assess severity level
   - Assign incident commander (IC)
   - Create incident channel (e.g., #incident-2024-0042)

3. RESPOND
   Incident Commander responsibilities:
   - Coordinates the response (does NOT debug directly)
   - Assigns roles: investigation lead, communications lead
   - Makes decisions on mitigations
   - Tracks timeline of events
   
   Investigation Lead:
   - Diagnoses the root cause
   - Proposes and implements mitigations
   - Communicates technical details to IC

   Communications Lead:
   - Posts status updates to stakeholders
   - Updates status page
   - Handles customer-facing communication

4. MITIGATE
   Priority: Restore service first, investigate root cause later
   Common mitigations:
   - Rollback recent deployment
   - Scale up resources
   - Redirect traffic (failover)
   - Enable feature flag kill switch
   - Restart services

5. RESOLVE
   - Confirm service is fully restored
   - Verify monitoring shows normal metrics
   - Stand down the incident team
   - Schedule postmortem within 48 hours

6. FOLLOW UP
   - Write blameless postmortem
   - Create action items with owners and deadlines
   - Share learnings with the broader organization
```

### On-Call Rotation Setup

```yaml
# pagerduty-schedule-example.yaml
# This represents a typical on-call rotation structure

rotation:
  name: "Backend SRE On-Call"
  timezone: "America/New_York"
  
  primary:
    rotation_type: weekly       # Each person is on-call for 1 week
    handoff_time: "09:00"       # Monday at 9 AM
    members:
      - alice@company.com
      - bob@company.com
      - charlie@company.com
      - diana@company.com
    # 4 people = each person is on-call 1 week out of 4

  secondary:                    # Backup on-call (escalation)
    rotation_type: weekly
    handoff_time: "09:00"
    members:
      - bob@company.com          # Secondary is the NEXT week's primary
      - charlie@company.com
      - diana@company.com
      - alice@company.com

  escalation_policy:
    - level: 1
      target: primary
      timeout: 5m               # If no acknowledgment in 5 minutes
    - level: 2
      target: secondary
      timeout: 10m
    - level: 3
      target: engineering-manager
      timeout: 15m
```

### On-Call Best Practices

```
DO:
  - Limit on-call shifts to 1 week maximum
  - Provide at least 1 week off between on-call shifts
  - Compensate for on-call time (time off, extra pay)
  - Maintain runbooks for common incidents
  - Have a secondary on-call as backup
  - Conduct regular game days (practice incidents)
  - Review on-call load quarterly (balance alert volume)

DON'T:
  - Page for non-actionable alerts
  - Have a single person always on-call
  - Page for issues that can wait until business hours
  - Skip postmortems for recurring incidents
  - Ignore on-call burnout
```

### Incident Runbook Template

```markdown
# Runbook: Database Connection Pool Exhaustion

## Symptoms
- Application returns 500 errors intermittently
- Alert: "DatabaseConnectionPoolExhausted" firing
- Grafana dashboard shows connection pool at 100%

## Impact
- SEV2: Users experience intermittent failures on read/write operations

## Diagnosis Steps
1. Check connection pool metrics:
   kubectl exec -it <app-pod> -- curl localhost:8080/metrics | grep db_pool

2. Check for long-running queries:
   kubectl exec -it <db-pod> -- psql -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC LIMIT 10;"

3. Check if recent deployment changed query patterns:
   kubectl rollout history deployment/my-app

## Mitigation Steps
1. IMMEDIATE: Restart application pods to release connections
   kubectl rollout restart deployment/my-app

2. If caused by long-running queries, terminate them:
   kubectl exec -it <db-pod> -- psql -c "SELECT pg_terminate_backend(<pid>);"

3. If caused by traffic spike, scale up:
   kubectl scale deployment/my-app --replicas=10

4. If caused by a bad deployment, rollback:
   kubectl rollout undo deployment/my-app

## Root Cause Investigation
- Check for N+1 query patterns in recent code changes
- Verify connection pool configuration (max connections, timeout)
- Check for connection leaks (connections not returned to pool)

## Prevention
- Add connection pool monitoring and alerting
- Set query timeouts at the application level
- Implement circuit breaker for database connections
```

---

## Postmortems & Blameless Retrospectives

A postmortem is a structured review of an incident, focused on learning and preventing recurrence, not on blame.

### Blameless Culture

```
BLAMELESS means:
  - Focus on WHAT went wrong, not WHO did something wrong
  - Assume everyone acted with the best intentions and information available
  - The system failed, not the person
  - "Human error" is never a root cause -- ask WHY the system allowed it

BLAMELESS does NOT mean:
  - No accountability
  - No action items
  - Ignoring negligence or repeated mistakes
  - Avoiding difficult conversations

Example of BLAME:
  "Bob deployed the bad config that caused the outage."

Example of BLAMELESS:
  "A configuration change was deployed that contained an error.
   The deployment pipeline lacked validation checks that would
   have caught the misconfiguration before it reached production."
```

### Postmortem Template

```markdown
# Postmortem: Payment Service Outage
# Date: 2025-03-15
# Duration: 47 minutes (14:23 UTC - 15:10 UTC)
# Severity: SEV1
# Author: Alice Chen
# Status: Action items in progress

## Executive Summary
The payment service experienced a complete outage lasting 47 minutes,
affecting all users attempting to complete purchases. The root cause
was an expired TLS certificate on the payment gateway integration.
Approximately 12,000 transactions failed during the incident.

## Impact
- Duration: 47 minutes
- Users affected: ~8,500 active users
- Revenue impact: ~$45,000 in failed transactions
- Transactions failed: ~12,000
- SLO impact: Consumed 35% of monthly error budget

## Timeline (all times UTC)
- 14:20 - TLS certificate for payment gateway expires
- 14:23 - Monitoring alert fires: "PaymentGateway5xxRate > 50%"
- 14:25 - On-call SRE (Alice) acknowledges alert
- 14:28 - Incident declared as SEV1, channel #inc-2025-0042 created
- 14:30 - Investigation begins, initial suspicion: gateway service down
- 14:35 - Logs reveal TLS handshake failures
- 14:38 - Root cause identified: expired TLS certificate
- 14:45 - Certificate renewal initiated via ACME/Let's Encrypt
- 14:55 - New certificate issued and deployed
- 15:05 - Payment success rate returning to normal
- 15:10 - All metrics nominal, incident resolved
- 15:15 - Incident closed, postmortem scheduled

## Root Cause
The TLS certificate used for mutual TLS (mTLS) authentication with
the payment gateway provider expired on 2025-03-15 at 14:20 UTC.
The certificate had a 1-year validity period and was manually
provisioned. No automated renewal or expiration alerting was in place.

## What Went Well
- Alert fired within 3 minutes of the first failures
- On-call responded within 2 minutes of the alert
- Root cause was identified within 13 minutes
- Team had documented the certificate renewal process

## What Went Wrong
- No automated certificate renewal was configured
- No monitoring for certificate expiration dates
- The certificate owner (previous team member) had left the company
- Certificate expiration was not tracked in any inventory

## Action Items
| Action | Owner | Priority | Deadline |
|--------|-------|----------|----------|
| Implement cert-manager for automatic TLS renewal | Bob | P0 | 2025-03-22 |
| Add certificate expiration monitoring (alert 30 days before) | Alice | P0 | 2025-03-19 |
| Create certificate inventory for all services | Charlie | P1 | 2025-03-29 |
| Add TLS certificate check to quarterly security review | Diana | P2 | 2025-04-01 |
| Document all external service integration credentials | Team | P2 | 2025-04-15 |

## Lessons Learned
1. Manual certificate management does not scale and creates single
   points of failure, especially when team members change.
2. Monitoring must cover not just service health but also the health
   of supporting infrastructure (certificates, DNS, tokens).
3. An inventory of all time-bound credentials is essential.
```

---

## Chaos Engineering Principles

Chaos engineering is the discipline of experimenting on a system to build confidence in its ability to withstand turbulent conditions in production.

### Core Principles

```
1. BUILD A HYPOTHESIS AROUND STEADY STATE
   - Define what "normal" looks like (metrics, behavior)
   - Example: "Under normal conditions, our API responds to 99.9%
     of requests in under 200ms"

2. VARY REAL-WORLD EVENTS
   - Simulate failures that actually happen:
     - Server crashes
     - Network partitions
     - Disk full
     - CPU spikes
     - Dependency failures
     - DNS issues

3. RUN EXPERIMENTS IN PRODUCTION
   - Start in staging, but production is where it matters
   - Production has real traffic patterns and data
   - Staged environments miss edge cases

4. AUTOMATE TO RUN CONTINUOUSLY
   - One-off experiments provide limited value
   - Schedule regular experiments to catch regressions
   - Integrate with CI/CD pipelines

5. MINIMIZE BLAST RADIUS
   - Start small (one pod, one node)
   - Use abort conditions to stop if things go wrong
   - Have a rollback plan before starting
   - Gradually increase scope as confidence grows
```

### Common Chaos Experiments

```
Experiment                | What It Tests                    | Expected Outcome
--------------------------|----------------------------------|----------------------------------
Kill random pod           | Pod self-healing                 | K8s restarts pod, no user impact
Network latency injection | Timeout handling                 | Circuit breaker activates
CPU stress on node        | Autoscaling and scheduling       | HPA scales up, pods rescheduled
Kill a node               | Pod distribution and recovery    | Pods rescheduled to other nodes
DNS failure               | DNS caching and fallback         | Cached responses used, graceful degradation
Disk fill                 | Disk usage alerts and cleanup    | Alert fires, log rotation triggers
Database failover         | DB high-availability setup       | Replica promotes, app reconnects
Dependency outage         | Circuit breaker and fallback     | Fallback response served to users
```

---

## Litmus ChaosCenter

Litmus is an open-source chaos engineering platform for Kubernetes. ChaosCenter provides a web UI for managing and running chaos experiments.

### Install Litmus ChaosCenter

```bash
# Prerequisites: A running Kubernetes cluster with kubectl configured

# Method 1: Install using kubectl (recommended for learning)
kubectl apply -f https://litmuschaos.github.io/litmus/3.0.0/litmus-3.0.0.yaml

# This creates the litmus namespace with ChaosCenter components
kubectl get pods -n litmus -w
# NAME                                       READY   STATUS    RESTARTS   AGE
# litmusportal-auth-server-xxx               1/1     Running   0          2m
# litmusportal-frontend-xxx                  1/1     Running   0          2m
# litmusportal-server-xxx                    1/1     Running   0          2m
# mongodb-xxx                                1/1     Running   0          2m

# Method 2: Install using Helm
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update
helm install litmus litmuschaos/litmus \
  --namespace litmus \
  --create-namespace

# Access the ChaosCenter dashboard
kubectl port-forward svc/litmusportal-frontend-service -n litmus 9091:9091 &
# Open: http://localhost:9091
# Default credentials: admin / litmus

# Verify ChaosCenter is running
kubectl get svc -n litmus
```

### Install Litmus ChaosHub (Experiment Library)

```bash
# ChaosHub contains pre-built experiment definitions
# It is available by default in ChaosCenter

# You can also install experiment CRDs directly for CLI usage:
kubectl apply -f https://hub.litmuschaos.io/api/chaos/3.0.0?file=charts/generic/experiments.yaml -n litmus

# List available chaos experiments
kubectl get chaosexperiments -n litmus
```

### Pod Delete Experiment

```yaml
# pod-delete-experiment.yaml
# This experiment randomly kills pods to test self-healing
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-delete-chaos
  namespace: default
spec:
  # Target application
  appinfo:
    appns: default                    # Namespace of target app
    applabel: app=my-web-app          # Label selector for target pods
    appkind: deployment               # Resource type

  # Chaos will run, not just be defined
  engineState: active

  # Run as a one-time experiment
  chaosServiceAccount: litmus-admin

  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            # How many pods to kill
            - name: TOTAL_CHAOS_DURATION
              value: "30"             # Duration in seconds
            - name: CHAOS_INTERVAL
              value: "10"             # Kill a pod every 10 seconds
            - name: FORCE
              value: "false"          # Graceful termination
            - name: PODS_AFFECTED_PERC
              value: "50"             # Kill 50% of matching pods
        probe:
          # Verify the application is still accessible during chaos
          - name: check-app-health
            type: httpProbe
            httpProbe/inputs:
              url: http://my-web-app.default.svc:80/health
              method:
                get:
                  criteria: ==
                  responseCode: "200"
            mode: Continuous
            runProperties:
              probeTimeout: 5s
              retry: 3
              interval: 5s
```

```bash
# Apply the chaos experiment
kubectl apply -f pod-delete-experiment.yaml

# Watch the experiment progress
kubectl get chaosengine pod-delete-chaos -n default -w

# Check experiment results
kubectl get chaosresult pod-delete-chaos-pod-delete -n default -o yaml

# View the chaos runner logs
kubectl logs -f -l app.kubernetes.io/component=experiment -n default
```

### Network Chaos Experiment (Latency Injection)

```yaml
# network-latency-experiment.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: network-chaos
  namespace: default
spec:
  appinfo:
    appns: default
    applabel: app=my-web-app
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: NETWORK_INTERFACE
              value: "eth0"
            - name: NETWORK_LATENCY
              value: "2000"           # Add 2000ms (2s) of latency
            - name: JITTER
              value: "500"            # +/- 500ms jitter
            - name: CONTAINER_RUNTIME
              value: "containerd"
            - name: SOCKET_PATH
              value: "/run/containerd/containerd.sock"
```

### Node CPU Stress Experiment

```yaml
# node-cpu-stress.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: node-cpu-chaos
  namespace: default
spec:
  appinfo:
    appns: default
    applabel: app=my-web-app
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: node-cpu-hog
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: NODE_CPU_CORE
              value: "2"              # Stress 2 CPU cores
            - name: CPU_LOAD
              value: "80"             # 80% CPU load
          nodeSelector:
            kubernetes.io/hostname: worker-node-1
```

---

## Chaos Monkey

Chaos Monkey was created by Netflix as part of the "Simian Army." It randomly terminates virtual machine instances in production to ensure services can tolerate instance failures.

### Chaos Monkey Concepts

```
The Simian Army (Netflix):
  Chaos Monkey      - Kills random instances
  Latency Monkey    - Adds artificial latency
  Conformity Monkey - Finds non-conforming instances
  Security Monkey   - Finds security violations
  Chaos Gorilla     - Kills entire availability zones
  Chaos Kong        - Kills entire regions

Key Ideas:
  - Run in production (where it matters)
  - Random but controlled (specific services, specific times)
  - Opt-in per service (teams decide their services are ready)
  - Business hours only (humans are available to respond)
```

### Kube-Monkey (Chaos Monkey for Kubernetes)

```bash
# Install kube-monkey via Helm
helm repo add kubemonkey https://asobti.github.io/kube-monkey/charts/repo
helm repo update

helm install kube-monkey kubemonkey/kube-monkey \
  --namespace kube-monkey \
  --create-namespace \
  --set config.dryRun=true    # Start with dry run to test
```

```yaml
# kube-monkey-config.yaml (Helm values)
config:
  dryRun: false               # Set to false for actual chaos
  runHour: 8                  # Start killing at 8 AM
  startHour: 8                # Earliest kill time
  endHour: 16                 # Latest kill time (4 PM)
  gracePeriod: -1             # Use pod's terminationGracePeriodSeconds
  timeZone: America/New_York
  
  whitelistedNamespaces:
    - default
    - my-app
  # Only kill pods in these namespaces

# To opt-in a deployment to kube-monkey, add these labels:
# deployment-with-chaos.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    kube-monkey/enabled: enabled       # Opt in to chaos
    kube-monkey/identifier: my-app     # Unique identifier
    kube-monkey/mtbf: "2"              # Mean time between failures (days)
    kube-monkey/kill-mode: fixed       # fixed, random-max-percent
    kube-monkey/kill-value: "1"        # Kill 1 pod at a time
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0.0
```

---

## Load Testing with k6

k6 is an open-source load testing tool that helps you catch performance issues and SLO violations before they reach production.

### Install k6

```bash
# macOS
brew install k6

# Ubuntu/Debian
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D68
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# Docker
docker run --rm -i grafana/k6 run -
```

### Basic Load Test Script

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const latencyTrend = new Trend('request_latency');

// Test configuration
export const options = {
  // Staged ramp-up: gradually increase load
  stages: [
    { duration: '1m', target: 10 },    // Ramp up to 10 users over 1 min
    { duration: '3m', target: 10 },    // Stay at 10 users for 3 min
    { duration: '1m', target: 50 },    // Ramp up to 50 users
    { duration: '3m', target: 50 },    // Stay at 50 users for 3 min
    { duration: '1m', target: 100 },   // Ramp up to 100 users
    { duration: '5m', target: 100 },   // Stay at 100 users for 5 min
    { duration: '2m', target: 0 },     // Ramp down to 0
  ],

  // SLO-based thresholds: test FAILS if these are violated
  thresholds: {
    http_req_duration: [
      'p(95)<500',           // 95% of requests must be under 500ms
      'p(99)<1000',          // 99% of requests must be under 1000ms
    ],
    http_req_failed: ['rate<0.01'],  // Less than 1% failure rate
    errors: ['rate<0.05'],           // Custom error rate under 5%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export default function () {
  // Test the health endpoint
  const healthRes = http.get(`${BASE_URL}/health`);
  check(healthRes, {
    'health check status is 200': (r) => r.status === 200,
    'health check response time < 100ms': (r) => r.timings.duration < 100,
  });

  // Test the main API endpoint
  const apiRes = http.get(`${BASE_URL}/api/products`);
  const apiSuccess = check(apiRes, {
    'API status is 200': (r) => r.status === 200,
    'API response time < 500ms': (r) => r.timings.duration < 500,
    'API returns valid JSON': (r) => {
      try {
        JSON.parse(r.body);
        return true;
      } catch (e) {
        return false;
      }
    },
  });

  // Track custom metrics
  errorRate.add(!apiSuccess);
  latencyTrend.add(apiRes.timings.duration);

  // Test a POST endpoint
  const payload = JSON.stringify({
    name: 'Test Product',
    price: 29.99,
  });
  const params = {
    headers: { 'Content-Type': 'application/json' },
  };
  const postRes = http.post(`${BASE_URL}/api/products`, payload, params);
  check(postRes, {
    'POST status is 201 or 200': (r) => r.status === 201 || r.status === 200,
  });

  // Simulate user think time (1-3 seconds between requests)
  sleep(Math.random() * 2 + 1);
}
```

### Run k6 Load Tests

```bash
# Run a simple load test
k6 run load-test.js

# Run with custom base URL
k6 run -e BASE_URL=http://my-app.example.com load-test.js

# Run with a quick smoke test (override stages)
k6 run --vus 5 --duration 30s load-test.js

# Run and output results to JSON
k6 run --out json=results.json load-test.js

# Run and send results to Prometheus (via remote write)
k6 run --out experimental-prometheus-rw load-test.js

# Example output:
#          /\      |------| k6 v0.49.0
#     /\  /  \     |      |
#    /  \/    \    |      | https://k6.io
#   /          \   |      |
#  / __________ \  |------|
#
#   execution: local
#   scenarios: default
#
#  running (16m00s), 000/100 VUs, 28543 complete iterations
#
#      checks.........................: 98.23% 112340 out of 114378
#      data_received..................: 245 MB
#      http_req_duration..............: avg=123.4ms p(95)=342ms p(99)=678ms
#      http_req_failed................: 0.42%
#      errors.........................: 1.77%
#      iterations.....................: 28543
#
#  THRESHOLDS:
#    http_req_duration:
#      p(95)<500......................: PASSED (342ms)
#      p(99)<1000.....................: PASSED (678ms)
#    http_req_failed:
#      rate<0.01......................: PASSED (0.42%)
#    errors:
#      rate<0.05......................: PASSED (1.77%)
```

### Spike Test (Sudden Traffic Surge)

```javascript
// spike-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 10 },    // Normal load
    { duration: '30s', target: 500 },   // SPIKE to 500 users
    { duration: '2m', target: 500 },    // Sustain spike
    { duration: '30s', target: 10 },    // Back to normal
    { duration: '2m', target: 10 },     // Recovery period
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],   // Relaxed threshold during spike
    http_req_failed: ['rate<0.10'],      // Allow up to 10% failure during spike
  },
};

export default function () {
  const res = http.get('http://my-app.example.com/api/products');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
  sleep(1);
}
```

### Soak Test (Extended Duration)

```javascript
// soak-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '5m', target: 50 },     // Ramp up
    { duration: '4h', target: 50 },      // Stay at 50 users for 4 hours
    { duration: '5m', target: 0 },       // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

// Soak tests help find:
// - Memory leaks (gradual increase in memory usage)
// - Connection leaks (running out of DB connections)
// - Disk space issues (logs filling up)
// - Resource exhaustion over time

export default function () {
  const res = http.get('http://my-app.example.com/api/products');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'duration < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(2);
}
```

### k6 in CI/CD Pipeline

```yaml
# .github/workflows/load-test.yaml
name: Load Test

on:
  pull_request:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: |
          docker compose up -d
          sleep 10  # Wait for app to be ready

      - name: Install k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring \
            --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
            --keyserver hkp://keyserver.ubuntu.com:80 \
            --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D68
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
            sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update && sudo apt-get install k6

      - name: Run smoke test
        run: k6 run --vus 10 --duration 30s tests/load-test.js

      - name: Run load test
        run: k6 run tests/load-test.js
        env:
          BASE_URL: http://localhost:8080

      - name: Stop application
        if: always()
        run: docker compose down
```

---

## Projects

### Project 1: Define SLOs for a Web Application

```
Goal: Create a complete SLO document for a web application, set up
      monitoring to track SLIs, and configure alerting based on error budgets.

Steps:
  1. Choose a sample application (deploy a simple web app on Kubernetes)

  2. Define three SLOs:
     - Availability: 99.9% of requests return non-5xx responses
     - Latency: 95% of requests complete in under 300ms
     - Throughput: Sustain at least 100 requests/second

  3. Instrument the application with Prometheus metrics:
     - http_requests_total (counter with status label)
     - http_request_duration_seconds (histogram)

  4. Create a Grafana dashboard that shows:
     - Current availability percentage (last 30 days)
     - Error budget remaining (percentage and minutes)
     - P50, P95, P99 latency over time
     - Request rate (requests/second)

  5. Create Prometheus alerting rules:
     - Alert when error budget burn rate exceeds 14.4x (fast burn)
     - Alert when P99 latency exceeds 500ms for 5 minutes

  6. Write an error budget policy document defining what happens at
     each threshold (50%, 25%, 10%, 0% budget remaining)

Deliverables:
  - SLO definition document (YAML or Markdown)
  - Grafana dashboard JSON export
  - Prometheus alerting rules YAML
  - Error budget policy document
```

### Project 2: Create an Incident Runbook

```
Goal: Build a comprehensive incident runbook for a common failure
      scenario and practice the incident response process.

Steps:
  1. Deploy a sample application with known failure modes:
     - A web app that depends on a database
     - An API that depends on an external service

  2. Write runbooks for three failure scenarios:
     a. Database connection failure
     b. External API dependency timeout
     c. Memory leak causing OOM kills

  3. Each runbook must include:
     - Symptoms (what alerts fire, what users see)
     - Diagnosis steps (specific commands to run)
     - Mitigation steps (ordered from fastest to most thorough)
     - Rollback procedures
     - Root cause investigation checklist
     - Prevention recommendations

  4. Practice an incident:
     - Have a teammate inject a failure
     - Follow the runbook to diagnose and resolve
     - Write a blameless postmortem using the template above
     - Identify action items to prevent recurrence

Deliverables:
  - Three runbook documents
  - Blameless postmortem from the practice incident
  - Improvements made to runbooks after the practice
```

### Project 3: Run Chaos Experiments

```
Goal: Install a chaos engineering tool and run experiments to validate
      your application's resilience.

Steps:
  1. Deploy a multi-replica application on Kubernetes:
     kubectl create deployment my-app --image=nginx --replicas=3

  2. Install Litmus ChaosCenter:
     kubectl apply -f https://litmuschaos.github.io/litmus/3.0.0/litmus-3.0.0.yaml

  3. Access ChaosCenter at http://localhost:9091

  4. Run three chaos experiments:

     Experiment 1: Pod Delete
     - Kill 1 of 3 pods
     - Hypothesis: "Service remains available during pod restart"
     - Verify: curl the service continuously, no errors expected
     - Record: time to recover, any failed requests

     Experiment 2: Network Latency
     - Add 2s latency to pod network
     - Hypothesis: "Client-side timeouts handle latency gracefully"
     - Verify: Application returns degraded response, not error
     - Record: response times, error rates

     Experiment 3: Node CPU Stress
     - Stress 80% CPU on a node
     - Hypothesis: "HPA scales pods when CPU is high"
     - Verify: HPA triggers, pods scheduled on other nodes
     - Record: scaling time, performance during stress

  5. Run a k6 load test during each experiment to measure impact:
     k6 run --vus 20 --duration 2m load-test.js

  6. Document results:
     - Did the hypothesis hold?
     - What was the actual impact?
     - What improvements are needed?

Deliverables:
  - Chaos experiment YAML files
  - k6 load test results (before, during, after chaos)
  - Summary report with findings and recommended improvements
  - At least one fix implemented based on findings
```

---

## Summary

| Concept         | Purpose                                          |
|----------------|--------------------------------------------------|
| SLIs           | Measure what matters to users                    |
| SLOs           | Set reliability targets                          |
| SLAs           | Contractual promises with consequences           |
| Error Budgets  | Balance reliability with feature velocity        |
| Incident Mgmt  | Structured response to service disruptions       |
| Postmortems    | Learn from failures without blame                |
| Chaos Eng      | Proactively find weaknesses before users do      |
| Load Testing   | Verify performance under expected and peak loads |

**Key Takeaways:**
- SRE is about applying engineering to operations, not just keeping things running
- Error budgets give teams a data-driven way to balance reliability and speed
- Blameless postmortems focus on systemic improvements, not individual mistakes
- Chaos engineering builds confidence by testing failure modes before they happen in the wild
- Load testing with k6 validates SLOs and catches performance regressions early
