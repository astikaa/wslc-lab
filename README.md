# WSLC Lab

Empirical validation and experimentation repository for Windows Subsystem for Linux Containers (WSLC).

## Test Environment

- Windows: 10.0.26200.9457
- WSL: 3.0.1.0
- WSL Kernel: 6.18.40.1-1
- Ubuntu: 24.04 / WSL2
- WSLC: 3.0.1.0
- Docker Desktop: 4.82.0
- Docker Engine: 29.6.1
- Docker Compose: 5.3.0

## Purpose

This repository documents reproducible WSLC experiments and compares observed behavior with an existing Docker Desktop + WSL development environment.

The project follows an evidence-first approach:

1. Record the environment.
2. Document the exact test command.
3. Record the observed result.
4. Separate observations from assumptions.
5. Keep experiments disposable.
6. Do not modify existing Docker workloads unless explicitly required.

## Current Status

Phase 1 basic validation completed.

Validated:

- WSL 3.0.1 upgrade
- Docker Desktop coexistence
- WSLC image pull and container execution
- WSLC/Docker Desktop image-store isolation
- WSLC bridge networking
- Windows localhost port publishing
- Ubuntu-to-WSLC network boundary
- Windows NTFS bind mounts
- Ubuntu WSL filesystem bind mounts
- Bidirectional filesystem read/write

Not yet validated:

- WSLC managed volumes
- Custom networks
- Container-to-container DNS/networking
- Dockerfile builds
- GPU passthrough
- CPU/memory limits
- Performance
- Docker Compose compatibility
- Docker Engine API compatibility

## Documentation

- `docs/VALIDATION-WSLC-3.0.1.md`
- `docs/TEST-MATRIX.md`

## Repository Layout

- `docs/` - test reports and findings
- `experiments/networking/` - network experiments
- `experiments/storage/` - bind-mount/filesystem experiments
- `experiments/volumes/` - managed-volume experiments
- `experiments/build/` - image build experiments
- `experiments/gpu/` - GPU passthrough experiments
- `experiments/compose/` - Compose/API compatibility research
- `scripts/` - reproducible test helpers

## Safety

Experiments should use disposable containers, images, networks, volumes, ports, and test data.

Existing Docker Desktop workloads and persistent development data must not be reused as writable WSLC test state.
