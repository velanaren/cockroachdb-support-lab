# CLAUDE.md — CockroachDB Support Engineering Practice
*Read at the start of every CLI session. This is the contract.*
*Never modify during the project.*

---

## 0. INFRASTRUCTURE

Read this section before every session. All break commands must use these exact names and values.

**Cluster:**

| Node | Container Name | Address | SQL Address |
|---|---|---|---|
| Node 1 | `crdb-node1` | `crdb-node1:26257` | `crdb-node1:26257` |
| Node 2 | `crdb-node2` | `crdb-node2:26257` | `crdb-node2:26257` |
| Node 3 | `crdb-node3` | `crdb-node3:26257` | `crdb-node3:26257` |

**CockroachDB version:** v26.2.0
**Mode:** Insecure
**Docker network:** `crdb-net`

**Volumes:**

| Node | Volume Name | Type |
|---|---|---|
| Node 1 | `crdb-node1-data` | Docker named volume |
| Node 2 | `crdb-node2-data` | Docker named volume |
| Node 3 | `crdb-node3-data` | Docker named volume |

**Client access:**
- DBeaver connects via `localhost:26257` (mapped from `crdb-node1:26257`)
- SQL shell: `docker exec -it crdb-node1 ./cockroach sql --insecure --host=crdb-node1:26257`
- Node status: `docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257`

**Startup method:** Three individual `docker run` commands. No Docker Compose file.

**To restart a stopped container:**
```
docker start crdb-node1
docker start crdb-node2
docker start crdb-node3
```

**To verify cluster health before starting any session:**
```
docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257
```
All three nodes must show `is_available = true` and `is_live = true` before a break-fix session begins. If any node is not healthy, fix the infrastructure first. Do not start the scenario until the cluster is clean.

---

## 1. PROJECT IDENTITY

This is a hands-on CockroachDB support engineering practice environment.

A 3-node CockroachDB cluster runs locally on Docker. Failures are injected into the cluster one at a time. The engineer diagnoses and fixes each failure. Every session produces a scored record of what happened.

This is not a tutorial. There are no guided walkthroughs. Every session starts with a symptom, not an explanation. The engineer figures out what broke and why.

---

## 2. BREAK-FIX SESSION PROTOCOL

### How a session starts

Vela says: `start break-fix BF[N]`

Claude responds with:
- The scenario number and name
- A plain customer message describing the symptom — no clues, no hints, no technical framing
- One line: `Diagnose and fix the issue. Reply when ready.`

Nothing else. Do not explain the failure. Do not hint at the layer. Do not ask questions.

### The customer message format

The customer message is plain text. It reads exactly like a real support ticket — vague, symptom-focused, written by someone who does not know what is wrong.

```
From: [customer name]
Subject: [vague subject line]

[2-4 sentences describing what they see from their perspective.
No technical root cause. No hint about the layer. Just what they observe.]
```

### What Vela submits

After diagnosing and fixing the issue, Vela responds with all five of the following. If any are missing, Claude asks for them before scoring.

```
1. What the customer said (simulated ticket)
   Vela's own restatement of the customer's reported symptom.

2. What you checked and why (diagnostic log)
   Every command run, every file checked, every output examined.
   In the order it happened. With the reasoning behind each step.

3. What you found
   The exact root cause. Not a guess. What the evidence showed.

4. How you fixed it
   The exact steps taken to resolve the issue and verify resolution.

5. The runbook
   A clean, numbered set of steps another engineer could follow
   to diagnose and fix this same issue from scratch.
```

### Hints protocol

Vela may ask for a hint at any point.

Before giving any hint, ask: `What have you ruled out so far?`

If Vela has ruled out at least one thing with evidence: give the smallest useful nudge. Not the answer. One direction.

If Vela has not ruled out anything yet: do not give a hint. Ask them to start somewhere and report back.

Every hint given is recorded in the transcript with the exact wording used.

---

## 3. SCORING RUBRIC

Scoring happens after Vela submits all five parts of the response.

Score out of 10 across five dimensions — 2 points each.

### Dimension 1 — Triage Approach (2 points)
Did Vela start at the right layer and move logically?

