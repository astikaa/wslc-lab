# WSLC Validation — Phase 5: Docker Compose Compatibility

**Test date:** 2026-10-04
**Host:** Windows 11 / WSL
**WSL:** 3.0.1.0
**Kernel:** 6.18.40.1-1
**WSLC:** 3.0.1.0
**Docker Desktop:** 4.82.0
**Docker Desktop Engine:** 29.6.1
**WSLC internal Docker Engine:** 25.0.3
**WSLC internal Docker API:** 1.44
**External Compose used for validation:** Docker Compose v5.5.1 (`docker:cli`)

## 1. Objective

Phase 5 evaluates Docker Compose compatibility around WSLC 3.0.1.

The goals were to determine:

1. whether WSLC exposes a native Compose command;
2. whether the Docker engine used by WSLC exposes a usable Docker API;
3. whether an external Docker Compose client can orchestrate that engine;
4. whether normal Compose networking, DNS, volume, reconciliation, and teardown semantics work;
5. how resources created directly through the Docker API relate to the resource view exposed by `wslc.exe`.

This phase intentionally distinguishes **Compose workload compatibility** from **native WSLC Compose integration**.

## 2. Executive Result

**Overall result: PARTIAL PASS.**

WSLC 3.0.1 does **not** expose a native `wslc compose` command, and the Docker CLI present inside the WSLC session does not include the Compose plugin.

However, the WSLC session contains a real Docker Engine with a Unix socket at `/var/run/docker.sock`. An external Docker Compose v5.5.1 client was successfully connected to that socket and performed a complete multi-service lifecycle against the WSLC engine.

Validated behavior includes:

- Compose configuration parsing;
- creation and startup of multiple services;
- Compose-managed bridge networking;
- service-name DNS;
- container-to-container HTTP;
- named-volume creation and use;
- service reconciliation after manual container deletion;
- named-volume persistence across container recreation;
- named-volume persistence across full `compose down`;
- recreation of the full stack while retaining volume data;
- `compose down -v` destructive cleanup.

A significant integration boundary was also observed: containers created directly through Docker API/Compose were visible through the internal Docker daemon but were not shown by `wslc list`. Containers created through the WSLC CLI carried Microsoft-specific metadata absent from the Compose-created containers.

The tests establish substantial Docker Compose workload compatibility through the WSLC engine API, but they do **not** establish complete native Compose integration into the WSLC CLI surface.

## 3. Native WSLC Compose Command

The native CLI was queried directly:

```text
C:\wslc-lab-build>"C:\Program Files\WSL\wslc.exe" compose --help
Unrecognized command: 'compose'
```

The top-level help listed commands including `container`, `image`, `network`, `volume`, `build`, `run`, `exec`, and others, but no Compose command.

**Result: native `wslc compose` — NOT AVAILABLE.**

For comparison, Docker Desktop on the Windows host reported:

```text
Docker Compose version v5.3.0
```

That host Compose installation belongs to the Docker Desktop control path and was not treated as evidence of WSLC-native Compose support.

## 4. Docker Desktop and WSLC Are Distinct Control Paths

Windows Docker contexts showed Docker Desktop as the active Docker CLI context:

```text
NAME              DESCRIPTION                               DOCKER ENDPOINT
...
desktop-linux *   Docker Desktop                            npipe:////./pipe/dockerDesktopLinuxEngine
```

Inside Ubuntu-24.04, `docker version` also identified the server as Docker Desktop 4.82.0 / Engine 29.6.1.

Separately, `wslc system session run` exposed the environment used by WSLC itself.

The WSLC session reported:

```text
NAME="Microsoft Azure Linux"
VERSION="3.0.20260616"
ID=azurelinux
```

and contained its own Docker socket:

```text
/run/docker.sock
/var/run/docker.sock
```

The session also contained active Docker/containerd processes:

```text
/usr/bin/containerd ...
/usr/bin/dockerd --containerd /run/containerd/containerd.sock
```

This establishes that the WSLC test path used an internal Docker engine rather than merely forwarding all operations to the active Docker Desktop context.

## 5. WSLC Internal Docker Engine

The Docker CLI inside the WSLC session reported:

```text
Client:
 Version:           25.0.7
 API version:       1.44

Server:
 Engine:
  Version:          25.0.3
  API version:      1.44
```

Direct HTTP access to the Unix socket succeeded:

```text
curl --unix-socket /var/run/docker.sock http://localhost/_ping
OK
```

The `/version` endpoint identified:

```text
Engine 25.0.3
ApiVersion 1.44
KernelVersion 6.18.40.1-microsoft-standard-WSL2
Os linux
Arch amd64
```

**Result: WSLC internal Docker API — PASS for tested API access.**

This does not claim complete Docker Engine API compatibility across every endpoint or API feature.

