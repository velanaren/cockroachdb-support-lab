# BF04 — Client Cannot Connect — Connection Refused

**Score: 8/10**

| Dimension | Score |
|---|---|
| Triage Approach | 2/2 |
| Tools Used | 2/2 |
| Documentation | 1/2 |
| Root Cause Accuracy | 1/2 |
| Runbook Quality | 2/2 |

## Rationale

**Triage Approach — 2/2**
Systematic end-to-end with no wasted steps: node status confirmed cluster health, Admin UI corroborated, DBeaver reproduced the error, listen-addr checked in logs, refused errors found in logs and analyzed at the IP level, docker ps confirmed the missing port mapping, fix applied and verified.

**Tools Used — 2/2**
CockroachDB internal logs were used to get IP-level evidence (DNS resolved, packet arrived, port rejected on 172.26.0.2:26257), and docker ps was the exact right tool to confirm the missing port mapping — both tools used correctly and output interpreted accurately.

**Documentation — 1/2**
Found the correct page (cluster-setup-troubleshooting) and the right section (client connection issues) without a URL being provided, but the citation was bare — no description of what the doc said, and the fix came entirely from the engineer's own analysis rather than from anything in the documentation.

**Root Cause Accuracy — 1/2**
Correctly identified the missing `-p 26257:26257` port mapping as the cause, but the mechanism was not explained — Docker publishes ports by creating iptables NAT rules that forward traffic from the host interface to the container; without that rule, the host kernel has nothing listening on port 26257 and returns connection refused immediately.

**Runbook Quality — 2/2**
First session to earn 2/2 on this dimension: every critical step has a specific executable command, the full docker run fix command is present, and the runbook closes with four explicit verification steps (docker ps, node status, SQL shell, DBeaver).

## What would a 10 look like for this scenario

A perfect response would add `docker inspect crdb-node1 | grep -i port` alongside `docker ps` to show the published port configuration directly from the container spec, explain the Docker port-publishing mechanism (iptables NAT rules created by the Docker daemon when `-p` is used), and reference the specific line in the cluster-setup-troubleshooting doc that states "ensure port 26257 is not blocked by a firewall or missing from the container's port bindings" — applying the doc content explicitly rather than citing it as a trailing footnote.
