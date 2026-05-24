# BF01 — Node Fails to Start — Bad Data Directory

**Score: 8/10**

| Dimension | Score |
|---|---|
| Triage Approach | 2/2 |
| Tools Used | 2/2 |
| Documentation | 1/2 |
| Root Cause Accuracy | 2/2 |
| Runbook Quality | 1/2 |

## Rationale

**Triage Approach — 2/2**
Started at the right layer (cluster-wide node status), moved immediately to container logs when the problem node was identified, and went directly to a fix — no random commands, no wasted steps.

**Tools Used — 2/2**
Used `node status --all` to see the full node picture including `is_draining` and `is_decommissioning` flags, then `docker logs` to read the startup error — exactly the right tools in the right order, and the output was read correctly.

**Documentation — 1/2**
The node status doc was directly applicable; the disaster recovery planning page covers the concept but is a broad overview page — the most specific target for this scenario is the "Decommission a node" or "Add or remove a node" documentation, which describes the exact wipe-and-rejoin procedure.

**Root Cause Accuracy — 2/2**
Named the exact root cause (malformed Pebble MANIFEST file preventing store initialization), correctly explained that a restart would not resolve it, and noted additional scenarios where the same pattern can appear (wrong volume, reused volume after cluster wipe).

**Runbook Quality — 1/2**
The diagnostic steps are correct and the fix steps are logically ordered, but the runbook omits the actual `docker run` command that recreates the node — another engineer would have to derive it — and there is no explicit verification step (e.g., run `node status` again and confirm `is_live=true` for the new node).

## What would a 10 look like for this scenario

A perfect response would include the full `docker run` command verbatim in the runbook so another engineer can execute it without inference, an explicit numbered verification step (`docker exec -it crdb-node2 ./cockroach node status --insecure --host=crdb-node2:26257` and confirm `is_live=true` for the new node), and a note in the runbook that the old dead node ID should be formally decommissioned (`cockroach node decommission <id>`) to clean up the cluster membership view. The documentation reference would point to the specific "Decommission a node" page rather than the general disaster recovery overview.
