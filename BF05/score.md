# BF05 — Wrong Connection String / Wrong Port

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
Systematic and correct: confirmed cluster health, reproduced the client error via DBeaver, ruled out listen-addr misconfiguration, identified the port mismatch via docker ps, updated the connection string, and verified — no detours or wrong layers.

**Tools Used — 2/2**
docker ps was the exact right tool to surface the host port mapping discrepancy, and the CockroachDB internal log grep for listen-addr was used to correctly rule out a server-side configuration issue before pivoting to the container layer.

**Documentation — 1/2**
Two correct pages cited — connection-parameters.html is specifically the right reference for connection string format and port configuration, and it was found without a URL being provided — but neither doc's content was described or shown to have informed the fix; the diagnosis and fix came entirely from reading docker ps output.

**Root Cause Accuracy — 1/2**
Correctly identified that the host port mapping changed from 26257 to 26260 during maintenance and the client connection string was not updated to match, with a clear rule articulated; the mechanism was not explained — when a Docker container is stopped and recreated, its `-p` bindings are not preserved from the prior run, so the new `docker run` command established a different host port, silently breaking any client that still dialed the old one.

**Runbook Quality — 2/2**
Second consecutive 2/2: specific commands at every critical step, the port mapping rule is formalized with concrete examples (26260:26257 → dial 26260), and explicit verification at the end with node status command.

## What would a 10 look like for this scenario

A perfect response would add `docker inspect crdb-node1 --format '{{json .HostConfig.PortBindings}}'` to show the exact port binding from the container spec alongside docker ps, explain that Docker port bindings are set at container creation time and are not preserved across stop/rm/recreate cycles, and reference the specific parameter format from connection-parameters.html (e.g., `postgresql://root@localhost:26260/defaultdb?sslmode=disable`) to show the corrected connection URL — applying the doc directly rather than citing it as a footnote.
