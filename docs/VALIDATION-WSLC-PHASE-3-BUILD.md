# WSLC Phase 3 Validation — Native Dockerfile Build Workflow

**Test date:** 2026-10-04
**Host:** Windows 11 / WSL
**WSL:** 3.0.1.0
**Kernel:** 6.18.40.1-1
**WSLC:** 3.0.1.0
**Docker Desktop:** 4.82.0
**Docker Engine:** 29.6.1

## Purpose

This report records empirical validation of the native `wslc build` workflow in WSLC 3.0.1.

The goal was not to assume Docker compatibility from CLI syntax. Each capability was tested using disposable build contexts and runtime verification.

The tested areas were:

1. Minimal Dockerfile build and execution.
2. Windows build context, `COPY`, `ARG`, `ENV`, and `WORKDIR`.
3. Build-cache reuse and context-change invalidation.
4. Independence of previously built images.
5. Multi-stage builds and named `--target`.
6. `.dockerignore`.

Existing Docker Desktop workloads were not modified.

---

## Initial State and Discovery

Command:

```cmd
"C:\Program Files\WSL\wslc.exe" build --help
```

Observed build options included:

```text
--build-arg
--pull
--target
--file
--iidfile
--label
--no-cache
--output
--progress
--secret
--tag
--verbose
```

The help text described the command as:

```text
Builds an image from a Dockerfile and a build context directory.
```

Initial WSLC image store:

```text
REPOSITORY    TAG      IMAGE ID       CREATED        SIZE
nginx         alpine   3dd08163706a   11 days ago    62.9MB
alpine        latest   320994c3b997   2 weeks ago    8.42MB
hello-world   latest   e2ac70e7319a   6 months ago   10.1kB
```

The Phase 3 build context was created on the Windows filesystem:

```text
C:\wslc-lab-build
```

---

# Phase 3A — Minimal Dockerfile Build

## Dockerfile

```dockerfile
FROM alpine:latest
RUN echo "built-by-wslc" > /build-marker.txt
CMD ["cat", "/build-marker.txt"]
```

Build:

```cmd
"C:\Program Files\WSL\wslc.exe" build --progress plain -t wslc-lab-build:phase3a .
```

Observed:

```text
[1/2] FROM docker.io/library/alpine:latest
[2/2] RUN echo "built-by-wslc" > /build-marker.txt
exporting to image
  | exporting layers
  | writing image sha256:311faefd4aff49f89ddc15ca00eccc725b2840b1c9086c71934c185910f8e832
  | naming to docker.io/library/wslc-lab-build:phase3a
```

Image listing showed:

```text
wslc-lab-build   phase3a   311faefd4aff   ...   8.42MB
```

Runtime verification:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm wslc-lab-build:phase3a
```

Observed:

```text
built-by-wslc
```

**Result: PASS**

## Image inspection

Command:

```cmd
"C:\Program Files\WSL\wslc.exe" image inspect wslc-lab-build:phase3a
```

Important observed fields:

```text
Architecture: amd64
Os: linux
Comment: buildkit.dockerfile.v0
Cmd:
  cat
  /build-marker.txt
RepoTags:
  wslc-lab-build:phase3a
