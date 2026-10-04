# WSLC Phase 2 Validation — Managed Volumes and Custom Networking

**Test date:** 2026-10-04
**Environment:** WSL 3.0.1.0 / Kernel 6.18.40.1-1 / WSLC 3.0.1.0

## Purpose

Record reproducible empirical validation of WSLC managed named volumes and custom bridge networking. Tests used disposable WSLC resources and did not modify existing Docker Desktop workloads.

## Phase 2A — Managed Volume (`guest`)

### Discovery

WSLC exposes `volume create`, `remove`, `inspect`, `list`, and `prune`. `volume create --help` showed at least two drivers: `guest` (default) and `vhd`.

Initial state:

```text
DRIVER   VOLUME NAME
```

### Create and inspect

```cmd
"C:\Program Files\WSL\wslc.exe" volume create wslc-lab-vol-persistence
"C:\Program Files\WSL\wslc.exe" volume list
"C:\Program Files\WSL\wslc.exe" volume inspect wslc-lab-vol-persistence
```

Observed:

```text
DRIVER   VOLUME NAME
guest    wslc-lab-vol-persistence
```

Important inspect fields:

```text
Driver:     guest
Mountpoint: /var/lib/docker/volumes/wslc-lab-vol-persistence/_data
Scope:      local
```

The mountpoint above is the path reported by WSLC. This test does not infer the physical storage backend from that path.

### Persistence after writer destruction

Writer:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-vol-writer -v wslc-lab-vol-persistence:/data alpine sh -c "echo phase2-writer > /data/marker.txt && ls -la /data && cat /data/marker.txt"
```

Observed:

```text
phase2-writer
```

The writer used `--rm` and disappeared after execution.

Independent reader:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-vol-reader -v wslc-lab-vol-persistence:/data alpine cat /data/marker.txt
```

Observed:

```text
phase2-writer
```

**Result: PASS.** The named volume survived destruction of the writer container.

### Cross-container mutation

Mutator:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-vol-mutator -v wslc-lab-vol-persistence:/data alpine sh -c "echo phase2-mutated > /data/marker.txt && echo second-file > /data/second.txt && ls -la /data"
```

Independent verifier:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-vol-verifier -v wslc-lab-vol-persistence:/data alpine sh -c "printf 'marker=' && cat /data/marker.txt && printf 'second=' && cat /data/second.txt"
```

Observed:

```text
marker=phase2-mutated
second=second-file
```

**Result: PASS.**

### Volume lifecycle cleanup

After all disposable volume-test containers exited, the volume still existed. It was then explicitly removed:

```cmd
"C:\Program Files\WSL\wslc.exe" volume remove wslc-lab-vol-persistence
"C:\Program Files\WSL\wslc.exe" volume list
```

Final state:

```text
DRIVER   VOLUME NAME
```

**Result: PASS.**

### Phase 2A conclusion

The tested `guest` named-volume lifecycle works:

```text
create
-> inspect
-> write from container A
-> destroy A
-> read from container B
-> mutate from another container
-> verify from another container
-> containers disappear
-> volume persists
-> explicit volume removal
```

The `vhd` driver was discovered but not tested.

## Phase 2B — Custom Bridge Networking

### Discovery

`network create --help` exposed bridge networking plus options for subnet, gateway, IP range, internal networks, labels, and driver options. `network connect --help` exposed static IPv4 assignment and network aliases.

Initial built-in networks:

```text
bridge
host
none
```

### Create custom bridge

```cmd
"C:\Program Files\WSL\wslc.exe" network create wslc-lab-net
"C:\Program Files\WSL\wslc.exe" network inspect wslc-lab-net
```

Observed:

```text
Driver:   bridge
Subnet:   172.18.0.0/16
Gateway:  172.18.0.1
Scope:    local
IPv4:     enabled
IPv6:     disabled
Internal: false
```

### Attach nginx server

```cmd
"C:\Program Files\WSL\wslc.exe" run -d --name wslc-lab-net-server --network wslc-lab-net nginx:alpine
```

No host port was published.

Inspect showed:

```text
Network:   wslc-lab-net
Gateway:   172.18.0.1
IPAddress: 172.18.0.2
Prefix:    /16
```

