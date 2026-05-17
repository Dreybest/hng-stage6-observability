# Post-Incident Review (PIR)
## Blameless Post-Incident Review — Simulated Latency Incident

**Date:** May 17, 2026  
**Severity:** Critical  
**Duration:** 47 minutes  
**Author:** Dreybest  
**Status:** Resolved  

---

## Incident summary
A latency injection test was performed as part of Game Day Scenario 2.
High latency was simulated on the application server causing HTTP probe
response times to spike above 2 seconds. This caused the availability
SLI to drop below the 99.5% SLO target, triggering the SLOFastBurn
alert. The error budget burn rate reached 18x — above the 14.4x
critical threshold. The incident was detected automatically within
2 minutes and resolved within 47 minutes.

---

## Timeline

| Time | Event |
|---|---|
| 10:00 | Latency injection applied to application server |
| 10:02 | Blackbox probe response time spikes above 2 seconds |
| 10:02 | Availability SLI drops below 99.5% threshold |
| 10:04 | SLOFastBurn alert fires in Slack #DevOps-Alerts |
| 10:05 | On-call engineer acknowledges alert in Slack |
| 10:07 | Engineer opens Unified Observability dashboard |
| 10:09 | Metric spike identified in Response Time vs SLO panel |
| 10:11 | Engineer scrolls to Live Logs panel — error logs visible |
| 10:13 | trace_id clicked in log line — Tempo opens automatically |
| 10:15 | Trace shows latency concentrated in application service layer |
| 10:18 | Root cause confirmed — latency injection on application server |
| 10:22 | Latency injection removed from application server |
| 10:25 | Response times return to normal — under 200ms |
| 10:27 | SLOFastBurn alert resolves — recovery notification fires in Slack |
| 10:47 | Full monitoring confirmed stable — incident closed |

**Detection time:** 4 minutes (from injection to alert in Slack)  
**Response time:** 1 minute (from alert to engineer acknowledging)  
**Resolution time:** 22 minutes (from root cause confirmed to fix applied)  
**Total incident duration:** 47 minutes  

---

## Root cause
Artificial latency was injected into the application server as part of
a planned Game Day chaos exercise. The latency caused HTTP probe
response times to exceed the Blackbox Exporter timeout threshold,
causing probes to be recorded as failures and degrading the
availability SLI below the 99.5% SLO target.

In a real incident this pattern would indicate one of:
- A memory leak causing the application to slow down under load
- A database query taking too long and blocking request processing
- A downstream service dependency timing out
- Insufficient server resources for the current traffic level

---

## Impact
| Area | Impact |
|---|---|
| Availability | Dropped from 100% to 94% during incident window |
| Error budget | Approximately 8% of monthly budget consumed |
| Users affected | Simulated — no real users affected |
| Services affected | Application HTTP endpoint |
| Data loss | None |

---

## What went well

- **Alerting worked correctly** — SLOFastBurn fired within 4 minutes
  of the latency injection being applied
- **Slack notification was structured and actionable** — included
  dashboard link, runbook link, severity, and current metric value
- **Drill-down worked end to end** — metric spike led to logs,
  logs led to trace, trace identified the exact service layer
- **Recovery alert fired automatically** — when latency was removed
  Slack sent a resolved notification without manual intervention
- **No manual log searching required** — Loki and Tempo correlation
  reduced investigation time significantly

---

## What went wrong

### Issue 1 — Alert took 4 minutes to fire
The `for: 2m` duration on the SLOFastBurn alert meant the alert did
not fire until 2 minutes after the condition was met. For a fast burn
scenario this delay could allow significant budget consumption.

**Action item:** Consider reducing `for` duration on SLOFastBurn
from 2 minutes to 1 minute for faster response.  
**Owner:** Dreybest  
**Due:** Before production deployment  

### Issue 2 — No automated runbook link in terminal
During investigation the engineer had to manually open the browser
to find the runbook. The Slack alert contained the runbook link
but it was not immediately obvious.

**Action item:** Add runbook URL as the first line of the Slack
alert text so it is immediately visible without scrolling.  
**Owner:** Dreybest  
**Due:** Before production deployment  

### Issue 3 — Tempo trace search required manual service name input
When opening Tempo from the Unified dashboard the service name filter
was not pre-populated, requiring manual input before traces appeared.

**Action item:** Update the Tempo panel query in
`unified-observability.json` to pre-filter by service name.  
**Owner:** Dreybest  
**Due:** Next sprint  

---

## Action items

| # | Action | Owner | Due date | Status |
|---|---|---|---|---|
| 1 | Reduce SLOFastBurn `for` from 2m to 1m | Dreybest | Before production | Open |
| 2 | Move runbook URL to first line of Slack template | Dreybest | Before production | Open |
| 3 | Pre-filter Tempo panel by service name | Dreybest | Next sprint | Open |
| 4 | Add post-deployment health check to GitHub Actions | Dreybest | Next sprint | Open |
| 5 | Document latency injection procedure for future Game Days | Dreybest | Next sprint | Open |

---

## Lessons learned

1. **Burn rate alerting works** — the multi-window approach correctly
   identified the incident as fast burn and fired the right severity
2. **Drill-down is the most valuable feature** — finding root cause
   took 9 minutes instead of the typical 30+ minutes of manual log
   searching
3. **Recovery alerts are important** — knowing automatically when
   the incident resolved removed the need for manual monitoring
   during the recovery phase
4. **Game Day exercises reveal real gaps** — the three action items
   above would not have been identified without actually running the
   scenario end to end

---

## Follow-up

This PIR will be reviewed at the next team meeting.
All action items will be tracked in GitHub Issues.
The next Game Day is scheduled after production deployment
to validate all action items have been implemented.

---

*This is a blameless review. The focus is on system and process
improvements, not individual performance.*

*HNG DevOps Track — Stage 6 | Dreybest | May 2026*