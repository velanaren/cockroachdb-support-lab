# BF07 — Transaction Retry Errors — No Retry Logic
## Session Transcript

---

### Scenario Presented

**BF07 — Transaction Retry Errors — No Retry Logic**

```
From: Aiden Park
Subject: Application getting "restart transaction" errors intermittently

Hi,

We recently migrated our order processing system to CockroachDB.
The cluster seems healthy and most queries work fine, but our order
update jobs are failing randomly with an error we've never seen before:

  ERROR: restart transaction: TransactionRetryWithProtoRefreshError:
  TransactionRetryError: retry txn (RETRY_SERIALIZABLE)

It's not happening every time — maybe 1 in 20 transactions — but it's
causing orders to fail silently. We're not sure if this is a CockroachDB
bug, a configuration issue, or something we're doing wrong.

— Aiden
```

Diagnose and fix the issue. Reply when ready.

---

### Vela's Response

As per the customer, some of the transactions are failing with below error 

```
ERROR: restart transaction: TransactionRetryWithProtoRefreshError:
    TransactionRetryError: retry txn (RETRY_SERIALIZABLE)
```

1 in 20 transactions, but still it is failing and they want to know why it is happening 

Reproduced customer error 
Opened 2 terminals for this issue

```
docker exec -it crdb-node1 ./cockroach sql --insecure --host=crdb-node1:26257
docker exec -it crdb-node2 ./cockroach sql --insecure --host=crdb-node2:26257
```

Terminal 1 - Ran below commands 
```sql
BEGIN;
SELECT * FROM orders WHERE customer_id = 101;
```

Terminal 2 - Ran below commands (simultaneously with terminal 1)
```sql
BEGIN;
UPDATE orders SET status = 'shipped' WHERE customer_id = 101;
COMMIT;
```

Got below error 
```
ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionAbortedError(ABORT_REASON_CLIENT_REJECT): "sql txn" meta={id=901acd51 key=/Table/115/1/1178443844613144581/0 iso=Serializable pri=0.02561053 epo=0 ts=1779705853.396195501,0 min=1779705853.396195501,0 seq=2} lock=true stat=PENDING rts=1779705853.396195501,0 gul=1779705853.896195501,0 obs={n2@1779705853.396195501,0 n5@1779705878.705703639,0}
SQLSTATE: 40001
HINT: See: https://www.cockroachlabs.com/docs/v26.2/transaction-retry-error-reference.html#abort_reason_client_reject
```

why is this happening?

This is a feature of CockroachDB. It uses optimistic concurrency control, so transactions proceed without locking rows. At commit time, CockroachDB checks whether anything the transaction read has changed since it started.

PostgreSQL uses row-level locking. When Transaction 2 tries to read a row that Transaction 1 has locked, it waits. No retry needed because no conflict is reported — it is just a delay.

CockroachDB cannot use the same locking approach efficiently across a distributed cluster. Locks require coordination between nodes. That coordination is expensive at scale. So CockroachDB optimistically allows both transactions to proceed and resolves conflicts at commit time. The error is SQLSTATE: 40001.

Temporary Fix:

Confirm there is no open transaction
```sql
SHOW TRANSACTION STATUS;
```

Check there are no locks on the row
```sql
SELECT * FROM crdb_internal.cluster_locks
WHERE table_name = 'orders';
```

Retry the update and commit in terminal 2 — it will go through successfully.

What should we recommend to users:

Simple applications — catch 40001, retry full transaction. Easy to implement in any language. The application must catch the specific error code and retry the entire transaction from BEGIN.

Complex applications — savepoint protocol preserves transaction context better for long multi-statement transactions. CockroachDB provides a savepoint mechanism that avoids re-issuing BEGIN on every retry.

Documentation:
https://www.cockroachlabs.com/docs/stable/transaction-retry-error-reference.html
section — RETRY_SERIALIZABLE / ABORT_REASON_CLIENT_REJECT

Runbook:
This is not a typical issue to diagnose and fix — this is a feature in CockroachDB.
When a customer reports such an issue, explain the feature and provide the documentation link: https://www.cockroachlabs.com/docs/stable/transaction-retry-error-reference.html and provide the below suggestions based on the application type.

Simple applications — catch 40001, retry full transaction. Easy to implement in any language. The application must catch the specific error code and retry the entire transaction from BEGIN.

Complex applications — savepoint protocol preserves transaction context better for long multi-statement transactions. CockroachDB provides a savepoint mechanism that avoids re-issuing BEGIN on every retry.

---

### Hints

None requested.

---

### Score

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BF07 — Transaction Retry Errors — No Retry Logic
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Triage Approach        2/2
Tools Used             2/2
Documentation          2/2
Root Cause Accuracy    2/2
Runbook Quality        1/2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                  9/10
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Triage was exact — two concurrent SQL sessions on separate nodes reproduced the contention immediately, got the precise 40001 error, and confirmed cleanup with `SHOW TRANSACTION STATUS` and `crdb_internal.cluster_locks`. Tools were right: `crdb_internal.cluster_locks` is the CockroachDB-specific diagnostic for active lock inspection, and using two nodes for the concurrent sessions is more realistic than both sessions on the same node. Documentation was found and applied — transaction-retry-error-reference.html is the correct page and the fix recommendations (catch 40001 / savepoint protocol) come directly from it. Root cause is the best explanation yet — optimistic concurrency control named and explained, the distributed cost of pessimistic locking articulated, the PostgreSQL contrast is accurate, and the commit-time conflict detection mechanism is correct. Runbook is the one gap: it reads as a recommendation document rather than a numbered runbook — no steps for reproducing the error, no commands for checking `crdb_internal.cluster_locks`, no code snippet showing what a retry loop actually looks like; a support engineer picking this up cold would know what to recommend but not how to get there.
