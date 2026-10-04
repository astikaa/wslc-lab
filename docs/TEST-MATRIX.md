# WSLC Test Matrix

Test environment: WSL 3.0.1.0 / Kernel 6.18.40.1-1 / WSLC 3.0.1.0

| Capability | Result |
|---|---|
| WSL 2.6.1 -> 3.0.1 upgrade | PASS |
| Kernel 6.6 -> 6.18 | PASS |
| Existing Docker Desktop after upgrade | PASS |
| Existing Docker volumes | PASS |
| Existing WSL bind mounts | PASS |
| PostgreSQL 17.7 after upgrade | PASS |
| Router9 after upgrade | PASS |
| WSLC CLI | PASS |
| WSLC session manager | PASS |
| OCI image pull | PASS |
| Container execution | PASS |
| Separate WSLC/Docker image stores | OBSERVED |
| Detached container | PASS |
| WSLC bridge networking | PASS |
| Windows localhost port publishing | PASS |
| Ubuntu -> WSLC localhost | NOT REACHABLE |
| Ubuntu -> Windows gateway -> WSLC | NOT REACHABLE |
| Ubuntu -> WSLC bridge IP | NOT REACHABLE |
| Windows NTFS bind mount | PASS |
| NTFS bidirectional read/write | PASS |
| Ubuntu filesystem bind mount | PASS |
| Ubuntu filesystem bidirectional read/write | PASS |
| WSLC managed volumes | NOT TESTED |
| Custom WSLC networks | NOT TESTED |
| Container-to-container networking/DNS | NOT TESTED |
| Dockerfile build | NOT TESTED |
| GPU passthrough | NOT TESTED |
| CPU/memory limits | NOT TESTED |
| Performance | NOT TESTED |
| Docker Compose compatibility | NOT TESTED |
| Docker Engine API compatibility | NOT TESTED |

## Phase 2 — Managed Volumes and Networking

| Capability | Result |
|---|---|
| WSLC managed volume creation | PASS |
| Default `guest` volume driver | PASS |
| Volume inspect | PASS |
| Volume persistence after writer destruction | PASS |
| Cross-container volume read | PASS |
| Cross-container volume mutation | PASS |
| Explicit volume deletion lifecycle | PASS |
| Custom WSLC bridge network | PASS |
| Automatic IPv4 IPAM | PASS |
| Container-name DNS | PASS |
| Container-to-container HTTP by name | PASS |
| Container-to-container HTTP by IP | PASS |
| Network alias DNS | PASS |
| HTTP by network alias | PASS |
| Default bridge → custom-network DNS | NOT REACHABLE (tested topology) |
| Default bridge → custom-network direct IP | NOT REACHABLE / TIMEOUT (tested topology) |
| Phase 2 resource cleanup | PASS |

Detailed evidence: `docs/VALIDATION-WSLC-PHASE-2.md`
