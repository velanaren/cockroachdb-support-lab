# BF06 — Client Cannot Connect — TLS Certificate Error
## Session Transcript

---

### Scenario Presented

**BF06 — Client Cannot Connect — TLS Certificate Error**

```
From: Sofia Reyes
Subject: Nobody can connect to our primary database node after security upgrade

Hi,

Our security team enabled TLS on our database cluster last night as part
of a compliance requirement. Since then, nobody can connect to node1.
The node process is running and the ops team can see it in their
monitoring, but every client — our application, DBeaver, everything —
fails to connect.

Nodes 2 and 3 are fine. Only node1 is affected.

— Sofia
```

Diagnose and fix the issue. Reply when ready.

---

### Vela's Response

Customer says cannot connect to Node 1 after TLS upgrade 
Application, DBeaver everything fails to connect, node 2 and node 3 are healthy 

First step to check the node and cluster health 

```
docker exec -it crdb-node2 ./cockroach node status --insecure --host=crdb-node2:26257

  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              | locality | attrs | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+----------+-------+--------------+----------
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-25 03:31:21.650925 +0000 UTC | 2026-05-25 08:28:16.519257 +0000 UTC |          | []    | true         | true
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-25 03:31:22.861657 +0000 UTC | 2026-05-25 08:28:17.806345 +0000 UTC |          | []    | true         | true
   5 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-25 07:08:48.879102 +0000 UTC | 2026-05-25 07:17:42.036745 +0000 UTC |          | []    | false        | false
(3 rows)
```

Node 1 is not available. Admin UI also confirms the same Node 1 is dead

| nodes | Uptime | Replicas | Capacity Usage | Memory Use | vCPUs | Version | Status | |
|---|---|---|---|---|---|---|---|---|
| crdb-node2:26257 (n2) | 5 hours | 58 | 1 % | 11 % | 8 | v26.2.0 | LIVE | Logs |
| crdb-node3:26257 (n3) | 5 hours | 58 | 1 % | 12 % | 8 | v26.2.0 | LIVE | Logs |
| crdb-node1:26257 (n5) | an hour (stale) | 58 (stale) | 0 % (stale) | 16 % (stale) | 8 | v26.2.0 | DEAD | |

check the cockroach logs for node 1

```
docker exec -it crdb-node1 cat /cockroach/cockroach-data/logs/cockroach.log | tail -20

W260525 08:29:53.204747 221713 gossip/client.go:121 ⋮ [T1,Vsystem,n5] 14286  failed to start gossip client to ‹crdb-node2:26257›: initial connection heartbeat failed: grpc: ‹connection error: desc = "transport: authentication handshake failed: tls: first record does not look like a TLS handshake"› [code 14/Unavailable]
```

it failed with TLS handshake, it tried connecting to other nodes in cluster and failed with TLS handshake error. The customer also said there was a TLS upgrade, the error indicates it is expecting TLS from node 2 and it is not receiving TLS so it is failing. This means node1 might be secure and node 2 and node 3 are not secure causing this error 

Confirm which node is secure and which node is not secure 

check cockroach logs

```
docker exec -it crdb-node1 \
  grep "insecure\|secure\|certs" \
  /cockroach/cockroach-data/logs/cockroach.log | head -5

I260525 07:16:08.140147 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--certs-dir=/cockroach/certs› ‹--advertise-addr=crdb-node1:26257› ‹--http-addr=0.0.0.0:8080› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]
```

```
docker exec -it crdb-node3 \
  grep "insecure\|secure\|certs" \
  /cockroach/cockroach-data/logs/cockroach.log | head -5

I260525 03:31:21.590673 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--insecure› ‹--listen-addr=0.0.0.0:26257› ‹--advertise-addr=crdb-node3:26257› ‹--http-addr=0.0.0.0:8080› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]
I260525 03:31:21.590738 1 1@cli/start.go:1489 ⋮ [n?] 2  To start a secure server without mandating TLS for clients,
I260525 03:31:21.590738 1 1@cli/start.go:1489 ⋮ [n?] 2 +- https://www.cockroachlabs.com/docs/v26.2/secure-a-cluster.html
I260525 03:31:25.425804 73 1@cli/start.go:1304 ⋮ [T1,Vsystem,n3] 82 +RPC client flags:    ‹/cockroach/cockroach <client cmd> --host=crdb-node3:26257 --insecure›
```

```
docker exec -it crdb-node2 \
  grep "insecure\|secure\|certs" \
  /cockroach/cockroach-data/logs/cockroach.log | head -5

I260525 03:31:19.548753 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--insecure› ‹--listen-addr=0.0.0.0:26257› ‹--advertise-addr=crdb-node2:26257› ‹--http-addr=0.0.0.0:8080› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]
I260525 03:31:19.548840 1 1@cli/start.go:1489 ⋮ [n?] 2  To start a secure server without mandating TLS for clients,
I260525 03:31:19.548840 1 1@cli/start.go:1489 ⋮ [n?] 2 +- https://www.cockroachlabs.com/docs/v26.2/secure-a-cluster.html
I260525 03:31:25.420948 86 1@cli/start.go:1304 ⋮ [T1,Vsystem,n2] 91 +RPC client flags:    ‹/cockroach/cockroach <client cmd> --host=crdb-node2:26257 --insecure›
```

