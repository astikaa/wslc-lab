# WSLC 3.0.1 Phase 1 Validation Report

**Test date:** 2026-10-04
**Host:** Windows 11 / WSL
**WSL:** 3.0.1.0
**Kernel:** 6.18.40.1-1
**WSLC:** 3.0.1.0
**Ubuntu:** 24.04 / WSL2
**Docker Desktop:** 4.82.0
**Docker Engine:** 29.6.1
**Docker Compose:** 5.3.0

## 1. Purpose

This report records the first empirical validation of WSLC 3.0.1 on an existing WSL + Docker Desktop development workstation.

Objectives:

- validate the WSL 3 upgrade;
- verify existing Docker workloads after the upgrade;
- establish whether WSLC is operational;
- test basic WSLC image and container lifecycle;
- determine whether WSLC and Docker Desktop share image state;
- characterize default WSLC networking;
- test Windows and Ubuntu filesystem bind mounts;
- record failures as well as successful tests.

No existing development workload was migrated to WSLC.

## 2. Pre-upgrade Baseline

Before the upgrade:

```text
WSL version: 2.6.1.0
Kernel version: 6.6.87.2-1
WSLg version: 1.0.66
Windows version: 10.0.26200.9457
```

Distributions:

```text
NAME              STATE     VERSION
docker-desktop    Running   2
Ubuntu-24.04      Running   2
```

Docker baseline:

```text
Docker Desktop: 4.82.0
Docker Engine:  29.6.1
API version:    1.55
containerd:     2.2.5
runc:           1.3.6
Compose:        5.3.0
```

Representative existing workloads included Router9, PostgreSQL 17.7, MariaDB, and development application containers.

PostgreSQL used the Docker managed volume:

```text
docker_pgdata17 -> /var/lib/postgresql/data
```

Router9 used the WSL filesystem bind mount:

```text
/home/astika/projects/ai_gateway/data -> /app/data
```

## 3. WSL Upgrade

Executed from Windows:

```cmd
wsl --update
```

Observed:

```text
Checking for updates.
Updating Windows Subsystem for Linux to version: 3.0.1.
```

After upgrade:

```text
WSL version: 3.0.1.0
Kernel version: 6.18.40.1-1
WSLg version: 1.0.79
MSRDC version: 1.2.7214
Direct3D version: 1.611.1-81528511
DXCore version: 10.0.26100.1-240331-1435.ge-release
Windows version: 10.0.26200.9457
```

The distributions continued to report `VERSION 2`. This is distinct from the installed WSL package version.

Inside Ubuntu:

```bash
uname -a
```

Observed:

```text
Linux LAPTOP-FASAU61A 6.18.40.1-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Fri Jul 31 22:12:15 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
```

**Result: PASS**

WSL upgraded from 2.6.1.0 to 3.0.1.0 and the kernel changed from 6.6.87.2 to 6.18.40.1.

## 4. Docker Desktop Regression Check

After the WSL upgrade:

```text
Docker Engine: 29.6.1
Docker Compose: 5.3.0
```

Important containers included:

```text
router9      running
postgres17   running / healthy
pgbouncer    running / healthy
mariadb10    running
```

Restart-policy inspection showed:

```text
router9              restart=unless-stopped
postgres17           restart=always
pgbouncer            restart=always
admin_tp-app         restart=no
admin_tp-functions   restart=no
```

The `admin_tp` development containers remained exited after the WSL restart because their restart policy was `no`. Their logs also showed a prior SIGTERM during shutdown. Existing application-level errors in those logs were not attributed to the WSL upgrade.

Persistent mounts remained intact.

PostgreSQL:

```text
docker_pgdata17 -> /var/lib/postgresql/data
```

Router9:

```text
/home/astika/projects/ai_gateway/data -> /app/data
```

**Result: PASS**

No WSL 3 regression was observed in the existing Docker engine, persistent PostgreSQL volume, Router9 bind mount, or automatically restarted infrastructure containers.

## 5. WSLC Discovery

Inside Ubuntu:

```bash
wslc
```

was not available as a Linux command.

From the current Windows command environment:

```cmd
wslc version
```

was initially not found through PATH.

The executable existed at:

```text
C:\Program Files\WSL\wslc.exe
```

Direct execution:

```cmd
"C:\Program Files\WSL\wslc.exe" version
```

returned:

```text
wslc 3.0.1.0
```

**Result: PASS WITH PATH CAVEAT**

WSLC was installed and functional, but the tested Windows command session did not initially discover `wslc.exe` through PATH.

## 6. WSLC System Information

Command:

```cmd
"C:\Program Files\WSL\wslc.exe" info
```

Observed:

