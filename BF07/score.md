# BF07 — Transaction Retry Errors — No Retry Logic

**Score: 9/10**

| Dimension | Score |
|---|---|
| Triage Approach | 2/2 |
| Tools Used | 2/2 |
| Documentation | 2/2 |
| Root Cause Accuracy | 2/2 |
| Runbook Quality | 1/2 |

## Rationale

**Triage Approach — 2/2**
Immediately reproduced the error with two concurrent SQL sessions on separate nodes, confirmed the 40001 SQLSTATE, then used `SHOW TRANSACTION STATUS` and `crdb_internal.cluster_locks` to clean up and verify — systematic and correct for a transaction contention scenario.

**Tools Used — 2/2**
Two concurrent sessions on separate nodes (crdb-node1 and crdb-node2) correctly simulate distributed transaction contention, and `crdb_internal.cluster_locks` is the CockroachDB-specific view for inspecting active locks — the exact right tools for this failure type.

**Documentation — 2/2**
transaction-retry-error-reference.html is the correct and specific page for 40001 errors, and the fix recommendations (catch 40001 / savepoint protocol) were applied directly from its content.

**Root Cause Accuracy — 2/2**
Best root cause explanation across all sessions: optimistic concurrency control named and explained, commit-time conflict detection described accurately, the distributed cost of pessimistic locking articulated, and the PostgreSQL contrast correctly drawn — mechanism fully explained.

**Runbook Quality — 1/2**
The runbook correctly identifies this as a feature explanation rather than a break-fix, but reads as a recommendation document rather than numbered steps — no reproduction commands, no `crdb_internal.cluster_locks` query, no code snippet showing what a retry loop looks like; a support engineer picking this up cold would know what to say but not how to diagnose or demonstrate it.

## What would a 10 look like for this scenario

A perfect runbook would include: (1) numbered reproduction steps with the exact two-terminal SQL sequence to demonstrate the error; (2) the `crdb_internal.cluster_locks` query with expected output; (3) a concrete retry loop code snippet — e.g., Python pseudocode showing `while retries < MAX: try: BEGIN; ...; COMMIT; break; except error 40001: rollback; retries++`; and (4) a note distinguishing `RETRY_SERIALIZABLE` (read-write conflict detected at commit) from `ABORT_REASON_CLIENT_REJECT` (conflicting transaction aborted by a peer), since customers often see both and need to understand they require the same fix.
