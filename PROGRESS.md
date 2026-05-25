# PROGRESS.md — CockroachDB Support Engineering Practice

---

## OVERALL PROGRESS

**Completed: 7 / 15**
**Average Score: 8.4**

---

## SCENARIO TRACKER

| ID | Scenario | Level | Score | Date |
|---|---|---|---|---|
| BF01 | Node Fails to Start — Bad Data Directory | 0 | 8/10 | 2026-05-24 |
| BF02 | Node Fails to Start — File Descriptor Limit | 0 | 9/10 | 2026-05-24 |
| BF03 | Cluster Won't Form — Wrong Join Flags | 0 | 7/10 | 2026-05-24 |
| BF04 | Client Cannot Connect — Connection Refused | 1 | 8/10 | 2026-05-25 |
| BF05 | Wrong Connection String / Wrong Port | 1 | 8/10 | 2026-05-25 |
| BF06 | Client Cannot Connect — TLS Certificate Error | 1 | 9/10 | 2026-05-25 |
| BF07 | Transaction Retry Errors — No Retry Logic | 2 | 9/10 | 2026-05-25 |
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
| Triage Approach | 7 | 2.0 |
| Tools Used | 7 | 2.0 |
| Documentation | 7 | 1.7 |
| Root Cause Accuracy | 7 | 1.6 |
| Runbook Quality | 7 | 1.6 |
| **Overall** | **7** | **8.4** |

**Weakest dimension so far:** Runbook Quality — the recurring gap has shifted from format (BF01–BF03) to content depth; runbooks now have the right structure but for non-standard scenarios (BF07) the diagnostic commands and code examples are missing

---

## PATTERNS AND OBSERVATIONS

*Updated after every session. This section tells you where your consistent gaps are — not what you did, but what you need to fix.*

**BF01 (2026-05-24):** Documentation was the gap — broad disaster recovery page cited instead of the specific "Decommission a node" doc. Runbook had the right steps but omitted the exact `docker run` command and had no final verification step. Triage and root cause were strong.

**BF02 (2026-05-24):** Strong session — used `docker inspect` to confirm config, not just the error message. Documentation exact. Runbook is the recurring gap: fix step mentioned the `--ulimit` flag but omitted the full `docker run` command; verification step had no command. Same pattern as BF01.

**BF03 (2026-05-24):** Best documentation navigation yet — found exact section independently. Triage was clean. Two gaps: `docker network inspect` used wrong network name (`crdb-test` vs `crdb-net`), and the internal CRDB log was not checked (would have shown exact port being dialed). Root cause identified the fact (port mismatch) but not the mechanism. Runbook improved — full fix command present — but still no explicit verification step.

**BF04 (2026-05-25):** Strong triage and tools — CockroachDB internal logs used to confirm IP-level evidence (DNS resolved, packet arrived, port rejected), docker ps confirmed missing port mapping. First session with a 2/2 runbook: full fix command, specific commands throughout, four explicit verification steps at the end. Recurring gaps: documentation cited as a footnote with no evidence it was consulted; root cause named the missing port mapping correctly but didn't explain the Docker NAT/iptables mechanism behind it.

**BF05 (2026-05-25):** Same pattern as BF04 — systematic triage, correct tools, runbook 2/2. Improvement: two docs cited this time, with connection-parameters.html being the more specific page. Recurring gaps unchanged: docs cited but not applied; root cause correctly identified the port mismatch and articulated the mapping rule clearly, but didn't explain that Docker port bindings are set at container creation time and are lost when a container is stopped and recreated.

**BF06 (2026-05-25):** Best session to date — 9/10. First 2/2 on documentation: transport-layer-security.html found and applied (all nodes must share the same security mode). Tools were precise: grepping internal logs for `insecure|secure|certs` across all three nodes. Runbook 2/2 again. Root cause remains the one gap — secure/insecure mismatch correctly identified, but the mTLS inter-node gRPC mechanism (why the handshake fails at the protocol level) was not explained.

**BF07 (2026-05-25):** 9/10 — first 2/2 on root cause: OCC explained correctly, distributed cost of pessimistic locking articulated, PostgreSQL contrast drawn accurately. Triage and tools also 2/2 (two concurrent sessions on separate nodes, crdb_internal.cluster_locks). Documentation 2/2. Runbook is the one gap: reads as a recommendation document rather than numbered steps — missing reproduction commands, crdb_internal.cluster_locks query, and a concrete retry loop code snippet.

---

## NEXT SESSION

**Break-fix:** `start break-fix BF08`