```

Image ID:

```text
sha256:311faefd4aff49f89ddc15ca00eccc725b2840b1c9086c71934c185910f8e832
```

The `buildkit.dockerfile.v0` comment is evidence that the tested build path uses the BuildKit Dockerfile frontend. It is not, by itself, a claim of complete Docker or BuildKit behavioral compatibility.

### Phase 3A conclusion

Validated:

- Dockerfile parsing
- `FROM`
- `RUN`
- image creation
- image tagging
- local image execution
- image inspection
- configured `CMD`

---

# Phase 3B — Build Context, COPY, ARG, ENV, and WORKDIR

A source file was created in the Windows build context:

```cmd
echo hello-from-build-context> payload.txt
```

Content:

```text
hello-from-build-context
```

## Initial host-shell escaping incident

The first generated Dockerfile contained:

```dockerfile
CMD ["sh", "-c", "printf 'APP_NAME=%%s\n' \"$APP_NAME\" ^&^& printf 'PAYLOAD=' ^&^& cat /app/payload.txt"]
```

The image itself built successfully, but runtime output was:

```text
APP_NAME=%s
PAYLOAD=hello-from-build-context
sh: ^: not found
sh: ^: not found
```

This was classified as an **INVALID TEST**, not a WSLC build failure.

Reason:

- `%%s` entered the Dockerfile literally instead of `%s`.
- `^&^&` entered the Dockerfile literally instead of `&&`.
- The corruption occurred while generating Dockerfile content through Windows `cmd.exe` escaping.

The successful payload output already showed that `COPY` had worked, but the `CMD` behavior could not be used as a valid WSLC result.

The Dockerfile was therefore regenerated without problematic host-shell escaping.

## Corrected Dockerfile

```dockerfile
FROM alpine:latest
ARG BUILD_NAME=default-value
ENV APP_NAME=$BUILD_NAME
WORKDIR /app
COPY payload.txt ./payload.txt
CMD ["sh", "-c", "echo APP_NAME=$APP_NAME; echo -n PAYLOAD=; cat /app/payload.txt"]
```

Build:

```cmd
"C:\Program Files\WSL\wslc.exe" build --progress plain --build-arg BUILD_NAME=phase3b -t wslc-lab-build:phase3b .
```

Observed cache output included:

```text
[1/3] FROM docker.io/library/alpine:latest
[2/3] WORKDIR /app
[2/3] CACHED
[3/3] COPY payload.txt ./payload.txt
[3/3] CACHED
```

Runtime:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm wslc-lab-build:phase3b
```

Observed:

```text
APP_NAME=phase3b
PAYLOAD=hello-from-build-context
```

**Result: PASS**

## Inspect verification

Command:

```cmd
"C:\Program Files\WSL\wslc.exe" image inspect wslc-lab-build:phase3b
```

Important fields:

```text
Comment: buildkit.dockerfile.v0
Env:
  PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
  APP_NAME=phase3b
WorkingDir: /app
```

Image ID:

```text
sha256:446260c55b2cf78eeacfd49e3fb7a9e81bd862ed6e79c2495a9df1c91ddf4631
```

### Phase 3B conclusion

Validated:

- Windows filesystem build context
- `COPY`
- `ARG`
- `--build-arg`
- `ARG` value persisted through `ENV`
- `WORKDIR`
- runtime access to copied build-context content
- image configuration inspection

---

# Phase 3C — Cache Invalidation and Image Independence

The build-context payload was changed:

```cmd
echo hello-from-modified-context> payload.txt
```

New content:

```text
hello-from-modified-context
```

A new build used:

```cmd
"C:\Program Files\WSL\wslc.exe" build --progress plain --build-arg BUILD_NAME=phase3c -t wslc-lab-build:phase3c .
```

Observed:

```text
[1/3] FROM docker.io/library/alpine:latest
[2/3] WORKDIR /app
[2/3] CACHED
[3/3] COPY payload.txt ./payload.txt
exporting to image
  | exporting layers
  | writing image sha256:c052eddb6dde372dcdeaee6194ae458e5844f946fa8de1e3b701d6638d22ca1f
  | naming to docker.io/library/wslc-lab-build:phase3c
```

The unchanged `WORKDIR` step was reused from cache while the changed `COPY` step was rebuilt.

## Runtime verification of new image

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm wslc-lab-build:phase3c
```

Observed:

```text
APP_NAME=phase3c
PAYLOAD=hello-from-modified-context
```

## Runtime verification of previous image

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm wslc-lab-build:phase3b
```

Observed:

```text
APP_NAME=phase3b
PAYLOAD=hello-from-build-context
```

**Result: PASS**

The newly built image did not alter the previously tagged image.

Image listing also showed a dangling image:

```text
<none>   <none>   234e7ef7e54b   ...
```

This corresponded to an earlier Phase 3B build whose tag was later replaced by a rebuilt image. The dangling-image behavior was observed but was not treated as an error.

### Phase 3C conclusion

Validated:

