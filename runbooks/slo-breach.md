# Runbook: SLO Breach — General Guide

## What is this document?
This runbook covers what to do when an SLO has been fully breached —
meaning the error budget has been completely exhausted for the month.
This is different from the burn rate runbooks which fire as warnings
before the budget runs out.

| SLO | Target | Window |
|---|---|---|
| Availability | 99.5% | 30 days rolling |
| Latency | 95% of requests under 500ms | 30 days rolling |
| Error Rate | 99% of requests succeed | 30 days rolling |

**Dashboard:** [SLO & Error Budget](http://ec2-3-84-114-138.compute-1.amazonaws.com:3000/d/slo-error-budget)  
**Runbook:** This file

---

## Likely causes
- Multiple incidents in the same month exhausted the budget
- A single long outage used up the entire month's budget
- Burn rate alerts were ignored or missed
- The SLO target was set too aggressively for the current infrastructure
- Underlying reliability issues were not addressed after previous incidents

---

## First 3 investigation steps

### Step 1 — Confirm the breach on the dashboard
Open the SLO & Error Budget dashboard:
http://ec2-3-84-114-138.compute-1.amazonaws.com:3000/d/slo-error-budget
Confirm:
- Error Budget Remaining shows 0% or negative
- Which SLO was breached — availability, latency, or error rate?
- When did the budget run out?

### Step 2 — Review the incident timeline
```bash
# Check Prometheus for the availability history
# Query in Grafana:
avg_over_time(probe_success{job="blackbox-exporter"}[30d])

# Check when availability dropped below 99.5%
# Look at the Availability vs SLO Target panel
# Identify every dip below the target line
```

### Step 3 — Identify all contributing incidents
Go to the Unified Observability dashboard and review:
- Every period where metrics dropped below SLO targets
- Corresponding logs from those periods
- Any traces showing errors during those windows
- GitHub Actions history for deployments that coincided with drops

---

## Immediate actions on breach

### Step 1 — Declare feature freeze
All new feature development stops immediately.
No new deployments to production until SLO is restored.
Notify the entire team in Slack.

### Step 2 — Notify stakeholders
Post in #DevOps-Alerts:
🚨 SLO BREACH — Error budget exhausted for this month.
Availability SLO: 99.5% target — current: [VALUE]%
Feature freeze is now in effect.
Reliability sprint begins immediately.
Expected recovery: [DATE — start of next month or sooner]
### Step 3 — Begin reliability sprint
Focus 100% of engineering effort on:
- Fixing the root cause of the largest incidents this month
- Adding better alerting to catch issues earlier
- Improving deployment safety — better tests, staged rollouts
- Reviewing and updating runbooks based on what was learned

---

## Error budget policy — full reference

| Budget consumed | Status | Action |
|---|---|---|
| 0–50% | Healthy | Normal operations, ship features freely |
| 50–75% | Caution | Monitor closely, reduce deployment frequency |
| 75–100% | Warning | Stop non-critical deployments, focus on stability |
| 100% breached | Critical | Feature freeze, reliability sprint, stakeholder notification |

---

## SLO review process
SLOs are reviewed at the end of every month:

1. Was the SLO met this month?
2. If yes — is the target still meaningful or should it be raised?
3. If no — was the target too aggressive or was there a real reliability problem?
4. Update SLO targets in `rules/alerts.yml` and `prometheus.yml` if needed
5. Document the decision and reasoning in the repo

---

## Post-incident requirement
After any SLO breach a blameless Post-Incident Review must be written
and committed to the repository within 48 hours covering:
- Full incident timeline
- Root cause
- User impact
- What went wrong in detection or response
- Action items with owners and due dates

---

## When to roll back
If the breach was caused by a recent deployment:
```bash
# Revert the deployment immediately
git revert HEAD
git push origin main

# On the app server
docker compose down
docker compose up -d last-stable-image
```

---

## Escalation
| Situation | Action |
|---|---|
| Budget at 0% | Immediate feature freeze + notify team lead |
| Breach lasting more than 24 hours | Escalate to HNG infrastructure admin |
| Same SLO breached two months in a row | Review SLO targets with entire team |
| Unable to restore reliability | Request infrastructure upgrade from HNG admin |