```text
Client:
WSL version: 3.0.1.0
Kernel version: 6.18.40.1-1
Direct3D version: 1.611.1-81528511
DXCore version: 10.0.26100.1-240331-1435.ge-release
Windows version: 10.0.26200.9457
Settings file: C:\Users\ASTIKA\AppData\Local\wslc\settings.yaml

Server:
Session manager version: 3.0.1
Sessions: 1
ID   Creator PID   Display Name
1    35368         wslc-cli-ASTIKA
```

**Result: PASS**

## 7. Basic Container Execution

Command:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm hello-world
```

WSLC pulled `hello-world:latest` and executed it successfully.

The image printed `Hello from Docker!`. This string is output by the `hello-world` image itself and does not by itself establish use of the Docker Desktop daemon.

**Result: PASS**

Validated:

- registry pull;
- image storage;
- container creation;
- container execution;
- stdout;
- automatic container removal with `--rm`.

## 8. WSLC vs Docker Desktop Image State

WSLC:

```cmd
"C:\Program Files\WSL\wslc.exe" images
```

showed:

```text
REPOSITORY    TAG      IMAGE ID       CREATED        SIZE
hello-world   latest   e2ac70e7319a   6 months ago   10.1kB
```

From Ubuntu:

```bash
docker images hello-world
```

returned no matching Docker Desktop image.

**Result: OBSERVED ISOLATION**

For this test, an image pulled through WSLC did not appear in the Docker Desktop image listing. This is evidence that the tested WSLC and Docker Desktop environments do not share this image state.

## 9. WSLC Published Port Test

Started nginx:

```cmd
"C:\Program Files\WSL\wslc.exe" run -d --name wslc-nginx-test -p 8081:80 nginx:alpine
```

WSLC reported:

```text
127.0.0.1:8081->80/tcp
```

Windows test:

```cmd
curl http://localhost:8081
```

returned the nginx welcome page.

**Result: PASS**

Windows could access the published WSLC service through localhost.

## 10. Ubuntu to WSLC Published Port

From Ubuntu:

```bash
curl -I http://localhost:8081
curl -I http://127.0.0.1:8081
```

Both failed to connect.

Ubuntu network information:

```text
default via 172.25.224.1 dev eth0 proto kernel onlink
nameserver 10.255.255.254
```

Test through the Windows-side gateway:

```bash
curl -v --connect-timeout 3 http://172.25.224.1:8081/
```

Timed out after three seconds.

Windows listener inspection:

```cmd
netstat -ano | findstr :8081
```

showed:

```text
TCP    127.0.0.1:8081    0.0.0.0:0    LISTENING    12828
```

**Result: NOT REACHABLE FROM UBUNTU IN DEFAULT TEST**

The effective published listener was bound to Windows loopback.

## 11. WSLC Default Network

Command:

```cmd
"C:\Program Files\WSL\wslc.exe" network list
```

showed:

```text
NETWORK ID     NAME      DRIVER    SCOPE
f4050e305dd9   bridge    bridge    local
6d0b881f2455   host      host      local
4c55cac583a0   none      null      local
```

Container inspection showed:

```text
NetworkMode: bridge
Gateway:     172.17.0.1
IPAddress:   172.17.0.2
Prefix:      /16
MAC:         02:42:ac:11:00:02
```

Published port:

```text
Container: 80/tcp
Host IP:   127.0.0.1
Host Port: 8081
```

Bridge inspection showed:

```text
Subnet:   172.17.0.0/16
Gateway:  172.17.0.1
IPv4:     enabled
IPv6:     disabled
MTU:      1500
```

Observed bridge options included:

```text
com.docker.network.bridge.default_bridge=true
com.docker.network.bridge.enable_icc=true
com.docker.network.bridge.enable_ip_masquerade=true
com.docker.network.bridge.host_binding_ipv4=0.0.0.0
com.docker.network.bridge.name=docker0
```

The effective container port binding nevertheless reported `HostIp: 127.0.0.1`, and Windows confirmed the listener on `127.0.0.1`.

## 12. Ubuntu to WSLC Container IP

From Ubuntu:

```bash
ip route get 172.17.0.2
```

returned:

```text
172.17.0.2 via 172.25.224.1 dev eth0 src 172.25.231.217 uid 1000
```

Direct request:

```bash
curl -v --connect-timeout 3 http://172.17.0.2/
```

Timed out.

**Result: NOT REACHABLE FROM UBUNTU IN DEFAULT TEST**

Ubuntu did not have a direct local route to the WSLC bridge and could not reach the tested container through the resolved gateway path.

## 13. Windows NTFS Bind Mount

Created on Windows:

```cmd
mkdir C:\wslc-test
echo hello-from-windows> C:\wslc-test\windows.txt
```

Mounted into an Alpine container:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm -v "C:\wslc-test:/data" alpine ls -la /data
```