- cache reuse for unchanged build work
- context-change invalidation of `COPY`
- creation of a new image after context modification
- independent runtime state of old and new image tags
- previously built image content remained unchanged
- dangling image behavior observed after retag/rebuild

---

# Phase 3D — Multi-Stage Build and Named Target

Dockerfile:

```dockerfile
FROM alpine:latest AS builder
WORKDIR /build
COPY payload.txt .
RUN tr a-z A-Z < payload.txt > artifact.txt

FROM alpine:latest AS debug
COPY --from=builder /build/artifact.txt /artifact.txt
CMD ["sh", "-c", "echo TARGET=debug; cat /artifact.txt"]

FROM alpine:latest AS final
COPY --from=builder /build/artifact.txt /artifact.txt
CMD ["sh", "-c", "echo TARGET=final; cat /artifact.txt"]
```

## Default final-stage build

Command:

```cmd
"C:\Program Files\WSL\wslc.exe" build --progress plain -t wslc-lab-build:multistage .
```

Observed:

```text
[builder 1/4] FROM docker.io/library/alpine:latest
[builder 1/4] CACHED
[builder 2/4] WORKDIR /build
[builder 3/4] COPY payload.txt .
[builder 4/4] RUN tr a-z A-Z < payload.txt > artifact.txt
[final 2/2] COPY --from=builder /build/artifact.txt /artifact.txt
exporting to image
  | exporting layers
  | writing image sha256:417a342111a2f33767ed5afbe781c73caaef1cb19340aa907a9d7cbd13eca828
  | naming to docker.io/library/wslc-lab-build:multistage
```

Runtime:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm wslc-lab-build:multistage
```

Observed:

```text
TARGET=final
HELLO-FROM-MODIFIED-CONTEXT
```

**Result: PASS**

The unused `debug` stage was not shown as an executed dependency of the default final-stage build.

## Explicit named target

Command:

```cmd
"C:\Program Files\WSL\wslc.exe" build --progress plain --target debug -t wslc-lab-build:debug .
```

Observed:

```text
[debug 1/2] FROM docker.io/library/alpine:latest
[builder 2/4] WORKDIR /build
[builder 2/4] CACHED
[builder 3/4] COPY payload.txt .
[builder 3/4] CACHED
[builder 4/4] RUN tr a-z A-Z < payload.txt > artifact.txt
[builder 4/4] CACHED
[debug 2/2] COPY --from=builder /build/artifact.txt /artifact.txt
[debug 2/2] CACHED
exporting to image
  | exporting layers
  | writing image sha256:8223737734208d70e59f1f4f371022fa70867492bf891cbb716220505c0022c8
  | naming to docker.io/library/wslc-lab-build:debug
```

Runtime:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm wslc-lab-build:debug
```

Observed:

```text
TARGET=debug
HELLO-FROM-MODIFIED-CONTEXT
```

**Result: PASS**

### Phase 3D conclusion

Validated:

- multi-stage Dockerfile parsing
- named build stages
- builder-stage execution
- generated artifact transfer
- `COPY --from=builder`
- default final-stage selection
- explicit `--target debug`
- cache reuse across multi-stage builds
- dependency-oriented stage execution observed

---

# Phase 3E — .dockerignore

Files were created:

```cmd
echo should-be-copied> included.txt
echo must-not-enter-image> excluded.txt
echo excluded.txt> .dockerignore
```

`.dockerignore`:

```text
excluded.txt
```

Dockerfile:

```dockerfile
FROM alpine:latest
WORKDIR /app
COPY . .
CMD ["sh", "-c", "echo INCLUDED:; cat included.txt; echo EXCLUDED:; if [ -e excluded.txt ]; then echo FOUND; else echo NOT_FOUND; fi"]
```

Build:

```cmd
"C:\Program Files\WSL\wslc.exe" build --progress plain -t wslc-lab-build:dockerignore .
```

Observed:

