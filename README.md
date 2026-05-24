# CockroachDB Support Engineering Practice

A hands-on lab for building deep CockroachDB troubleshooting competence through real failure injection and structured diagnosis.

---

## What This Is

This project is a self-directed support engineering practice environment built around CockroachDB. The goal is simple: set up a real cluster, break it in ways that reflect actual production failures, diagnose and fix it, and leave behind a runbook that another engineer could follow.

Each scenario is drawn from real customer-reported issues — cluster setup failures, connectivity problems, SQL behavior issues, and distributed system degradation. The failures progress incrementally from the most fundamental (a node that won't start) to the most complex (a node being marked SUSPECT due to CPU starvation).

This is not a tutorial walkthrough. Every scenario starts with a symptom, not an explanation.

---

## How This Works

**Setup**
A 3-node CockroachDB cluster is run locally using Docker. This simulates a real self-hosted deployment with inter-node communication, a load balancer, and the Admin UI.

**The Workflow**
1. A failure is injected into the cluster — simulating what a customer might report
2. The engineer diagnoses the issue using CockroachDB tooling, Linux diagnostics, and SQL
3. The fix is applied and verified
4. A runbook is written capturing the diagnostic path and resolution

**Scoring**
Each scenario is scored against a rubric covering triage approach, tools used, documentation navigation, root cause accuracy, and runbook quality. Scores are tracked in `PROGRESS.md` and detailed in each scenario's `score.md`.

---

## Break-Fix Scenarios

| ID | Scenario | Level | Score |
|---|---|---|---|
| BF01 | Node Fails to Start — Bad Data Directory | 0 | — |
| BF02 | Node Fails to Start — File Descriptor Limit | 0 | — |
| BF03 | Cluster Won't Form — Wrong Join Flags | 0 | — |
| BF04 | Client Cannot Connect — Connection Refused | 1 | — |
| BF05 | Wrong Connection String / Wrong Port | 1 | — |
| BF06 | Client Cannot Connect — TLS Certificate Error | 1 | — |
| BF07 | Transaction Retry Errors — No Retry Logic | 2 | — |
| BF08 | Slow Queries — Missing Index | 3 | — |
| BF09 | Transaction Contention Spike | 3 | — |
| BF10 | Hotspot on Sequential Insert | 3 | — |
| BF11 | Connection Pool Exhaustion | 3 | — |
| BF12 | Node OOM Restart | 4 | — |
| BF13 | CPU Starvation — Node Marked SUSPECT | 4 | — |
| BF14 | Clock Skew Between Nodes | 4 | — |
| BF15 | Backup Failing — GC TTL Mismatch | 3 | — |

**Level Guide**

| Level | What it means |
|---|---|
| 0 | The node or cluster won't start — nothing is running |
| 1 | The cluster is running but the client cannot reach it |
| 2 | The client connects but operations fail |
| 3 | Operations run but results are wrong, slow, or failing under load |
| 4 | The cluster is running but degrading at the distributed systems layer |

---

## What Each Scenario Produces

Every scenario folder contains two files:

**`transcript.md`** — The full diagnostic session. What the failure looked like, what was checked, what was found, how it was fixed, and the runbook left behind for the next engineer.

**`score.md`** — The score out of 10 with rationale across five dimensions: triage approach, tools used, documentation navigation, root cause accuracy, and runbook quality.

---

## Repository Structure

```
cockroachdb-support-lab/
├── README.md
├── CLAUDE.md
├── PROGRESS.md
└── break-fix/
    ├── BF01-node-fails-to-start-bad-data-directory/
    │   ├── transcript.md
    │   └── score.md
    ├── BF02-node-fails-to-start-file-descriptor-limit/
    │   ├── transcript.md
    │   └── score.md
    ├── BF03-cluster-wont-form-wrong-join-flags/
    │   ├── transcript.md
    │   └── score.md
    ├── BF04-client-cannot-connect-connection-refused/
    │   ├── transcript.md
    │   └── score.md
    ├── BF05-wrong-connection-string-wrong-port/
    │   ├── transcript.md
    │   └── score.md
    ├── BF06-client-cannot-connect-tls-certificate-error/
    │   ├── transcript.md
    │   └── score.md
    ├── BF07-transaction-retry-errors-no-retry-logic/
    │   ├── transcript.md
    │   └── score.md
    ├── BF08-slow-queries-missing-index/
    │   ├── transcript.md
    │   └── score.md
    ├── BF09-transaction-contention-spike/
    │   ├── transcript.md
    │   └── score.md
    ├── BF10-hotspot-on-sequential-insert/
    │   ├── transcript.md
    │   └── score.md
    ├── BF11-connection-pool-exhaustion/
    │   ├── transcript.md
    │   └── score.md
    ├── BF12-node-oom-restart/
    │   ├── transcript.md
    │   └── score.md
    ├── BF13-cpu-starvation-node-marked-suspect/
    │   ├── transcript.md
    │   └── score.md
    ├── BF14-clock-skew-between-nodes/
    │   ├── transcript.md
    │   └── score.md
    └── BF15-backup-failing-gc-ttl-mismatch/
        ├── transcript.md
        └── score.md
```

---

## References

- [CockroachDB Documentation](https://www.cockroachlabs.com/docs/stable/)
- [Troubleshoot Cluster Setup](https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting)
- [Common Errors and Solutions](https://www.cockroachlabs.com/docs/stable/common-errors)
- [Troubleshoot SQL Statements](https://www.cockroachlabs.com/docs/stable/query-behavior-troubleshooting)
- [Transaction Retry Error Reference](https://www.cockroachlabs.com/docs/stable/transaction-retry-error-reference)
