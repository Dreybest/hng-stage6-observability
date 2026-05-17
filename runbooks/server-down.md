# Runbook: ServerDown

## What is this alert?
The Blackbox Exporter has failed to reach the server for more than 2 consecutive 
minutes. The server is completely unreachable from the monitoring stack.

**Severity:** Critical  
**Alert name:** ServerDown  
**Dashboard:** [Blackbox Exporter](http://ec2-3-84-114-138.compute-1.amazonaws.com:3000/d/blackbox)  
**Fires when:** `probe_success == 0` for 2+ minutes  

---

## Likely causes
- The EC2 instance has stopped or crashed
- The Security Group is blocking traffic on the probed port
- The application process has crashed but the server is still running
- Network issue between monitoring server and app server
- The server ran out of memory and became unresponsive

---

## First 3 investigation steps

### Step 1 — Check AWS Console
Go to AWS EC2 → Instances and check the instance state:
- If **Stopped** → start it immediately
- If **Running** but unreachable → proceed to Step 2
- If **Terminated** → notify team lead immediately

### Step 2 — Try to SSH into the server
```bash
ssh -i your-key.pem ubuntu@YOUR_APP_SERVER_IP
```
- If SSH works → the server is up but the application crashed. Check Step 3
- If SSH fails → the server itself is down or network is blocked

### Step 3 — Check if the application is running
Once SSHed in, run:
```bash
docker compose ps
sudo systemctl status your-app
journalctl -u your-app -n 50
```
Look for any crashed containers or error messages in the logs.

---

## How to resolve it

**If the server is stopped:**
```bash
# Start from AWS Console or CLI
aws ec2 start-instances --instance-ids YOUR_INSTANCE_ID
```

**If the application crashed:**
```bash
# Restart the application
docker compose restart
# Or restart a specific service
docker compose restart your-service-name
```

**If out of memory:**
```bash
# Check memory
free -h
# Kill memory-hungry processes
docker stats
docker compose restart memory-hungry-service
```

---

## When to roll back
Roll back if:
- The server came back up but the application keeps crashing after restart
- Logs show errors that started after a recent deployment
- The issue began within 30 minutes of a deployment

To roll back:
```bash
# In your GitHub Actions — re-run the last successful deployment workflow
# Or manually on the server:
docker compose down
docker compose pull previous-image-tag
docker compose up -d
```

---

## Escalation
| Time down | Action |
|---|---|
| 0–5 minutes | Investigate yourself using steps above |
| 5–15 minutes | Notify team lead on Slack |
| 15+ minutes | Escalate to HNG infrastructure admin |

**Who to contact:** Team lead → HNG infrastructure admin  
**Escalation channel:** #DevOps-Alerts (already firing there)