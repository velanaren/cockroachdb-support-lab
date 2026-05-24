# BF02 — Node Fails to Start — File Descriptor Limit
## Session Transcript

---

### Scenario Presented

**BF02 — Node Fails to Start — File Descriptor Limit**

```
From: Priya Nair
Subject: Database node not coming back after server reboot

Hi,

One of our three database nodes went down last night and won't come back up after
we rebooted the server. The other two nodes are running fine and the application
is still working, but this node just exits immediately every time we try to start it.

We didn't change anything in the database config. The server itself seems healthy.
Not sure where to look.

— Priya
```

Diagnose and fix the issue. Reply when ready.

---

### Vela's Response

customer says one of the 3 nodes is not coming up. It is starting and immediately crashing. Remaining both the nodes are healthy. 

checking the status of the nodes using cockroach node status --all

```
docker exec -it crdb-node2 ./cockroach node status --all --insecure --host=crdb-node2:26257

  id |     address      |   sql_address    |  build  |         started_at           |         updated_at           | locality | attrs | is_available | is_live | replicas_leaders | replicas_leaseholders | ranges | ranges_unavailable | ranges_underreplicated | is_decommissioning |  membership    | is_draining
-----+------------------+------------------+---------+------------------------------+------------------------------+----------+-------+--------------+---------+------------------+-----------------------+--------+--------------------+------------------------+--------------------+----------------+--------------
   1 | NULL             | NULL             | NULL    | NULL                         | 2026-05-24 12:27:38.575 UTC  | NULL     | NULL  | false        | false   | NULL             | NULL                  | NULL   | NULL               | NULL                   | true               | decommissioned | true
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-24 06:15:51.014 UTC  | 2026-05-24 12:31:10.807 UTC  |          | []    | true         | true    |               29 |                    29 |     57 |                  0 |                     29 | false              | active         | false
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-24 06:16:00.852 UTC  | 2026-05-24 12:31:11.631 UTC  |          | []    | true         | true    |               28 |                    28 |     57 |                  0 |                     28 | false              | active         | false
   4 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-24 08:46:19.674 UTC  | 2026-05-24 12:27:38.576 UTC  |          | []    | false        | false   |                6 |                     3 |     57 |                  0 |                      0 | false              | active         | false
(4 rows)
```

Node 1 is not available and not alive

checking the Admin UI 

node 1 is appearing as dead 

| nodes | Uptime | Replicas | Capacity Usage | Memory Use | vCPUs | Version | Status |
|---|---|---|---|---|---|---|---|
| crdb-node2:26257 (n2) | 7 hours | 57 | 0 % | 12 % | 8 | v26.2.0 | LIVE |
| crdb-node3:26257 (n3) | 7 hours | 57 | 0 % | 12 % | 8 | v26.2.0 | LIVE |
| crdb-node1:26257 (n4) | 4 hours (stale) | 57 (stale) | 1 % (stale) | 11 % (stale) | 8 | v26.2.0 | DEAD |

checking node 1 logs indicates below error 

```
docker logs crdb-node1

initiating hard shutdown of server
too early to drain; used hard shutdown instead
*
* ERROR: ERROR: server startup failed: failed to start server: failed to create engines: hard open file descriptor limit of 50 is under the minimum required 1956
* please see https://www.cockroachlabs.com/docs/v26.2/recommended-production-settings.html for more details
*
ERROR: server startup failed: failed to start server: failed to create engines: hard open file descriptor limit of 50 is under the minimum required 1956
please see https://www.cockroachlabs.com/docs/v26.2/recommended-production-settings.html for more details
Failed running "start"
```

The error says this node is configured with file desscriptor of 50, the minimum required is 1956
check how the node is configured 

```
docker inspect crdb-node1 | grep -i -A 6 ulimits

            "Ulimits": [
                {
                    "Name": "nofile",
                    "Hard": 50,
                    "Soft": 50
                }
            ],
```

the ulimit is 50, it should be minimum 1956

The documentation in the error logs indicate - CockroachDB can use a large number of open file descriptors, often more than is available by default. Therefore, note the following recommendations. Recommeded limit is 15000. For fix we will recreate node 1 with ulimit Hard and soft as 20000, stop the node1, delete node 1 and then recreate it 

