# Runbook: CPUHighWarning / CPUHighCritical

## What is this alert?
CPU usage on the server has exceeded the threshold for a sustained period.

| Alert | Threshold | Duration |
|---|---|---|
| CPUHighWarning | Above 80% | 5+ minutes |
| CPUHighCritical | Above 90% | 10+ minutes |

**Dashboard:** [Node Exporter](http://ec2-54-226-46-2.compute-1.amazonaws.com:3000/d/node-exporter)  
**Runbook:** This file  

---

## Likely causes
- A runaway process consuming all CPU
- A recent deployment introduced an infinite loop or inefficient code
- Unusually high traffic spike
- A background job (backup, cron) running at the wrong time
- The server is undersized for the current workload

---

## First 3 investigation steps

### Step 1 — Find what is using the CPU
SSH into the server and run:
```bash
top -b -n 1 | head -20
```
Look for any process using a high CPU %. Note the process name and PID.

### Step 2 — Check if it started after a deployment
```bash
# Check recent Docker container CPU usage
docker stats --no-stream

# Check system logs for anything unusual
journalctl -n 100 --since "30 minutes ago"
```

### Step 3 — Check the load average trend
```bash
uptime
# Shows load average for 1m, 5m, 15m
# If 1m is high but 15m is normal — it is a spike, wait and watch
# If all three are high — it is sustained, take action
```

---

## How to resolve it

**If a specific container is causing it:**
```bash
# Restart the offending container
docker compose restart service-name

# If it keeps spiking, stop it temporarily
docker compose stop service-name
```

**If it is a cron job or background task:**
```bash
# List running cron jobs
crontab -l
sudo crontab -l

# Kill a specific runaway process by PID
sudo kill -15 PID_NUMBER
```

**If traffic spike:**
- Check Blackbox dashboard for unusual probe patterns
- Monitor — if CPU drops back below 80% within 15 minutes it was a temporary spike
- No action needed if it resolves itself

---

## When to roll back
Roll back if:
- CPU spike started within 30 minutes of a deployment
- Restarting the container does not bring CPU down
- Logs show errors related to the new deployment

---

## Escalation
| Severity | Wait time | Action |
|---|---|---|
| Warning | 15 minutes | Notify team lead if not resolved |
| Critical | 5 minutes | Notify team lead immediately |
| Critical sustained | 15 minutes | Escalate to HNG infrastructure admin |