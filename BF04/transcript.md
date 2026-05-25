# BF04 — Client Cannot Connect — Connection Refused
## Session Transcript

---

### Scenario Presented

**BF04 — Client Cannot Connect — Connection Refused**

```
From: Priya Nair
Subject: Can't connect to the database — getting a connection error

Hi,

Our application suddenly stopped being able to reach the database.
We're getting a connection error every time we try to connect. Nothing
changed on the application side that we're aware of.

We checked and the database servers are still running as far as we can
tell — the team says the cluster is up. But our app and DBeaver both
fail to connect immediately.

— Priya
```

Diagnose and fix the issue. Reply when ready.

---

### Vela's Response

customer says the cluster is healthy, nodes are available but when they connect to database they get connection refused error. 

First check is to check the cluster health and node status 

```
docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257

  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              | locality | attrs | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+----------+-------+--------------+----------
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-25 03:31:21.650925 +0000 UTC | 2026-05-25 03:50:09.169802 +0000 UTC |          | []    | true         | true
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-25 03:31:22.861657 +0000 UTC | 2026-05-25 03:50:11.312851 +0000 UTC |          | []    | true         | true
   5 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-25 03:43:58.148275 +0000 UTC | 2026-05-25 03:50:10.903811 +0000 UTC |          | []    | true         | true
(3 rows)
```

Admin UI also says all 3 nodes live

| nodes | Uptime | Replicas | Capacity Usage | Memory Use | vCPUs | Version | Status | |
|---|---|---|---|---|---|---|---|---|
| crdb-node2:26257 (n2) | 20 minutes | 58 | 0 % | 12 % | 8 | v26.2.0 | LIVE | Logs |
| crdb-node3:26257 (n3) | 20 minutes | 58 | 0 % | 11 % | 8 | v26.2.0 | LIVE | Logs |
| crdb-node1:26257 (n5) | 7 minutes | 58 | 0 % | 10 % | 8 | v26.2.0 | LIVE | Logs |

Nodes listen to outside request using listen-addr, it should be 0.0.0.0 to receive requests from external, if connection is refused we need to check what is the listen-addr for the 3 nodes in cockroach logs

```
docker exec -it crdb-node1 grep -i "listen-addr" /cockroach/cockroach-data/logs/cockroach.log | head -20

I260525 03:43:57.337050 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--insecure› ‹--listen-addr=0.0.0.0:26257› ‹--advertise-addr=crdb-node1:26257› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]

I260525 03:31:19.548753 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--insecure› ‹--listen-addr=0.0.0.0:26257› ‹--advertise-addr=crdb-node2:26257› ‹--http-addr=0.0.0.0:8080› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]

I260525 03:31:21.590673 1 util/log/file_sync_buffer.go:237 ⋮ [config]   arguments: [‹/cockroach/cockroach› ‹start› ‹--insecure› ‹--listen-addr=0.0.0.0:26257› ‹--advertise-addr=crdb-node3:26257› ‹--http-addr=0.0.0.0:8080› ‹--join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257› ‹--cache=256MiB› ‹--max-sql-memory=256MiB›]
```

The listen-addr is correct for all 3 nodes

Tried connecting to DB using Dbeaver and getting below error - "Connection to localhost:26257 refused. Check that the hostname and port are correct and that the postmaster is accepting TCP/IP connections. Connection refused" same as the user got 

cockroach logs also indicate the connection refused error 

```
docker exec -it crdb-node3 grep -i "refused" /cockroach/cockroach-data/logs/cockroach.log | head -20

ng dialback connection failed to ‹crdb-node1:26257›, n5, ‹connection error: desc = "transport: error while dialing: dial tcp 172.26.0.2:26257: connect: connection refused"›
```

```
docker exec -it crdb-node2 grep -i "refused" /cockroach/cockroach-data/logs/cockroach.log | head -20

I260525 03:41:51.776961 33673 rpc/context.go:2456 ⋮ [rnode=5,raddr=‹crdb-node1:26257›,class=system,rpc,dialback] 842  blocking dialback connection failed to ‹crdb-node1:26257›, n5, ‹connection error: desc = "transport: error while dialing: dial tcp 172.26.0.2:26257: connect: connection refused"›
```

No such errors for node 1. What we can understand from the logs, 

```
DNS resolution works    -   crdb-node1 → 172.26.0.2
Network path works      -   packet reached 172.26.0.2
Port 26257 refused       -   nothing listening on that port
```

This is for node 1 alone. it is a port issue. In node 1 cockroach db is not listening in port 26257

docker ps indicates there is no port mapping for node 1 

```
docker ps | grep cockroach

26b333439610   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   37 minutes ago   Up 37 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp                                                    crdb-node1
3da528b6b213   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   22 hours ago     Up 50 minutes   0.0.0.0:8082->8080/tcp, [::]:8082->8080/tcp, 0.0.0.0:26259->26257/tcp, [::]:26259->26257/tcp   crdb-node3
55ea0e237414   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   22 hours ago     Up 50 minutes   0.0.0.0:8081->8080/tcp, [::]:8081->8080/tcp, 0.0.0.0:26258->26257/tcp, [::]:26258->26257/tcp   crdb-node2
```