## 6. Compose Plugin Availability Inside the WSLC Session

The Docker CLI shipped in the session was tested directly:

```text
docker compose version
```

Result:

```text
docker: 'compose' is not a docker command.
See 'docker --help'
```

Plugin discovery showed Buildx but no Compose plugin:

```text
/root/.docker/buildx/...
/usr/libexec/docker/cli-plugins/docker-buildx
```

**Result: bundled Compose plugin — NOT AVAILABLE.**

## 7. External Compose Client

A `docker:cli` container was tested as an external Compose client.

Without the WSLC socket mounted, its default environment attempted to connect to `tcp://docker:2375`, which failed as expected.

The client itself reported:

```text
Docker Compose version v5.5.1
```

It was then explicitly connected to the WSLC engine using:

```text
-v /var/run/docker.sock:/var/run/docker.sock
-e DOCKER_HOST=unix:///var/run/docker.sock
```

With that configuration, `docker version` showed:

```text
Client:
 Version:           29.8.2
 API version:       1.44 (downgraded from 1.56)

Server:
 Engine:
  Version:          25.0.3
  API version:      1.44
```

This is useful evidence of successful client/server API negotiation against the WSLC daemon.

`docker compose ls` also succeeded against the engine.

**Result: external Compose client → WSLC Docker API — PASS.**

## 8. Important Socket-Mount Boundary

A direct command through `wslc.exe` attempted:

```text
wslc.exe run --rm -v /var/run/docker.sock:/var/run/docker.sock alpine ...
```

Inside that container, `/var/run/docker.sock` became a directory rather than the session's Unix socket.

Inspection showed the source interpreted as:

```text
Source: C:\var\run\docker.sock
Type: bind
```

This demonstrates an important path-semantics boundary: a bind path supplied to `wslc.exe` from Windows is interpreted through the Windows-side CLI path model.

In contrast, invoking `docker run` **inside the WSLC system session** correctly mounted the Unix socket:

```text
srw-rw---- ... /var/run/docker.sock
```

and a nested container successfully called:

```text
curl --unix-socket /var/run/docker.sock http://localhost/_ping
OK
```

Therefore, Phase 5 used `wslc system session run ... docker run ...` for the Compose client rather than attempting to bind the socket directly through Windows-side `wslc run`.

## 9. Compose Test Definition

A Compose file was created inside the WSLC session at:

```text
/tmp/wslc-phase5-compose.yaml
```

Content:

```yaml
services:
  web:
    image: nginx:alpine
    networks:
      - labnet

  client:
    image: alpine
    command: ["sleep", "3600"]
    networks:
      - labnet
    volumes:
      - labdata:/data

networks:
  labnet:

volumes:
  labdata:
```

The definition intentionally exercised:

- multiple services;
- custom Compose networking;
- service-name DNS;
- a named volume;
- long-running service lifecycle.

## 10. Compose Configuration Parsing

The external Compose client was run with the WSLC Docker socket and Compose file mounted read-only.

`docker compose ... config` succeeded and normalized the configuration to include:

```text
services:
  client:
    ...
    networks:
      labnet: null
    volumes:
      - type: volume
        source: labdata
        target: /data

  web:
    ...
    networks:
      labnet: null

networks:
  labnet:
    name: work_labnet

volumes:
  labdata:
    name: work_labdata
```

**Result: Compose configuration parsing — PASS.**

## 11. Compose `up -d`

The stack was started through the external Compose client.

Compose reported creation of:

```text
Volume work_labdata
Network work_labnet
Container work-web-1
Container work-client-1
```

Both containers were started successfully.

The output occasionally printed duplicate `Creating` / `Created` lines for network and volume operations. The resulting Docker resources themselves were not duplicated, and subsequent lifecycle behavior was correct.

This duplicated progress output is recorded as an **observed output behavior**, not as a functional failure.

**Result: Compose multi-service creation/start — PASS.**

## 12. Docker Daemon View vs `wslc list`

After Compose successfully started the stack, Windows-side:

```text
wslc.exe list
```

continued to show only the pre-existing WSLC-created container:

```text
wslc-nginx-test
```

The Compose containers were absent.

However, querying the Docker daemon inside the same WSLC session showed:

```text
work-web-1       nginx:alpine   Up
work-client-1    alpine         Up
wslc-nginx-test  nginx:alpine   Up
```

The pre-existing `wslc-nginx-test` container had the same container ID through both views:

```text
89fb2ba84f3bc6e23a6476cab965e67af1ef7753f24d8168dc7fd363b5772d81
```

This is strong evidence that the discrepancy is not simply explained by the commands targeting completely unrelated daemons.

Instead, an integration/visibility boundary exists between direct Docker API resources and the resource view exposed by `wslc list`.

## 13. Metadata Difference

The WSLC-created nginx container carried:

