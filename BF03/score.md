# BF03 — Cluster Won't Form — Wrong Join Flags

**Score: 7/10**

| Dimension | Score |
|---|---|
| Triage Approach | 2/2 |
| Tools Used | 1/2 |
| Documentation | 2/2 |
| Root Cause Accuracy | 1/2 |
| Runbook Quality | 1/2 |

## Rationale

**Triage Approach — 2/2**
Systematic and logical end-to-end: node status confirmed the problem, docker ps confirmed the process was up, docker logs surfaced the warning, network and config were checked, connectivity was verified before the fix was applied — no wasted steps or wrong layers.

**Tools Used — 1/2**
`docker inspect | grep join` and `docker inspect | grep adv` were the right calls and led directly to the answer; however, `docker network inspect crdb-test` uses the wrong network name (the network is `crdb-net`), and the internal CockroachDB log at `/cockroach/cockroach-data/logs/cockroach.log` was not checked — it shows the exact port 29999 being dialed on every retry and would have been faster, more specific evidence than inspecting container config.

**Documentation — 2/2**
Found the exact right page and section ("Incorrect --join address" in cluster-setup-troubleshooting) without any URL being provided by the error message — the best documentation navigation across all three sessions.

**Root Cause Accuracy — 1/2**
Correctly identified that the port in `--join` (29999) does not match the port in `--advertise-addr` (26257); the mechanism was not explained — a fresh node uses the `--join` addresses to make gRPC bootstrap RPCs to get assigned a node ID and receive cluster gossip topology, and when every address is unreachable the node stays permanently uninitialized, which is why it never appears in `node status` and rejects all SQL connections.

**Runbook Quality — 1/2**
Clear improvement over BF01 and BF02 — the full `docker run` fix command is present; step 1 still has no `docker exec` wrapper, and there is no explicit numbered verification step at the end with a command to confirm the node joined successfully.

## What would a 10 look like for this scenario

A perfect response would check the internal CockroachDB log (`docker exec crdb-node1 cat /cockroach/cockroach-data/logs/cockroach.log | grep "join rpc"`) to see the exact addresses and ports being dialed, explain that `--join` must list the `--advertise-addr` of existing cluster nodes (not the listen address) because this is the address peers announce to the cluster, and close the runbook with an explicit verification step (`docker exec crdb-node2 ./cockroach node status --insecure --host=crdb-node2:26257` confirming `is_available=true` and `is_live=true` for the rejoined node).