The file was visible.

Read test:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm -v "C:\wslc-test:/data" alpine cat /data/windows.txt
```

returned:

```text
hello-from-windows
```

Write test:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm -v "C:\wslc-test:/data" alpine sh -c "echo hello-from-wslc > /data/container.txt"
```

Windows:

```cmd
type C:\wslc-test\container.txt
```

returned:

```text
hello-from-wslc
```

**Result: PASS**

Bidirectional read/write against a Windows NTFS bind mount worked.

## 14. Ubuntu Native Filesystem Bind Mount

Created inside Ubuntu:

```bash
mkdir -p ~/wslc-test
echo "hello-from-ubuntu" > ~/wslc-test/ubuntu.txt
```

Windows could read it through:

```cmd
type "\\wsl.localhost\Ubuntu-24.04\home\astika\wslc-test\ubuntu.txt"
```

returned:

```text
hello-from-ubuntu
```

WSLC bind mount:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm -v "\\wsl.localhost\Ubuntu-24.04\home\astika\wslc-test:/data" alpine ls -la /data
```

showed `ubuntu.txt`.

WSLC write test:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm -v "\\wsl.localhost\Ubuntu-24.04\home\astika\wslc-test:/data" alpine sh -c "echo hello-from-wslc > /data/wslc.txt"
```

Ubuntu subsequently read:

```bash
cat ~/wslc-test/wslc.txt
```

and returned:

```text
hello-from-wslc
```

**Result: PASS**

WSLC successfully bind mounted the Ubuntu native filesystem through the `\\wsl.localhost` UNC path with bidirectional read/write.

## 15. Current Validation Matrix

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
| Separate WSLC/Docker image state | OBSERVED |
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

## 16. Observed Topology

```text
Windows 11
|
+-- WSL 3.0.1
|   `-- Linux kernel 6.18.40.1
|
+-- Ubuntu-24.04 / WSL2
|   `-- /home/astika/projects/...
|
+-- WSLC 3.0.1
|   +-- session manager
|   +-- observed independent image state
|   +-- bridge 172.17.0.0/16
|   +-- Windows localhost port publishing
|   +-- Windows NTFS bind mounts
|   `-- Ubuntu filesystem bind mounts through \\wsl.localhost
|
`-- Docker Desktop 4.82.0
    +-- Docker Engine 29.6.1
    +-- Docker Compose 5.3.0
    +-- Router9
    +-- PostgreSQL 17.7
    +-- MariaDB
    `-- development workloads
```

WSLC and Docker Desktop coexisted during the tests. No existing Docker workload was migrated to WSLC during Phase 1.

## 17. Phase 1 Conclusions

The tests establish that WSLC 3.0.1 is operational and supports more than basic container execution.

Empirically validated capabilities include:

- OCI image pull;
- container lifecycle;
- detached execution;
- bridge networking;
- Windows localhost port publishing;
- Windows NTFS bind mounts;
- Ubuntu WSL filesystem bind mounts;
- bidirectional filesystem read/write.

The tests also identified an important default network boundary:

```text
Windows localhost -> WSLC published service     PASS
Ubuntu localhost  -> WSLC published service     NOT REACHABLE
Ubuntu gateway    -> WSLC published service     NOT REACHABLE
Ubuntu            -> WSLC bridge container IP   NOT REACHABLE
```

WSLC should therefore not yet be considered a drop-in replacement for the existing Docker Desktop development environment.

Major unanswered areas include:

1. managed volumes;
2. custom networks;
3. container-to-container DNS/networking;
4. Dockerfile builds;
5. GPU passthrough;
6. CPU and memory controls;
7. filesystem and runtime performance;
8. Docker Compose compatibility or equivalent orchestration;
9. Docker Engine API/tool compatibility.

Existing Docker workloads should remain unchanged until those areas are evaluated.

## 18. Phase 2

Planned validation:

```text
managed volumes
      |
custom network
      |
container-to-container networking + DNS
      |
Dockerfile build
      |
env / env-file
      |
health checks
      |
CPU / memory limits
      |
GPU
      |
filesystem performance
      |
Compose / API investigation
      |
isolated real-world application PoC
```

Any real-world PoC must use disposable or copied state and must not allow Docker Desktop and WSLC workloads to write concurrently to the same persistent application data.

## Status

**WSL 3.0.1 upgrade:** PASS
**Docker Desktop coexistence:** PASS
**WSLC Phase 1 basic validation:** PASS
**WSLC as Docker Desktop replacement:** NOT ESTABLISHED
