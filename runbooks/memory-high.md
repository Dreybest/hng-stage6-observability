# Runbook: MemoryHighWarning / MemoryHighCritical

## What is this alert?
Memory usage on the server has exceeded the threshold for a sustained period.

| Alert | Threshold | Duration |
|---|---|---|
| MemoryHighWarning | Above 80% | 5+ minutes |
| MemoryHighCritical | Above 90% | 5+ minutes |

**Dashboard:** [Node Exporter](http://ec2-54-226-46-2.compute-1.amazonaws.com:3000/d/node-exporter)  
**Runbook:** This file  

---

## Likely causes
- A memory leak in the application — memory grows over time and never gets released
- Too many containers running on an undersized server
- A recent deployment introduced code that holds too much data in memory
- Loki or Prometheus retaining too much data in memory
- The server ran out of swap space as well as RAM

---

## First 3 investigation steps

### Step 1 — Check current memory usage breakdown
SSH into the server and run:
```bash
free -h
```
This shows total, used, free, and available memory plus swap usage.
- If **Swap is also high** — situation is critical, act immediately
- If **Swap is still free** — you have a small buffer, investigate calmly

### Step 2 — Find which container is using the most memory
```bash
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}"
```
This shows every container sorted by memory. Note the biggest offender.

### Step 3 — Check if memory is growing over time
```bash
# Watch memory usage every 2 seconds
watch -n 2 free -h
```
- If memory keeps growing → memory leak, restart the offending container
- If memory is stable but high → server is undersized, consider scaling up

---

## How to resolve it

**Restart the container using the most memory:**
```bash
docker compose restart service-name
```

**If Loki is the offender** (common on small servers):
```bash
docker compose restart loki
```

**If Prometheus is the offender:**
```bash
docker compose restart prometheus
```

**Emergency — free memory immediately:**
```bash
# Drop page cache to free memory
sudo sync && sudo sysctl -w vm.drop_caches=3

# Check if it helped
free -h
```

**If swap is full and memory is critical:**
```bash
# Find and kill the biggest memory process
ps aux --sort=-%mem | head -10
sudo kill -15 PID_NUMBER
```

---

## When to roll back
Roll back if:
- Memory spike started within 30 minutes of a deployment
- Restarting the container does not bring memory down
- Memory keeps climbing back up after restart — clear sign of a memory leak in new code

---

## Escalation
| Severity | Wait time | Action |
|---|---|---|
| Warning | 15 minutes | Notify team lead if not resolved |
| Critical | 5 minutes | Notify team lead immediately |
| Critical + swap full | Immediately | Escalate to HNG infrastructure admin |