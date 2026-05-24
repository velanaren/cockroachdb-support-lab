# BF02 — Node Fails to Start — File Descriptor Limit

**Score: 9/10**

| Dimension | Score |
|---|---|
| Triage Approach | 2/2 |
| Tools Used | 2/2 |
| Documentation | 2/2 |
| Root Cause Accuracy | 2/2 |
| Runbook Quality | 1/2 |

## Rationale

**Triage Approach — 2/2**
Started at cluster status, moved to logs, then went one step further to confirm the container configuration with `docker inspect` — the right instinct is to verify the config, not just accept the error message.

**Tools Used — 2/2**
Used `node status --all`, `docker logs`, and `docker inspect | grep ulimits` — exactly the right tools in the right order; the inspect output was read correctly and the `Hard: 50` value was immediately identified as the cause.

**Documentation — 2/2**
Found the exact page referenced in the error message, read the correct values (minimum 1956, recommended 15000), and applied them correctly in the fix by setting 20000.

**Root Cause Accuracy — 2/2**
Named the exact cause: the container was created with `--ulimit nofile=50:50`, which is below CockroachDB's hard minimum of 1956 file descriptors; confirmed via `docker inspect` rather than relying on the error message alone.

**Runbook Quality — 1/2**
Steps 3 and 4 have specific commands and are executable; step 1 has no `docker exec` wrapper, step 6 mentions the `--ulimit` flag but omits the full `docker run` command, and step 7's verification mentions checking the UI but provides no command — another engineer would have to reconstruct the recreate command from scratch.

## What would a 10 look like for this scenario

A perfect response would have the full `docker run` command (with `--ulimit nofile=20000:20000`) verbatim in step 6 of the runbook, the exact `docker exec ... node status` command in steps 1 and 7, and a note explaining that this setting must also be verified in the underlying host's `/etc/security/limits.conf` or systemd unit file for non-Docker deployments — because the same error can occur on bare-metal when the OS ulimit is too low, and the fix path is different.
