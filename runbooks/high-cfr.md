# Runbook: HighChangeFailureRate

## What is this alert?
More than 5% of deployments in the last 7 days have failed, rolled back,
or required a hotfix. This breaches the DORA Change Failure Rate SLO.

| Metric | Target | Current trigger |
|---|---|---|
| Change Failure Rate | Below 5% | Above 5% over 7 days |

**Dashboard:** [DORA Metrics](http://ec2-3-84-114-138.compute-1.amazonaws.com:3000/d/dora-metrics)  
**Runbook:** This file

---

## Likely causes
- Code is being merged and deployed without sufficient testing
- CI pipeline is not catching bugs before they reach production
- Deployments are happening too fast without proper review
- Infrastructure changes are breaking application compatibility
- Missing or insufficient staging environment before production
- Rollback procedures are not being tracked as failures

---

## First 3 investigation steps

### Step 1 — Check the DORA dashboard
Open the DORA Metrics dashboard:
http://ec2-3-84-114-138.compute-1.amazonaws.com:3000/d/dora-metrics
Look at:
- Current CFR percentage — how far above 5% is it?
- Change Failure Rate Over Time graph — when did it start rising?
- Which workflows are failing most frequently?

### Step 2 — Check GitHub Actions for failed workflows
Go to your GitHub repository:
https://github.com/Dreybest/hng-stage6-observability/actions
Look at:
- Which workflows failed recently?
- What was the error message in the failed runs?
- Did the failures start after a specific commit or pull request?

### Step 3 — Correlate failures with deployments
```bash
# On the app server, check recent container restarts
docker compose ps
docker compose logs --tail=50 your-service-name

# Check if any services are in a restart loop
docker ps --filter "status=restarting"
```

---

## How to resolve it

**If tests are not catching bugs:**
- Add more unit and integration tests to the CI pipeline
- Make tests mandatory before merge — add branch protection rules in GitHub
- Add a staging environment deployment step before production

**If deployments are too fast:**
- Implement pull request reviews — require at least one approval before merge
- Add a manual approval step in GitHub Actions before production deployment
- Slow down deployment frequency until CFR drops below 5%

**If a specific recent deployment is causing failures:**
```bash
# Roll back on the app server
docker compose down
docker compose up -d previous-working-image

# Or re-trigger the last successful GitHub Actions workflow
# Go to Actions → find last green run → Re-run jobs
```

**If infrastructure changes are causing failures:**
```bash
# Check what changed recently in your config files
git log --oneline -10
git diff HEAD~1 HEAD

# Revert the last commit if it caused the issue
git revert HEAD
git push origin main
```

---

## Toil identified
This alert often fires because of manual, repetitive processes that
should be automated. Two examples identified in this project:

**Toil 1 — Manual deployment verification**
Currently someone manually checks if a deployment succeeded.
Automation: Add a post-deployment health check step in GitHub Actions
that automatically probes the service and fails the workflow if the
service does not respond within 60 seconds.

**Toil 2 — Manual rollback process**
Currently rolling back requires SSHing into the server and running
commands manually.
Automation: Add a one-click rollback workflow in GitHub Actions that
can be triggered manually from the Actions tab without needing SSH
access.

---

## When to roll back
Roll back immediately if:
- CFR is above 15% — more than 1 in 6 deployments are failing
- The same deployment has failed more than twice
- Users are reporting errors in production

---

## Escalation
| CFR level | Wait time | Action |
|---|---|---|
| 5–10% | 24 hours | Review deployment process with team |
| 10–15% | 4 hours | Notify team lead, slow down deployments |
| Above 15% | Immediately | Stop all deployments, escalate to team lead |