```text
com.microsoft.wsl.container.metadata
```

Example:

```json
{
  "com.microsoft.wsl.container.metadata": "{\"V1\":{\"Flags\":0,\"InitProcessFlags\":0,\"Ports\":[{\"BindingAddress\":\"127.0.0.1\",\"ContainerPort\":80,\"Family\":2,\"HostPort\":8081,\"Protocol\":6,\"VmPort\":20002}],\"Volumes\":[]}}",
  "maintainer": "NGINX Docker Maintainers <docker-maint@nginx.com>"
}
```

The Compose-created containers instead carried normal Compose labels such as:

```text
com.docker.compose.project=work
com.docker.compose.service=web
com.docker.compose.version=5.5.1
```

and did not contain `com.microsoft.wsl.container.metadata`.

### Interpretation

The Microsoft metadata label is a strong observed discriminator between the WSLC-created and direct-API-created containers in this test.

However, this phase does **not** claim that the label is definitively the sole implementation mechanism behind `wslc list` filtering. Proving that would require additional controlled manipulation or implementation evidence.

Safe conclusion:

> Docker API-created resources can exist in the WSLC engine while remaining outside the resource view presented by `wslc list`; WSLC-created containers carry Microsoft-specific metadata that the tested Compose-created containers do not.

## 14. Compose Network Creation

Docker network enumeration after `compose up` showed:

```text
work_labnet   bridge   local
```

Inspection of `work-web-1` reported:

```text
NETWORK=work_labnet
```

**Result: Compose custom bridge network — PASS.**

## 15. Service-Name DNS and HTTP

From the Compose `client` service:

```text
docker exec work-client-1 wget -qO- http://web
```

returned the nginx default page, including:

```text
<title>Welcome to nginx!</title>
<h1>Welcome to nginx!</h1>
```

This proves that the Compose-created workload successfully used:

1. the custom Compose network;
2. service-name DNS for `web`;
3. container-to-container TCP/HTTP connectivity.

**Result: Compose service-name DNS — PASS.**
**Result: Compose inter-service HTTP — PASS.**

## 16. Named Volume Write

The client service wrote a proof file to its Compose named volume:

```text
echo phase5-compose-volume > /data/proof.txt
cat /data/proof.txt
```

Output:

```text
phase5-compose-volume
```

**Result: Compose named-volume mount/read/write — PASS.**

## 17. Reconciliation After Manual Container Deletion

`work-client-1` was manually destroyed outside Compose:

```text
docker rm -f work-client-1
```

The named volume remained present:

```text
NAME=work_labdata DRIVER=local
```

Running `docker compose up -d` again produced:

```text
Container work-web-1 Running
Container work-client-1 Creating
Container work-client-1 Created
Container work-client-1 Starting
Container work-client-1 Started
```

The recreated client then read:

```text
===PERSISTED===
phase5-compose-volume
```

This establishes both service reconciliation and named-volume persistence after destruction/recreation of the consuming container.

**Result: Compose service reconciliation — PASS.**
**Result: volume persistence across container recreation — PASS.**

## 18. `compose down` Semantics

The stack was stopped using normal:

```text
docker compose down
```

Compose removed:

- `work-client-1`;
- `work-web-1`;
- `work_labnet`.

Post-operation verification showed no Compose project containers and no `work_labnet` network.

The named volume remained:

```text
local     work_labdata
```

This matches expected Compose named-volume lifecycle behavior for `down` without `-v`.

**Result: `compose down` container cleanup — PASS.**
**Result: `compose down` network cleanup — PASS.**
**Result: named volume retained by normal `down` — PASS.**

## 19. Persistence Across Full Stack Teardown

After the complete `compose down`, the stack was recreated using `compose up -d`.

The network and both service containers were recreated.

The new `work-client-1` then read:

```text
===AFTER-FULL-RECREATE===
phase5-compose-volume
```

This is stronger than single-container recreation: the proof data survived removal of the entire Compose container/network stack and remained available when the project was subsequently recreated.

**Result: named-volume persistence across full stack teardown/recreation — PASS.**

## 20. `compose down -v` Semantics

Final destructive cleanup used:

```text
docker compose down -v
```

Compose reported removal of:

- `work-web-1`;
- `work-client-1`;
- `work_labdata`;
- `work_labnet`.

Verification returned empty filtered resource sets:

```text
===CONTAINERS===
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

===NETWORK===
NETWORK ID   NAME      DRIVER    SCOPE

===VOLUME===
DRIVER    VOLUME NAME
```

**Result: `compose down -v` destructive lifecycle — PASS.**

## 21. Compatibility Matrix