Fix is to stop, remove the crdb-node1 and restart with correct port mappings -p 26257:26257 

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

Port is mapped now, verify using docker ps command 

```
docker ps | grep cockroach

fb7e09868998   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   11 seconds ago   Up 9 seconds    0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp, 0.0.0.0:26257->26257/tcp, [::]:26257->26257/tcp   crdb-node1
3da528b6b213   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   22 hours ago     Up 57 minutes   0.0.0.0:8082->8080/tcp, [::]:8082->8080/tcp, 0.0.0.0:26259->26257/tcp, [::]:26259->26257/tcp   crdb-node3
55ea0e237414   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   22 hours ago     Up 57 minutes   0.0.0.0:8081->8080/tcp, [::]:8081->8080/tcp, 0.0.0.0:26258->26257/tcp, [::]:26258->26257/tcp   crdb-node2
```

DBeaver is able to connect to DB, the connection refused error is resolved.

All nodes and cluster health is good 

```
docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257

  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              | locality | attrs | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+----------+-------+--------------+----------
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-25 03:31:21.650925 +0000 UTC | 2026-05-25 04:31:12.986155 +0000 UTC |          | []    | true         | true
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-25 03:31:22.861657 +0000 UTC | 2026-05-25 04:31:14.086458 +0000 UTC |          | []    | true         | true
   5 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-25 04:28:53.372634 +0000 UTC | 2026-05-25 04:31:12.033471 +0000 UTC |          | []    | true         | true
(3 rows)
```

Able to connect to SQL shell 

```
docker exec -it crdb-node1 ./cockroach sql --insecure --host=crdb-node1:26257

Welcome to the CockroachDB SQL shell.
All statements must be terminated by a semicolon.
To exit, type: \q.
 Server version: CockroachDB CCL v26.2.0 (aarch64-unknown-linux-gnu, built 2026/04/21 18:34:41, go1.25.5) (same version as client)
 Cluster ID: af9fe653-6a58-4cf0-a3ae-2a630fda7590
Enter \? for a brief introduction.
root@crdb-node1:26257/defaultdb>
```

Documentation:
https://www.cockroachlabs.com/docs/v26.2/cluster-setup-troubleshooting.html
specific section - client connection issue 

Runbook:
1. check the cluster health and node status - `docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257`
2. If all nodes are healthy, try connecting to db using DBeaver - it may show connection refused error 
3. Nodes listen to connections using the listen-addr - check if listen-addr is configured properly in cockroach logs - `docker exec -it crdb-node1 grep -i "listen-addr" /cockroach/cockroach-data/logs/cockroach.log | head -20`
4. If it is incorrect then the issue is with listen-addr , we need to stop and remove the node and restart using correct listen-addr which is 0.0.0.0:26257
5. If all listen-addr are right , check the cockroach logs for connection refused errors for all 3 nodes 
6. `docker exec -it crdb-node3 grep -i "refused" /cockroach/cockroach-data/logs/cockroach.log | head -20`
    What we can understand from the logs, 
    
    DNS resolution works    -   crdb-node1 → 172.26.0.2
    Network path works      -   packet reached 172.26.0.2
    Port 26257 refused       -   nothing listening on that port
       
    This is a port issue, node 1 is not listening on 26257
7. check the port mapping using docker ps command - `docker ps | grep cockroach` - this will indicate node1 port is not mapped
8. Fix is to stop and remove the container and restart using correct port mapping 
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
9. verify using docker ps command to see if port is mapped now - `docker ps | grep cockroach`
10. verify the node status - `docker exec -it crdb-node1 ./cockroach node status --insecure --host=crdb-node1:26257`
11. login to SQL shell in node1 and check if we are able to login successfully - `docker exec -it crdb-node1 ./cockroach sql --insecure --host=crdb-node1:26257`
12. check if DBeaver is able to connect to DB 

---

### Hints

None requested.

---

### Score

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BF04 — Client Cannot Connect — Connection Refused
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

Triage was systematic and correct — node status, Admin UI, connection test, listen-addr check, log analysis, docker ps, fix — no detours, no wrong layers. Tools were exactly right: CockroachDB internal logs for the refused errors gave you the IP-level evidence (DNS resolved, packet arrived, port rejected), and docker ps confirmed the missing port mapping. Documentation was the right page and section, found without a URL being handed to you, but there's no evidence the doc content informed the fix — the fix came entirely from your own analysis, and the citation reads like it was added after the fact. Root cause correctly named the missing port mapping, but the mechanism wasn't explained — Docker publishes ports by adding iptables NAT rules forwarding host traffic to the container; without `-p 26257:26257`, no such rule exists and the host kernel returns connection refused to anything hitting that port. Runbook is the strongest yet — this is the first session where it earns 2/2: full docker run fix command present, explicit multi-step verification at the end (docker ps, node status, SQL shell, DBeaver), and specific commands at every critical step.
