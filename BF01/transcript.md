# BF01 — Node Fails to Start — Bad Data Directory
## Session Transcript

---

### Scenario Presented

**BF01 — Node Fails to Start — Bad Data Directory**

```
From: Marcus Webb
Subject: One of our database servers keeps crashing

Hi team,

We restarted our CockroachDB setup this morning after some routine maintenance
and one of the nodes keeps failing to come up. The other two nodes seem fine and
the cluster is still responding to queries, but every time we try to bring the
third node back online it just dies immediately.

We haven't made any config changes that we know of. Not sure what to look at next.

— Marcus
```

Diagnose and fix the issue. Reply when ready.

---

### Vela's Response

The customer say's 1/3 node is failing to come up and it crashes immediately. Other 2 nodes are fine and still responding to queries. 

Check the node status ( node status --all)

```
docker exec -it crdb-node2 ./cockroach node status --all --insecure --host=crdb-node2:26257

  id |     address      |   sql_address    |  build  |         started_at           |         updated_at           | locality | attrs | is_available | is_live | replicas_leaders | replicas_leaseholders | ranges | ranges_unavailable | ranges_underreplicated | is_decommissioning | membership | is_draining
-----+------------------+------------------+---------+------------------------------+------------------------------+----------+-------+--------------+---------+------------------+-----------------------+--------+--------------------+------------------------+--------------------+------------+--------------
   1 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-24 08:08:47.8538 UTC | 2026-05-24 08:09:08.961 UTC  |          | []    | false        | false   |                0 |                     0 |     56 |                  0 |                      0 | false              | active     | true
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-24 06:15:51.014 UTC  | 2026-05-24 08:29:54.572 UTC  |          | []    | true         | true    |               23 |                    23 |     56 |                  0 |                     23 | false              | active     | false
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-24 06:16:00.852 UTC  | 2026-05-24 08:29:55.976 UTC  |          | []    | true         | true    |               33 |                    33 |     56 |                  0 |                     33 | false              | active     | false
(3 rows)
```

node 1 is not active and is_draining is true 

Admin UI indicates node 1 is dead 

| nodes | Uptime | Replicas | Capacity Usage | Memory Use | vCPUs | Version | Status |
|---|---|---|---|---|---|---|---|
| crdb-node1:26257 (n1) | 31 minutes (stale) | 56 (stale) | 0 % (stale) | 12 % (stale) | 8 | v26.2.0 | DEAD |

check the node 1 Logs 

```
docker logs crdb-node1

initiating hard shutdown of server
too early to drain; used hard shutdown instead
*
* ERROR: ERROR: server startup failed: failed to start server: failed to create engines: pebble: malformed manifest file "MANIFEST-001086" for DB "/cockroach/cockroach-data"
*
ERROR: server startup failed: failed to start server: failed to create engines: pebble: malformed manifest file "MANIFEST-001086" for DB "/cockroach/cockroach-data"
Failed running "start"
```

The log indicates there is corruption in the manifest file in data directory
Data directory is where this node saves the data 
Restarting of this node will not help, as it will again be corrupted 

Will follow below steps to fix this issue 

Remove the container and remove the corrupted volume 
Recreate the container with new volume 
Node 2 and Node 3 are already healthy, so that data is already available, deletion of node 1 volume will not affect node 2 and node 3 

```
docker rm -f crdb-node1
docker volume rm crdb-node1-data
docker volume create crdb-node1-data
docker run -d \
  --name=crdb-node1 \
  --hostname=crdb-node1 \
  --network=crdb-net \
  -p 26257:26257 \
  -p 8080:8080 \
  -v crdb-node1-data:/cockroach/cockroach-data \
  cockroachdb/cockroach:latest start \
  --insecure \
  --listen-addr=0.0.0.0:26257 \
  --advertise-addr=crdb-node1:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257 \
  --cache=256MiB \
  --max-sql-memory=256MiB
```