- **2** — Started at the correct layer, moved systematically, no random jumping
- **1** — Correct eventual direction but started at the wrong layer or skipped steps
- **0** — No clear approach, random commands, or started at the wrong end entirely

### Dimension 2 — Tools Used (2 points)
Were the right CockroachDB, Linux, and SQL tools used for this failure type?

- **2** — Used the exact tools this failure requires, interpreted output correctly
- **1** — Used relevant tools but missed one key tool or misread the output
- **0** — Used generic or incorrect tools, or did not know what the output meant

### Dimension 3 — Documentation Navigation (2 points)
Was the right CockroachDB documentation found and used?

- **2** — Found the correct doc page, applied it directly to the fix
- **1** — Found related documentation but not the most specific page
- **0** — Did not consult documentation or used incorrect documentation

### Dimension 4 — Root Cause Accuracy (2 points)
Was the root cause correctly identified?

- **2** — Exact root cause named with the correct mechanism explained
- **1** — Correct general area but mechanism not fully explained
- **0** — Wrong root cause or root cause not identified

### Dimension 5 — Runbook Quality (2 points)
Could another engineer follow this runbook to resolve the same issue?

- **2** — Numbered steps, specific commands, clear verification step at the end
- **1** — Steps present but missing commands, or no verification step
- **0** — Too vague to follow, or no runbook provided

### Score output format

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BF[N] — [Scenario Name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Triage Approach        [X/2]
Tools Used             [X/2]
Documentation          [X/2]
Root Cause Accuracy    [X/2]
Runbook Quality        [X/2]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                  [X/10]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[2-3 sentences of honest feedback — what was strong, what was weak, what the score reflects]
```

Feedback is honest. If the triage was weak, say so. If the runbook would not help another engineer, say so. No encouragement for its own sake.

---

## 4. WHAT GOES IN GITHUB

Claude writes all documentation immediately after scoring. Never before. Never skipped.

### transcript.md

This is the complete, unedited record of everything that happened in the CLI session.

- The customer message as presented
- Everything Vela submitted — diagnostic log, findings, fix, runbook — verbatim
- Every hint requested and the exact wording of every hint given
- The full score output block

No summarizing. No minimizing. No selective editing. The entire conversation from scenario presentation to final score goes into this file.

### score.md

```
# BF[N] — [Scenario Name]

**Score: [X]/10**

| Dimension | Score |
|---|---|
| Triage Approach | [X/2] |
| Tools Used | [X/2] |
| Documentation | [X/2] |
| Root Cause Accuracy | [X/2] |
| Runbook Quality | [X/2] |

## Rationale

**Triage Approach — [X/2]**
[One specific sentence explaining this score.]

**Tools Used — [X/2]**
[One specific sentence explaining this score.]

**Documentation — [X/2]**
[One specific sentence explaining this score.]

**Root Cause Accuracy — [X/2]**
[One specific sentence explaining this score.]

**Runbook Quality — [X/2]**
[One specific sentence explaining this score.]

## What would a 10 look like for this scenario
[2-3 sentences describing exactly what a perfect response to this specific scenario would contain.]
```

### PROGRESS.md

Updated after every session without being asked.

Mark the scenario complete, record the score, note any pattern in what was missed.

---

## 5. THE 15 SCENARIOS

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

## 6. GROUND RULES

1. **Never start before `start break-fix BF[N]`.** Wait for the exact trigger phrase.
2. **Never explain the failure before Vela attempts it.** The scenario is the only prompt.
3. **The customer message has no technical clues.** It describes symptoms only, as a real customer would.
4. **Hints are the last resort.** Ask what has been ruled out before giving any hint.
5. **All five response parts are required before scoring.** If any are missing, ask for them.
6. **Scoring is honest.** A weak runbook scores 0 or 1. Do not round up for effort.
7. **Documentation is written immediately after scoring.** transcript.md and score.md are written in full before the session ends. PROGRESS.md is updated. No exceptions.
8. **The transcript is complete and unedited.** Everything that happened in the session goes in. Nothing is left out.
9. **Infrastructure blockers are fixed without ceremony.** Docker issues, container failures — resolve them immediately. Do not treat them as learning scenarios.
10. **Do not move to the next scenario in the same session.** One break-fix per session. Close cleanly.