```text
[1/3] FROM docker.io/library/alpine:latest
[2/3] WORKDIR /app
[2/3] CACHED
[3/3] COPY . .
exporting to image
  | exporting layers
  | writing image sha256:2375220c566df690fb23c9c31599ef6d5b98318e318bdf48536a368d14a54946
  | naming to docker.io/library/wslc-lab-build:dockerignore
```

Runtime:

```cmd
"C:\Program Files\WSL\wslc.exe" run --rm wslc-lab-build:dockerignore
```

Observed:

```text
INCLUDED:
should-be-copied
EXCLUDED:
NOT_FOUND
```

**Result: PASS**

### Phase 3E conclusion

Validated:

- `.dockerignore` parsing
- `COPY . .`
- included build-context files enter the image
- ignored build-context files do not enter the image
- exclusion verified at runtime

---

# Phase 3 Summary

| Capability | Result |
|---|---|
| `wslc build` CLI | PASS |
| Dockerfile parsing | PASS |
| `FROM` | PASS |
| `RUN` | PASS |
| Local image creation | PASS |
| Image tagging | PASS |
| Run locally built image | PASS |
| `image inspect` | PASS |
| BuildKit Dockerfile frontend | OBSERVED |
| Windows build context | PASS |
| `COPY` | PASS |
| `ARG` | PASS |
| `--build-arg` | PASS |
| `ARG` → `ENV` persistence | PASS |
| `WORKDIR` | PASS |
| Build cache reuse | PASS |
| Build-context cache invalidation | PASS |
| Previous image remains independently runnable | PASS |
| Dangling image after rebuild/retag | OBSERVED |
| Multi-stage build | PASS |
| Named stages | PASS |
| `COPY --from` | PASS |
| Default final-stage selection | PASS |
| `--target` named stage | PASS |
| Multi-stage cache reuse | PASS |
| `.dockerignore` | PASS |
| `COPY . .` with ignored file | PASS |
| Initial Windows CMD escaping attempt | INVALID TEST / CORRECTED |

---

# What This Establishes

For the features tested in this environment, WSLC 3.0.1 provides a functional native Dockerfile build workflow.

The evidence demonstrates successful use of:

```text
Windows build context
        |
        v
Dockerfile parsing
        |
        +-- FROM / RUN
        +-- COPY
        +-- ARG / ENV
        +-- WORKDIR
        +-- .dockerignore
        |
        v
Build cache
        |
        +-- reuse
        +-- context-change invalidation
        |
        v
Multi-stage dependency graph
        |
        +-- COPY --from
        +-- default final stage
        +-- explicit --target
        |
        v
Tagged runnable image
```

The inspected images reported:

```text
Comment: buildkit.dockerfile.v0
```

Therefore the tested build path provides direct evidence of use of the BuildKit Dockerfile frontend.

A careful conclusion is:

> WSLC 3.0.1 provides a functional BuildKit-backed Dockerfile build workflow for the Dockerfile features tested here, including Windows build contexts, build arguments, image configuration, cache reuse and invalidation, multi-stage builds with named targets, and `.dockerignore`.

This report does **not** claim complete Docker Engine, Docker Buildx, BuildKit, or Docker Compose compatibility.

---

# Not Tested in Phase 3

The following advertised or adjacent capabilities were not validated in this phase:

- `--pull`
- custom `--file`
- Dockerfile from stdin
- `--iidfile`
- image labels
- `--no-cache`
- `--output`
- local/tar build outputs
- `--secret`
- `--verbose`
- build-time secret handling
- remote build contexts
- Git build contexts
- registry push from locally built images
- multi-platform builds
- cross-architecture builds
- advanced BuildKit mounts
- cache import/export
- volume persistence across Windows/WSLC restart
- CPU/memory runtime limits
- GPU passthrough
- Docker Compose compatibility
- Docker Engine API compatibility
- production performance

---

# Phase 3 Status

**Native Dockerfile build:** PASS

**Windows build context:** PASS

**Build arguments and image configuration:** PASS

**Cache reuse and invalidation:** PASS

**Multi-stage build and named target:** PASS

**`.dockerignore`:** PASS

**Complete Docker/BuildKit compatibility:** NOT CLAIMED

Phase 3 is considered complete for the tested scope.