check if node 1 is available now 

All 3 are live in admin ui 

| nodes | Uptime | Replicas | Capacity Usage | Memory Use | vCPUs | Version | Status |
|---|---|---|---|---|---|---|---|
| crdb-node2:26257 (n2) | 3 hours | 56 | 0 % | 10 % | 8 | v26.2.0 | LIVE |
| crdb-node3:26257 (n3) | 3 hours | 56 | 0 % | 13 % | 8 | v26.2.0 | LIVE |
| crdb-node1:26257 (n4) | 2 minutes | 56 | 0 % | 10 % | 8 | v26.2.0 | LIVE |

```
docker exec -it crdb-node2 ./cockroach node status --all --insecure --host=crdb-node2:26257

  id |     address      |   sql_address    |  build  |         started_at           |         updated_at           | locality | attrs | is_available | is_live | replicas_leaders | replicas_leaseholders | ranges | ranges_unavailable | ranges_underreplicated | is_decommissioning | membership | is_draining
-----+------------------+------------------+---------+------------------------------+------------------------------+----------+-------+--------------+---------+------------------+-----------------------+--------+--------------------+------------------------+--------------------+------------+--------------
   1 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-24 08:08:47.8538 UTC | 2026-05-24 08:49:29.512 UTC  |          | []    | false        | false   |                0 |                     0 |     56 |                  0 |                      0 | false              | active     | true
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-24 06:15:51.014 UTC  | 2026-05-24 08:51:37.715 UTC  |          | []    | true         | true    |                5 |                     4 |     56 |                  0 |                      0 | false              | active     | false
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-24 06:16:00.852 UTC  | 2026-05-24 08:51:38.168 UTC  |          | []    | true         | true    |               34 |                    33 |     56 |                  0 |                      0 | false              | active     | false
   4 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-24 08:46:19.674 UTC  | 2026-05-24 08:51:37.911 UTC  |          | []    | true         | true    |               15 |                    16 |     56 |                  0 |                      0 | false              | active     | false
(4 rows)
```

In both the cases the dead node still appears
it is for operational purpose, to remove it from admin ui we may need to decommision it

---

Documentation:
- Node status — https://www.cockroachlabs.com/docs/stable/cockroach-node#node-status
- Recovering the corrupted node — https://www.cockroachlabs.com/docs/stable/disaster-recovery-planning#single-region-recovery
- Dead and Decommision — https://www.cockroachlabs.com/docs/stable/node-shutdown#node-shutdown-sequence

Runbook Steps :
1. check the status of the nodes with --all option to get complete status, this provides if the node is live, is draining, is decomissioning 
2. If the node appears dead, check the node logs
3. The next action should be based on the error of the log 
     The log for this break indicated the data directory is corrupted and it is unable to start, other reasons colud be wrong volume mounted from a different cluster, reusing a volume after a full cluster is down 
4. To fix it based on the logs 
     1. Remove the container 
     2. wipe the corrupted volume (ensure node 2 and 3 are healthy)
     3. create a volume
     4. Restart the container with correct volume     

---

### Hints

None requested.

---

### Score

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BF01 — Node Fails to Start — Bad Data Directory
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Triage Approach        2/2
Tools Used             2/2
Documentation          1/2
Root Cause Accuracy    2/2
Runbook Quality        1/2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                  8/10
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Triage was clean and methodical — status check first, then logs, then fix, no random detours. Root cause was exact: the Pebble MANIFEST file was malformed, the store couldn't open, and you correctly understood that a restart would not help. The documentation pages found were relevant but the disaster recovery planning page is broad; the more specific target was the "Replace or add a node" or "Decommission a node" docs, which directly describe the wipe-and-rejoin procedure. The runbook has the right steps but is missing the actual `docker run` command — another engineer reading it would know what to do but not the exact syntax to do it, and there's no explicit final verification step with the command to confirm all nodes are live.
