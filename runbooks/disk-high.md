# Runbook: DiskHighWarning / DiskHighCritical

## What is this alert?
Disk usage on the server has exceeded the threshold.

| Alert | Threshold | Duration |
|---|---|---|
| DiskHighWarning | Above 75% | 5+ minutes |
| DiskHighCritical | Above 90% | 5+ minutes |

**Dashboard:** [Node Exporter](http://ec2-3-84-114-138.compute-1.amazonaws.com:3000/d/node-exporter)  
**Runbook:** This file  

---

## Likely causes
- Prometheus metrics retention filling up disk over time
- Loki logs filling up disk — too many logs being ingested
- Docker images and stopped containers accumulating
- Application log files not being rotated
- Tempo traces filling up storage

---

## First 3 investigation steps

### Step 1 — Check which directory is using the most space
SSH into the server and run:
```bash