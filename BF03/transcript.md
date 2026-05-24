# BF03 — Cluster Won't Form — Wrong Join Flags
## Session Transcript

---

### Scenario Presented

**BF03 — Cluster Won't Form — Wrong Join Flags**

```
From: Daniel Cho
Subject: New database node stuck on startup — not joining the cluster

Hi,

We provisioned a new node to expand our CockroachDB cluster and it won't join.
The process is running and doesn't crash, but the node never shows up in our
monitoring and clients connecting to it get refused.

The other two nodes in the cluster are working fine. We've tried restarting the
new node several times but it just sits there doing nothing.

— Daniel
```

Diagnose and fix the issue. Reply when ready.

---

### Vela's Response

As per the customer, new node did not join the cluster, process is running did not crash, restarting did not work. Remaining nodes are healthy.

check node status - cockroach node status
node 1 is not available

```
docker exec -it crdb-node2 ./cockroach node status --insecure --host=crdb-node2:26257

  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              | locality | attrs | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+----------+-------+--------------+----------
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-24 06:15:51.014055 +0000 UTC | 2026-05-24 15:01:46.585646 +0000 UTC |          | []    | true         | true
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-24 06:16:00.852393 +0000 UTC | 2026-05-24 15:01:44.981072 +0000 UTC |          | []    | true         | true
   4 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-24 13:03:55.191695 +0000 UTC | 2026-05-24 14:58:59.867201 +0000 UTC |          | []    | false        | false
(3 rows)
```

As per customer node 1 is running fine and not crashing
check the docker container for node 1
node 1 is running fine

```
docker ps | grep cockroach

dd804ad971df   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   3 minutes ago    Up 3 minutes    0.0.0.0:8080->8080/tcp, 0.0.0.0:26257->26257/tcp   crdb-node1
3da528b6b213   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   9 hours ago      Up 9 hours      0.0.0.0:8082->8080/tcp, 0.0.0.0:26259->26257/tcp   crdb-node3
55ea0e237414   cockroachdb/cockroach:latest   "/cockroach/cockroac…"   9 hours ago      Up 9 hours      0.0.0.0:8081->8080/tcp, 0.0.0.0:26258->26257/tcp   crdb-node2
```

Node 1 is running but it is still not appearing as available
check the logs docker logs crdb-node1

```
docker logs crdb-node1

 WARNING: The server appears to be unable to contact the other nodes in the cluster. Please try:
*
* - starting the other nodes, if you haven't already;
* - double-checking that the '--join' and '--listen'/'--advertise' flags are set up correctly;
* - running the 'cockroach init' command if you are trying to initialize a new cluster.
```

It says the server is not able to connect other nodes in the cluster
so now the node is running but it is not able to contact other nodes in the cluster

check if all nodes are in same docker network
docker network inspect crdb-test - all 3 containers listed

Nodes will contact other nodes in the cluster based on their advertise_addr and the --join command includes the advertise-addr of other nodes that this node wants to connect
First we will check the join statement for the affected node

```
docker inspect crdb-node1 | grep "join"

            "--join=crdb-node1:29999,crdb-node2:29999,crdb-node3:29999",
                "--join=crdb-node1:29999,crdb-node2:29999,crdb-node3:29999",
```

node 1 was trying to connect to crdb-node2:29999 and it is not able to connect and the same way to crdb-node3:29999 and it is not able to connect, next step is to check what is the advertise-addr of node 2 and node 3 which they say other nodes can contact them

```
docker inspect crdb-node2 | grep "adv"

            "--advertise-addr=crdb-node2:26257",
                "--advertise-addr=crdb-node2:26257",

docker inspect crdb-node3 | grep "adv"

            "--advertise-addr=crdb-node3:26257",
                "--advertise-addr=crdb-node3:26257",

docker inspect crdb-node1 | grep "adv"

            "--advertise-addr=crdb-node1:26257",
                "--advertise-addr=crdb-node1:26257",
```

This clearly indicates the advert-addr of the nodes and the address provided in join statement did not match causing the issue

Fix is to restart node1 with correct advertise-addr of all 3 nodes in the join, but before that we will try to connect node 2 and node 3 from node 1 to see if the address resolves