```
docker stop crdb-node1
crdb-node1

docker rm crdb-node1
crdb-node1

docker run -d \
  --name=crdb-node1 \
  --hostname=crdb-node1 \
  --network=crdb-net \
  -p 26257:26257 \
  -p 8080:8080 \
  -v crdb-node1-data:/cockroach/cockroach-data \
  --ulimit nofile=20000:20000 \
  cockroachdb/cockroach:latest start \
  --insecure \
  --listen-addr=0.0.0.0:26257 \
  --advertise-addr=crdb-node1:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257 \
  --cache=256MiB \
  --max-sql-memory=256MiB
```

check the status after fix node 1 is available now 

```
docker exec -it crdb-node2 ./cockroach node status --all --insecure --host=crdb-node2:26257

  id |     address      |   sql_address    |  build  |         started_at           |         updated_at           | locality | attrs | is_available | is_live | replicas_leaders | replicas_leaseholders | ranges | ranges_unavailable | ranges_underreplicated | is_decommissioning |  membership    | is_draining
-----+------------------+------------------+---------+------------------------------+------------------------------+----------+-------+--------------+---------+------------------+-----------------------+--------+--------------------+------------------------+--------------------+----------------+--------------
   1 | NULL             | NULL             | NULL    | NULL                         | 2026-05-24 13:04:58.085 UTC  | NULL     | NULL  | false        | false   | NULL             | NULL                  | NULL   | NULL               | NULL                   | true               | decommissioned | true
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-24 06:15:51.014 UTC  | 2026-05-24 13:07:16.941 UTC  |          | []    | true         | false   |                0 |                     0 |     57 |                  0 |                      0 | false              | active         | false
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-24 06:16:00.852 UTC  | 2026-05-24 13:07:10.805 UTC  |          | []    | true         | false   |               21 |                     0 |     57 |                  0 |                      0 | false              | active         | false
   4 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-24 13:03:55.191 UTC  | 2026-05-24 13:07:19.315 UTC  |          | []    | true         | false   |               24 |                    15 |     57 |                  0 |                      0 | false              | active         | false
(4 rows)
```

Admin ui indicates all 3 are live now

| nodes | Uptime | Replicas | Capacity Usage | Memory Use | vCPUs | Version | Status |
|---|---|---|---|---|---|---|---|
| crdb-node2:26257 (n2) | 7 hours | 57 | 0 % | 12 % | 8 | v26.2.0 | LIVE |
| crdb-node3:26257 (n3) | 7 hours | 57 | 0 % | 13 % | 8 | v26.2.0 | LIVE |
| crdb-node1:26257 (n4) | 6 minutes | 57 | 0 % | 11 % | 8 | v26.2.0 | LIVE |

---

Documentation — https://www.cockroachlabs.com/docs/v26.2/recommended-production-settings.html

Runbook :
1. check the status of nodes using cockroachdb node status --all
2. check the admin UI for the status to identify the node with issue 
3. check the logs of the nodes that is not coming up - docker logs crdb-node1
4. If the log indicates, not able to start due to open file descriptor error , check the desfined descriptors - docker inspect crdb-node1 | grep -i -A 6  ulimits
5. This documentation page describes - minimum required is 1956and recommended it 15000
6. For fixing, stop the node1, remove it and start using  --ulimit nofile=20000:20000
7. once done run cockroachdb node status and check in Admin Ui to see if the node is up

---

### Hints

None requested.

---

### Score

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BF02 — Node Fails to Start — File Descriptor Limit
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

Triage was clean and went one step further than BF01 — after reading the error in the logs, `docker inspect` was used to confirm the actual container configuration, which is the right instinct (never trust just the error message; verify the config that caused it). Documentation was applied correctly: the right page, the right numbers, and the fix exceeded the minimum without being arbitrary. Root cause was exact. The runbook drops a point on the same gap as BF01: step 1 has no `docker exec` command, step 6 mentions the `--ulimit` flag but omits the full `docker run` command, and step 7's verification has no command. A runbook without executable commands in the critical fix step requires the next engineer to reconstruct the `docker run` from memory.
