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
| WSLC managed volumes | PASS |
| Custom WSLC networks | PASS |
| Container-to-container networking/DNS | PASS |
| Dockerfile build | PASS |
| GPU passthrough | PARTIAL PASS — `/dev/dxg` + WSL GPU libraries injected; hardware Vulkan not established |
| CPU/memory limits | PASS — CPU throttling and memory cgroup enforcement verified |
| Performance | NOT TESTED |
| Docker Compose compatibility | PARTIAL PASS — external Compose works against the WSLC Docker Engine; no native `wslc compose` command |
| Docker Engine API compatibility | PASS — tested subset through the internal Docker socket/API |

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

## Phase 3 — Native Dockerfile Build

| Capability | Result |
|---|---|
| Native `wslc build` | PASS |
| Dockerfile `FROM` / `RUN` | PASS |
| Local image creation and tagging | PASS |
| Run locally built image | PASS |
| Image inspect | PASS |
| BuildKit Dockerfile frontend | OBSERVED |
| Windows build context | PASS |
| `COPY` | PASS |
| `ARG` / `--build-arg` | PASS |
| `ARG` → `ENV` persistence | PASS |
| `WORKDIR` | PASS |
| Build cache reuse | PASS |
| Build-context cache invalidation | PASS |
| Previous image remains independently runnable | PASS |
| Multi-stage build | PASS |
| `COPY --from` | PASS |
| Default final-stage selection | PASS |
| `--target` named stage | PASS |
| `.dockerignore` | PASS |
| Complete Docker/BuildKit compatibility | NOT CLAIMED |

Detailed evidence: `docs/VALIDATION-WSLC-PHASE-3-BUILD.md`

## Phase 4 — Resource Limits and GPU

| Capability | Result |
|---|---|
| `--cpus` configuration | PASS |
| cgroup v2 CPU quota | PASS |
| CPU throttling under contention | PASS |
| `--memory` configuration | PASS |
| cgroup v2 `memory.max` | PASS |
| Memory pressure accounting | PASS |
| Anonymous allocation below limit | PASS |
| Swap use under memory pressure | OBSERVED |
| Independent swap ceiling | NOT ESTABLISHED |
| Default GPU isolation | PASS |
| `--gpus all` | PASS |
| `/dev/dxg` injection | PASS |
| WSL D3D12/DXCore library injection | PASS |
| Vulkan userspace initialization | PASS |
| Hardware Vulkan acceleration | NOT ESTABLISHED |
| Observed Vulkan device | llvmpipe CPU |
| NVIDIA/CUDA validation | NOT APPLICABLE |
| Phase 4 resource cleanup | PASS |

Detailed evidence: `docs/VALIDATION-WSLC-PHASE-4-RESOURCES-GPU.md`

## Phase 5 — Docker Compose Compatibility

| Capability | Result |
|---|---|
| Native `wslc compose` command | NOT AVAILABLE |
| Compose plugin in WSLC internal Docker CLI | NOT AVAILABLE |
| Internal WSLC Docker Engine | PASS |
| Internal `/var/run/docker.sock` | PASS |
| Docker Engine API access | PASS |
| External Compose client → WSLC Engine | PASS |
| Compose config parsing | PASS |
| Multi-service `compose up -d` | PASS |
| Compose-created custom network | PASS |
| Service-name DNS | PASS |
| Container-to-container HTTP | PASS |
| Compose named volume | PASS |
| Volume write/read | PASS |
| Service reconciliation after manual container removal | PASS |
| Named-volume persistence after container recreation | PASS |
| `compose down` container cleanup | PASS |
| `compose down` network cleanup | PASS |
| Named volume retained by `compose down` | PASS |
| Named-volume persistence after full stack recreation | PASS |
| `compose down -v` volume cleanup | PASS |
| Compose resources visible through Docker API | PASS |
| Compose-created containers visible in `wslc list` | NO |
| WSLC metadata marker on WSLC-created container | OBSERVED |
| Full native WSLC Compose integration | NOT ESTABLISHED |
| Complete Docker Compose compatibility | NOT CLAIMED |
| Phase 5 resource cleanup | PASS |

Detailed evidence: `docs/VALIDATION-WSLC-PHASE-5-COMPOSE.md`