```
docker exec -it crdb-node1 curl http://crdb-node2:8080/health
{

}

docker exec -it crdb-node1 curl http://crdb-node3:8080/health
{

}
```

There are no errors

Now going ahead with the fix
stop and remove node 1

```
docker stop crdb-node1
docker rm crdb-node1
```

Restart node 1 with correct join and advertise-addr

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
  --join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257 \
  --cache=256MiB \
  --max-sql-memory=256MiB
```

once the fix is done, verify the node status it should be available now and part of cluster

```
docker exec -it crdb-node2 ./cockroach node status --insecure --host=crdb-node2:26257

  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              | locality | attrs | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+----------+-------+--------------+----------
   2 | crdb-node2:26257 | crdb-node2:26257 | v26.2.0 | 2026-05-24 06:15:51.014055 +0000 UTC | 2026-05-24 16:43:23.161584 +0000 UTC |          | []    | true         | true
   3 | crdb-node3:26257 | crdb-node3:26257 | v26.2.0 | 2026-05-24 06:16:00.852393 +0000 UTC | 2026-05-24 16:43:24.373461 +0000 UTC |          | []    | true         | true
   4 | NULL             | NULL             | NULL    | NULL                                 | 2026-05-24 16:42:58.160346 +0000 UTC | NULL     | NULL  | false        | false
   5 | crdb-node1:26257 | crdb-node1:26257 | v26.2.0 | 2026-05-24 16:30:43.931299 +0000 UTC | 2026-05-24 16:43:23.145847 +0000 UTC |          | []    | true         | true
(4 rows)
```

---

Documentation — https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting#client-connection-issues
Specific section: Incorrect `--join` address

Runbook:
1. check the node status - cockraoch node status ( this will provide the status of the node, check the affected node status -it should be in is_available - false )
2. check if the process related to node 1 is running - `docker ps | grep cockroach` - check all 3 are running
     - This indicates node1 is up but it is not able to connect to other nodes
3. check why it is not able to connect to other nodes
     a) check if they are in the same docker network `docker network inspect crdb-net` - all 3 containers should be listed
     b) nodes connect to each other using the advertise-addr provided in join statement
     check the join in node 1
     `docker inspect crdb-node1 | grep "join"`
     All 3 nodes advertise-addr will be listed
4. Individually verify if the advertse-addr in join matches the advertise-addr of all 3 nodes
     `docker inspect crdb-node1 | grep "adv"`
     `docker inspect crdb-node2 | grep "adv"`
     `docker inspect crdb-node3 | grep "adv"`
5. if the results in step 4 did not match with the address of each node in join in step 3 then join is the issue
6. before running a fix try connecting to node 2 and node3 from node 1
     `docker exec -it crdb-node1 curl http://crdb-node2:8080/health`
     `docker exec -it crdb-node1 curl http://crdb-node3:8080/health`
7. If this returns no errors, node 1 is able to connect to node 2
8. Fix is to stop and remove node 1 and restart node 1 with correct advertise-addr in join
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
  --join=crdb-node1:26257,crdb-node2:26257,crdb-node3:26257 \
  --cache=256MiB \
  --max-sql-memory=256MiB
```

---

### Hints

None requested.

---

### Score

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BF03 — Cluster Won't Form — Wrong Join Flags
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Triage Approach        2/2
Tools Used             1/2
Documentation          2/2
Root Cause Accuracy    1/2
Runbook Quality        1/2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                  7/10
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Triage was systematic and logical — node status, container state, logs, network, config, connectivity, fix, with no random jumping. Tools used were appropriate and got to the right answer, but `docker network inspect crdb-test` uses the wrong network name (`crdb-net`), and the internal CockroachDB log at `/cockroach/cockroach-data/logs/cockroach.log` was not checked — it shows the exact port 29999 being dialed on every retry, which is faster evidence than going through inspect. Documentation was the best yet — found the exact right section independently with no URL handed by the error. Root cause correctly identified the port mismatch but the mechanism (gRPC bootstrap RPC, why wrong port prevents node ID assignment) was absent. Runbook is clearly better than BF01/BF02 — the full fix command is present — but step 1 has no command and there is no explicit numbered verification step at the end.
