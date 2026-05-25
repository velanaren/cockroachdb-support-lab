# BF05 — Wrong Connection String / Wrong Port
## Session Transcript

---

### Scenario Presented

**BF05 — Wrong Connection String / Wrong Port**

```
From: Marcus Webb
Subject: Database stopped accepting connections after server maintenance

Hi,

We did some routine maintenance on our database servers over the weekend
and now our application can't connect. We're seeing connection errors but
the ops team says the database is running and healthy.

We haven't changed anything on the application side. The connection string
is the same one that's been working for months.

— Marcus
```

Diagnose and fix the issue. Reply when ready.

---

### Vela's Response

customer is saying the node and cluster are healthy but they are not able to connect to Database and receiving connection refused error 

Tried checking the connecting string using DBeaver. It is throwing an error 

```
Connection to localhost:26257 refused. Check that the hostname and port are correct and that the postmaster is accepting TCP/IP connections.
  Connection refused
```

checking the node and cluster health - there are no issues

```
docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257

  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              | locality | attrs | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+----------+-------+--------------+----------
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-25 03:31:21.650925 +0000 UTC | 2026-05-25 06:01:52.816011 +0000 UTC |          | []    | true         | true
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-25 03:31:22.861657 +0000 UTC | 2026-05-25 06:01:52.410945 +0000 UTC |          | []    | true         | true
   5 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-25 05:53:14.399021 +0000 UTC | 2026-05-25 06:01:52.013695 +0000 UTC |          | []    | true         | true
(3 rows)
```

Admin UI also indicates there are no issues

| nodes | Uptime | Replicas | Capacity Usage | Memory Use | vCPUs | Version | Status | |
|---|---|---|---|---|---|---|---|---|
| crdb-node2:26257 (n2) | 3 hours | 58 | 0 % | 13 % | 8 | v26.2.0 | LIVE | Logs |
| crdb-node3:26257 (n3) | 3 hours | 58 | 1 % | 12 % | 8 | v26.2.0 | LIVE | Logs |
| crdb-node1:26257 (n5) | 14 minutes | 58 | 0 % | 12 % | 8 | v26.2.0 | LIVE | Logs |

cluster is fine and client is not able to connect. The nodes receive request externally through listen-addr will check if it is configured correctly for all 3 nodes

```
docker exec -it crdb-node1 grep -i "listen-addr" /cockroach/cockroach-data/logs/cockroach.log | head -20

I260525 05:53:12.054620 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--insecure› ‹--listen-addr=0.0.0.0:26257› ‹--advertise-addr=crdb-node1:26257› ‹--http-addr=0.0.0.0:8080› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]

I260525 03:31:19.548753 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--insecure› ‹--listen-addr=0.0.0.0:26257› ‹--advertise-addr=crdb-node2:26257› ‹--http-addr=0.0.0.0:8080› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]

I260525 03:31:21.590673 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--insecure› ‹--listen-addr=0.0.0.0:26257› ‹--advertise-addr=crdb-node3:26257› ‹--http-addr=0.0.0.0:8080› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]
```

The DBeaver error says connection refused and there is an issue in hostname/port 

hostname is localhost and port is 26257

docker ps indicates the port is 26260

```
docker ps | grep cockroach

d6791da355f8   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   32 minutes ago   Up 32 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp, 0.0.0.0:26260->26257/tcp, [::]:26260->26257/tcp   crdb-node1
3da528b6b213   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   24 hours ago     Up 3 hours      0.0.0.0:8082->8080/tcp, [::]:8082->8080/tcp, 0.0.0.0:26259->26257/tcp, [::]:26259->26257/tcp   crdb-node3
55ea0e237414   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   24 hours ago     Up 3 hours      0.0.0.0:8081->8080/tcp, [::]:8081->8080/tcp, 0.0.0.0:26258->26257/tcp, [::]:26258->26257/tcp   crdb-node2
```

docker port mapping allows external traffic to reach the isolated container, so from the external environment if they are reaching port 26260 then the traffic will be allowed to reach port 26257 in node 1 but the connection string says it is reaching port 26257 instead of 26260 so it is failing with connection refused error 

Tried changing the port to 26260 in the connection string and it worked fine in DBeaver

it says connected(739 ms)

```
Whatever port is mapped externally must match what the client dials

External mapping: 26260:26257  →  client must dial 26260
External mapping: 26257:26257  →  client must dial 26257
External mapping: 5000:26257   →  client must dial 5000
The internal port inside the cockroachdb container is 26257 because that is what cockroachdb listens on
```

Documentation:
https://www.cockroachlabs.com/docs/stable/connection-parameters.html — specifically: connect using URL section
https://www.cockroachlabs.com/docs/v26.2/cluster-setup-troubleshooting — client connection issues

Runbook:
1. check the health of the node and cluster
     `docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257`
2. check Admin UI for any issues and node health 
3. connect from a client (e.g. DBeaver) to check the connection
4. cluster is fine and client is not able to connect, the 3 nodes accept external requests using listen-addr, ensure it is configured properly. It should be 0.0.0.0:26257 to accept requests from anywhere and on port 26257
     `docker exec -it crdb-node1 grep -i "listen-addr" /cockroach/cockroach-data/logs/cockroach.log | head -20`
5. If listen-addr is configured properly for all 3 nodes, then check the docker ports, the connection refused error said there is an issue with port/hostname
     `docker ps | grep cockroach`
     check the port mapping, check how the external connections reach CockroachDB
6. The connection string in DBeaver should match the external port mapping in docker ps command 
7. If it is not matching, update the port number in DBeaver with the external port mapped
     External mapping: 26260:26257  →  client must dial 26260
     External mapping: 26257:26257  →  client must dial 26257
     External mapping: 5000:26257   →  client must dial 5000
8. Verify the DBeaver connection - it should say (connected)
9. Verify the node health `docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257`
10. Verify the node and cluster health in Admin UI

---

### Hints

None requested.

---

### Score

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BF05 — Wrong Connection String / Wrong Port
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Triage Approach        2/2
Tools Used             2/2
Documentation          1/2
Root Cause Accuracy    1/2
Runbook Quality        2/2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                  8/10
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Triage was systematic — cluster health confirmed, DBeaver reproduced the error, listen-addr ruled out, docker ps found the port mismatch, connection string updated and verified. Tools were exact and interpreted correctly: docker ps is the right tool to expose a host port mapping discrepancy. Two documentation pages cited this time, with connection-parameters.html being the more specific and correct page for this scenario, but neither doc was described or applied — the fix came entirely from reading docker ps output. Root cause correctly named the mismatch (host port 26260 vs connection string port 26257), and the port mapping rule is clearly articulated, but the actual mechanism wasn't explained — when a container is stopped and recreated, its `-p` flag bindings are not preserved from the previous run; the new `docker run` command dictates the new host port, so any client still using the old port gets connection refused. Runbook is again 2/2: specific commands at every critical step, the port mapping rule is formalized clearly, and explicit verification at the end.
