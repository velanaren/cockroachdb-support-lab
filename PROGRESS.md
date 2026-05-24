# PROGRESS.md — CockroachDB Support Engineering Practice

---

## OVERALL PROGRESS

**Completed: 1 / 15**
**Average Score: 8.0**

---

## SCENARIO TRACKER

| ID | Scenario | Level | Score | Date |
|---|---|---|---|---|
| BF01 | Node Fails to Start — Bad Data Directory | 0 | 8/10 | 2026-05-24 |
| BF02 | Node Fails to Start — File Descriptor Limit | 0 | — | — |
| BF03 | Cluster Won't Form — Wrong Join Flags | 0 | — | — |
| BF04 | Client Cannot Connect — Connection Refused | 1 | — | — |
| BF05 | Wrong Connection String / Wrong Port | 1 | — | — |
| BF06 | Client Cannot Connect — TLS Certificate Error | 1 | — | — |
| BF07 | Transaction Retry Errors — No Retry Logic | 2 | — | — |
| BF08 | Slow Queries — Missing Index | 3 | — | — |
| BF09 | Transaction Contention Spike | 3 | — | — |
| BF10 | Hotspot on Sequential Insert | 3 | — | — |
| BF11 | Connection Pool Exhaustion | 3 | — | — |
| BF12 | Node OOM Restart | 4 | — | — |
| BF13 | CPU Starvation — Node Marked SUSPECT | 4 | — | — |
| BF14 | Clock Skew Between Nodes | 4 | — | — |
| BF15 | Backup Failing — GC TTL Mismatch | 3 | — | — |

---

## SCORE SUMMARY

| Dimension | Sessions Scored | Running Average |
|---|---|---|
| Triage Approach | 1 | 2.0 |
| Tools Used | 1 | 2.0 |
| Documentation | 1 | 1.0 |
| Root Cause Accuracy | 1 | 2.0 |
| Runbook Quality | 1 | 1.0 |
| **Overall** | **1** | **8.0** |

**Weakest dimension so far:** Documentation and Runbook Quality (1/2 each)

---

## PATTERNS AND OBSERVATIONS

*Updated after every session. This section tells you where your consistent gaps are — not what you did, but what you need to fix.*

**BF01 (2026-05-24):** Documentation was the gap — broad disaster recovery page cited instead of the specific "Decommission a node" doc. Runbook had the right steps but omitted the exact `docker run` command and had no final verification step. Triage and root cause were strong.

---

## NEXT SESSION

**Break-fix:** `start break-fix BF02`