### Container-name DNS

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-net-client --network wslc-lab-net alpine getent hosts wslc-lab-net-server
```

Observed:

```text
172.18.0.2        wslc-lab-net-server  wslc-lab-net-server
```

**Result: PASS.**

### HTTP by container name

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-net-client --network wslc-lab-net alpine wget -qO- http://wslc-lab-net-server/
```

Observed: nginx welcome page.

**Result: PASS.** Communication occurred on the custom bridge without host port publishing.

### HTTP by direct container IP

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-net-ipclient --network wslc-lab-net alpine wget -qO- http://172.18.0.2/
```

Observed: nginx welcome page.

**Result: PASS.**

### Network alias

The running server was disconnected and reconnected with alias `web`:

```cmd
"C:\Program Files\WSL\wslc.exe" network disconnect wslc-lab-net wslc-lab-net-server
"C:\Program Files\WSL\wslc.exe" network connect --network-alias web wslc-lab-net wslc-lab-net-server
```

Inspect showed alias `web` and IP `172.18.0.2`.

Alias DNS:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --network wslc-lab-net alpine getent hosts web
```

Observed:

```text
172.18.0.2        web  web
```

HTTP by alias:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --network wslc-lab-net alpine wget -qO- http://web/
```

Observed: nginx welcome page.

**Result: PASS.**

### Observed separation from default bridge

DNS from a disposable container on the default network:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-isolation-client alpine getent hosts wslc-lab-net-server
```

Observed: no host result.

Direct-IP attempt from the default network:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm --name wslc-lab-isolation-ip alpine wget -T 3 -qO- http://172.18.0.2/
```

Observed:

```text
wget: download timed out
```

**Result: OBSERVED NOT REACHABLE UNDER TESTED TOPOLOGY.**

This result is intentionally limited to the tested topology. It is not a general claim that every WSLC bridge configuration is isolated under all conditions.

### Cleanup

```cmd
"C:\Program Files\WSL\wslc.exe" stop wslc-lab-net-server
"C:\Program Files\WSL\wslc.exe" remove wslc-lab-net-server
"C:\Program Files\WSL\wslc.exe" network remove wslc-lab-net
```

Final state:

```text
Containers:
  wslc-nginx-test

Networks:
  bridge
  host
  none

Volumes:
  none
```

The retained `wslc-nginx-test` container is the Phase 1 test resource.

**Cleanup result: PASS.**

## Phase 2 Summary

| Capability | Result |
|---|---|
| WSLC managed volume creation | PASS |
| Default `guest` volume driver | PASS |
| Volume inspect | PASS |
| Persistence after writer destruction | PASS |
| Cross-container volume read | PASS |
| Cross-container volume mutation | PASS |
| Explicit volume deletion | PASS |
| Custom bridge creation | PASS |
| Automatic IPv4 IPAM | PASS |
| Container-name DNS | PASS |
| Container-to-container HTTP by name | PASS |
| Container-to-container HTTP by IP | PASS |
| Network alias DNS | PASS |
| HTTP by network alias | PASS |
| Default bridge resolving custom-network container name | NOT REACHABLE |
| Default bridge direct-IP access to tested custom-network server | NOT REACHABLE / TIMEOUT |
| Test-resource cleanup | PASS |

## What This Establishes

For the tested WSLC 3.0.1 environment, the runtime provides working primitives for persistent named storage and multi-container application networking.

A topology conceptually similar to:

```text
application network
├── frontend
├── api
├── postgres
└── redis

api -> postgres:5432
api -> redis:6379
frontend -> api:3000
```

has the necessary basic networking primitives: custom bridge attachment, container-name resolution, aliases, and direct container-to-container connectivity.

This does **not** yet establish Docker Compose compatibility, Docker Engine API compatibility, production suitability, performance characteristics, restart behavior, backup semantics, or parity with Docker Desktop.

## Not Tested in Phase 2

- `vhd` volume driver
- volume backup/restore
- volume persistence across WSLC/Windows restart
- custom subnet/gateway configuration
- static container IP assignment
- `--internal` network behavior
- IPv6
- Dockerfile builds
- CPU/memory limits
- GPU passthrough
- performance
- Docker Compose compatibility
- Docker Engine API compatibility

## Phase 2 Status

**Managed `guest` volume lifecycle:** PASS

**Custom bridge networking and DNS:** PASS

**Observed default/custom bridge separation in tested topology:** PASS