| Capability | Result |
|---|---|
| Native `wslc compose` command | NOT AVAILABLE |
| Compose plugin bundled in WSLC session | NOT AVAILABLE |
| WSLC internal Docker daemon | PASS / OBSERVED |
| WSLC internal Docker Unix socket | PASS |
| Docker API `_ping` | PASS |
| Docker API version endpoint | PASS |
| External Compose client connection | PASS |
| Client/server API negotiation | PASS |
| Compose `config` | PASS |
| Compose multi-service `up -d` | PASS |
| Compose custom bridge network | PASS |
| Compose service-name DNS | PASS |
| Compose inter-service HTTP | PASS |
| Compose named-volume read/write | PASS |
| Reconciliation after manual service deletion | PASS |
| Volume persistence across container recreation | PASS |
| `compose down` removes containers | PASS |
| `compose down` removes project network | PASS |
| `compose down` retains named volume | PASS |
| Volume persistence across full stack recreation | PASS |
| `compose down -v` removes named volume | PASS |
| Compose-created containers visible to internal Docker CLI | PASS |
| Compose-created containers visible in `wslc list` | NO |
| Microsoft WSLC metadata on WSLC-created container | OBSERVED |
| Microsoft WSLC metadata on tested Compose containers | NOT PRESENT |
| Direct Windows-side bind of WSLC Unix socket via `wslc run -v` | NOT COMPATIBLE WITH TESTED PATH FORM |
| Complete Docker Compose compatibility | NOT CLAIMED |
| Complete Docker Engine API compatibility | NOT CLAIMED |

## 22. Architecture Finding

The tested architecture can be summarized as:

```text
Windows
  |
  +-- Docker Desktop CLI/context
  |     -> Docker Desktop Engine 29.6.1
  |
  +-- wslc.exe
        |
        +-- WSLC system session
              Microsoft Azure Linux 3.0
              |
              +-- dockerd 25.0.3
              +-- Docker API 1.44
              +-- /var/run/docker.sock
              |
              +-- WSLC-managed container resources
              |     -> visible through wslc list
              |     -> Microsoft WSLC metadata observed
              |
              +-- direct Docker API resources
                    -> visible through internal docker CLI
                    -> external Compose can orchestrate them
                    -> tested Compose resources not visible in wslc list
```

This distinction matters operationally. Docker API compatibility does not automatically imply that every API-created object participates fully in WSLC's higher-level CLI management model.

## 23. Practical Interpretation

For the tested WSLC 3.0.1 environment:

- Docker Compose workloads are technically viable against the internal Docker Engine when a Compose client is explicitly connected to the engine socket.
- The absence of `wslc compose` does not mean that the underlying engine cannot support Compose workloads.
- The tested Compose stack behaved normally for networking, DNS, volumes, reconciliation, and teardown.
- Direct Docker API resources may not be surfaced by `wslc list`.
- Windows-side WSLC CLI bind-path semantics make direct mounting of the internal Unix socket non-trivial; running the Compose client from within the WSLC session avoided that boundary.

This should currently be treated as an **advanced compatibility path**, not as evidence of a documented first-class WSLC Compose workflow.

## 24. Scope and Non-Claims

Phase 5 does **not** establish:

- complete Docker Compose specification compatibility;
- compatibility with arbitrary real-world Compose applications;
- Compose `build` behavior inside a multi-service project;
- secrets/configs support;
- profiles;
- `depends_on` health conditions;
- restart-policy behavior under WSLC session restart;
- host-published ports from Compose-created containers;
- multiple Compose networks per service;
- static IP configuration through Compose;
- Compose resource limits;
- Compose GPU reservations;
- complete Docker Engine API compatibility;
- that `com.microsoft.wsl.container.metadata` is the sole criterion used by WSLC for resource visibility;
- that direct Docker API manipulation of WSLC internals is officially supported by Microsoft.

Those require separate validation if needed.

## 25. Final Assessment

Phase 5 demonstrates a meaningful separation between **WSLC's user-facing CLI feature set** and the capabilities of its **underlying Docker-compatible engine**.

The native CLI currently lacks Compose, but the internal engine is sufficiently Docker-compatible for an external Compose v5.5.1 client to execute the tested Compose lifecycle successfully.

Therefore:

> **Docker Compose compatibility: PARTIAL PASS — external Compose works against the WSLC internal Docker API for the tested workload and lifecycle, but native WSLC Compose integration is absent and direct-API-created resources are not fully represented in the tested WSLC CLI view.**

For the Docker API itself:

> **Docker Engine API compatibility: PASS for the tested subset — socket access, version negotiation, container/network/volume operations required by the tested Compose workload all succeeded. Complete API compatibility is not claimed.**

## 26. Phase 5 Cleanup State

The Compose test resources were cleanly removed using `compose down -v`.

Verified absent after cleanup:

```text
work-web-1
work-client-1
work_labnet
work_labdata
```

The pre-existing WSLC validation container `wslc-nginx-test` was not part of the Compose project and was intentionally left untouched.