Node 1 is secure - node 2 and node 3 are not secure. So node is not able to join the cluster and DBeaver connects to node 1 since node 1 is not able to join the cluster it is showing as not available and DBeaver connection times out as node 1 is not available. This matches with the customer description of the problem 

To fix this as practice, we are bringing down node 1 and making it also insecure so it can communicate to other nodes, in production the best practice is if one node is secure make other nodes also secure 

```
docker stop crdb-node1
docker rm crdb-node1
```

```
docker run -d \
  --name=crdb-node1 \
  --hostname=crdb-node1 \
  --network=crdb-net \
  -p 26257:26257 \
  -p 8080:8080 \
  -v crdb-node1-data:/cockroach/cockroach-data \
  --ulimit nofile=35000:35000 \
  cockroachdb/cockroach:latest start \
  --insecure \
  --listen-addr=0.0.0.0:26257 \
  --advertise-addr=crdb-node1:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257 \
  --cache=256MiB \
  --max-sql-memory=256MiB
```

check the node status - affected node should be available now 

```
docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257

  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              | locality | attrs | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+----------+-------+--------------+----------
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-25 03:31:21.650925 +0000 UTC | 2026-05-25 08:54:14.700276 +0000 UTC |          | []    | true         | true
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-25 03:31:22.861657 +0000 UTC | 2026-05-25 08:54:15.094988 +0000 UTC |          | []    | true         | true
   5 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-25 08:44:21.303842 +0000 UTC | 2026-05-25 08:54:15.707428 +0000 UTC |          | []    | true         | true
(3 rows)
```

check DBeaver to check if it is connecting now 
connected (22589ms)

Documentation:
https://www.cockroachlabs.com/docs/stable/security-reference/transport-layer-security.html — section: TLS between CockroachDB nodes
https://www.cockroachlabs.com/docs/stable/logging-overview — remaining debugging and commands are from cockroach logs

Runbook:
1. check the status of node and cluster
     `docker exec -it crdb-node2 ./cockroach node status --insecure --host=crdb-node2:26257`
2. check the status in Admin UI
3. if any of the nodes is not available/showing as dead in Admin UI, check the cockroach logs for that node
     `docker exec -it crdb-node1 cat /cockroach/cockroach-data/logs/cockroach.log | tail -20`
4. If the log indicates the issue is due to TLS handshake, check if all 3 nodes are secure
     `docker exec -it crdb-node1 grep "insecure\|secure\|certs" /cockroach/cockroach-data/logs/cockroach.log | head -5`
5. All nodes should be either secure or insecure for it to communicate with each other and join the cluster
6. if the nodes don't have the same status ensure to make everything secure/insecure (workaround) so the nodes communicate, by bringing down the node impacted and running docker run command to make it secure or insecure
```
docker stop crdb-node1
docker rm crdb-node1

docker run -d \
  --name=crdb-node1 \
  --hostname=crdb-node1 \
  --network=crdb-net \
  -p 26257:26257 \
  -p 8080:8080 \
  -v crdb-node1-data:/cockroach/cockroach-data \
  --ulimit nofile=35000:35000 \
  cockroachdb/cockroach:latest start \
  --insecure \
  --listen-addr=0.0.0.0:26257 \
  --advertise-addr=crdb-node1:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257 \
  --cache=256MiB \
  --max-sql-memory=256MiB
```
7. check the node status
     `docker exec -it crdb-node2 ./cockroach node status --insecure --host=crdb-node2:26257`
8. Try making connection using DBeaver to check if the connection issue is resolved

---

### Hints

None requested.

---

### Score

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BF06 — Client Cannot Connect — TLS Certificate Error
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Triage Approach        2/2
Tools Used             2/2
Documentation          2/2
Root Cause Accuracy    1/2
Runbook Quality        2/2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                  9/10
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Triage was clean end-to-end — node status, Admin UI, CockroachDB internal logs, targeted grep to confirm which nodes are secure versus insecure, fix applied, verified. Tools were exactly right: grepping the internal log for `insecure|secure|certs` is the precise tool to confirm startup configuration across all three nodes, and reading the tail of the log surfaced the TLS handshake error directly. Documentation is the first 2/2 on this dimension — transport-layer-security.html is the correct and specific page for inter-node TLS behavior, and the understanding that all nodes in a cluster must share the same security mode was applied directly to diagnose the mismatch and inform the fix. Root cause correctly identified the secure/insecure mismatch and correctly interpreted the TLS handshake error from the logs, but the mechanism wasn't explained — CockroachDB uses mutual TLS (mTLS) for all inter-node gRPC connections; when node1 (secure) tries to connect to node2 (insecure), it sends a TLS ClientHello, node2 responds with a plaintext gRPC frame, and node1 rejects it with "first record does not look like a TLS handshake," breaking gossip and raft heartbeats entirely. Runbook earns 2/2 again — targeted log commands, the full fix command, the security mode matching rule clearly stated, and explicit verification at the end.
