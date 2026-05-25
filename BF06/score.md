# BF06 — Client Cannot Connect — TLS Certificate Error

**Score: 9/10**

| Dimension | Score |
|---|---|
| Triage Approach | 2/2 |
| Tools Used | 2/2 |
| Documentation | 2/2 |
| Root Cause Accuracy | 1/2 |
| Runbook Quality | 2/2 |

## Rationale

**Triage Approach — 2/2**
Systematic and correct: node status confirmed node1 as dead, Admin UI corroborated, CockroachDB internal logs surfaced the TLS handshake error, targeted grep confirmed the secure/insecure mismatch across all three nodes, fix applied and verified — no wasted steps.

**Tools Used — 2/2**
Grepping the internal log for `insecure|secure|certs` across all three nodes is the precise tool to compare startup configuration, and `cat cockroach.log | tail -20` surfaced the exact TLS handshake error — both tools used correctly and output interpreted accurately.

**Documentation — 2/2**
Found the correct and specific page (transport-layer-security.html, TLS between CockroachDB nodes section) without a URL being provided, and the key understanding from that doc — that all nodes in a CockroachDB cluster must share the same security mode — was applied directly to diagnose the mismatch and inform the fix.

**Root Cause Accuracy — 1/2**
Correctly identified the secure/insecure mismatch as the cause and correctly noted the fix ("in production make all nodes secure"), but the mechanism was not explained — CockroachDB uses mutual TLS (mTLS) for all inter-node gRPC; when node1 (secure) sends a TLS ClientHello to node2 (insecure), node2 responds with a plaintext gRPC frame, producing "first record does not look like a TLS handshake," which breaks gossip and raft heartbeats and causes node1 to appear dead to the cluster.

**Runbook Quality — 2/2**
Targeted log commands at each diagnostic step, the full docker run fix command is present, the security mode matching rule is explicitly stated, and the runbook closes with node status and DBeaver verification.

## What would a 10 look like for this scenario

A perfect response would explain the mTLS mechanism — CockroachDB nodes use mutual TLS for all inter-node gRPC traffic, meaning both the sending and receiving node must present valid certificates signed by the same CA; a secure node rejects any peer that doesn't initiate a TLS handshake, which is exactly what the "first record does not look like a TLS handshake" error describes — and would add `docker inspect crdb-node1 --format '{{range .HostConfig.Binds}}{{.}} {{end}}'` to show the cert volume mount alongside the log grep, confirming both how the node is configured and what certs it is using.
