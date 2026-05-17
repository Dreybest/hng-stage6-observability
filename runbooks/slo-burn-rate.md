# Runbook: SLOFastBurn / SLOSlowBurn

## What is this alert?

Your error budget is being consumed faster than expected.
The error budget is your monthly allowance of failure.
With a 99.5% SLO target, you are allowed 0.5% failure per month —
that is approximately 216 minutes of downtime per 30 days.

| Alert | Condition | Meaning |
|---|---|---|
| SLOFastBurn | 2% budget consumed in 1 hour | At this rate budget exhausted in ~2 days |
| SLOSlowBurn | 5% budget consumed in 6 hours | At this rate budget exhausted in ~5 days |

**Dashboard:** [SLO & Error Budget](http://ec2-54-226-46-2.compute-1.amazonaws.com:3000/d/slo-error-budget)  
**Runbook:** This file

---

## Likely causes
- The server or application is partially down — some probes failing
- High latency causing probe timeouts
- A bad deployment introduced errors affecting availability
- Network instability between monitoring server and app server
- SSL certificate expired causing HTTPS probes to fail

---

## First 3 investigation steps

### Step 1 — Check the SLO dashboard immediately
Open the SLO & Error Budget dashboard:
http://ec2-54-226-46-2.compute-1.amazonaws.com:3000/d/slo-error-budget
Look at:
- Current availability — is it below 99.5%?
- Burn rate graph — is it fast burn or slow burn?
- How much budget is remaining?

### Step 2 — Check the Blackbox Exporter dashboard
http://ec2-54-226-46-2.compute-1.amazonaws.com:3000/d/blackbox
Look at:
- Which probe is failing — is it one endpoint or all?
- When did probes start failing?
- Is response time spiking above 500ms?

### Step 3 — Correlate with logs and traces
Open the Unified Observability dashboard:
http://ec2-54-226-46-2.compute-1.amazonaws.com:3000/d/unified-observability
- Find error logs from the time the burn rate started
- Click any trace_id in the logs to open in Tempo
- Identify which service or endpoint is causing failures

---

## How to resolve it

**If probes are failing due to server being down:**
- Follow the ServerDown runbook immediately

**If probes are timing out due to high latency:**
```bash
# Check what is causing latency on the app server
ssh -i your-key.pem ubuntu@YOUR_APP_SERVER_IP
docker stats --no-stream
top -b -n 1 | head -20
```

**If a bad deployment is causing errors:**
```bash
# Roll back to the last working deployment
# Trigger the previous successful GitHub Actions workflow
# Or manually on the app server:
docker compose down
docker compose up -d previous-working-version
```

**If SSL certificate expired:**
```bash
# Check SSL expiry on the Blackbox dashboard
# Renew the certificate
sudo certbot renew
sudo systemctl restart nginx
```

---

## Error budget policy actions

| Budget remaining | Action required |
|---|---|
| 100–50% remaining | Normal operations. Monitor the burn rate. |
| 50–25% remaining | Slow down feature releases. Focus on stability. |
| 25–0% remaining | Stop all feature work. Full focus on reliability. |
| 0% — budget exhausted | Feature freeze. Reliability sprint begins. Notify team lead. |

---

## When to roll back
Roll back immediately if:
- Burn rate started within 30 minutes of a deployment
- Logs show errors that correlate with the new deployment
- Fast Burn alert is firing and budget is below 25%

---

## Escalation
| Alert | Wait time | Action |
|---|---|---|
| SLOSlowBurn | 30 minutes | Notify team lead if not resolved |
| SLOFastBurn | Immediately | Notify team lead — this is urgent |
| Budget exhausted | Immediately | Feature freeze + escalate to HNG admin |