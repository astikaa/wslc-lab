# WSLC 3.0.1 — Phase 4 Resource Limits and GPU Validation

**Test date:** 2026-10-04
**Host:** Windows 11 / WSL
**WSL:** 3.0.1.0
**Kernel:** 6.18.40.1-1
**WSLC:** 3.0.1.0
**Docker Desktop:** 4.82.0
**Docker Engine:** 29.6.1
**Host GPU:** Intel(R) Iris(R) Xe Graphics
**Host GPU driver:** 32.0.101.7076

---

## 1. Scope

Phase 4 validates selected WSLC runtime resource-control and GPU-passthrough behavior.

The tested areas are:

- CPU limit configuration and enforcement
- CPU throttling under load
- memory limit configuration and cgroup v2 state
- memory behavior above the configured RAM limit
- swap interaction
- `--gpus all`
- `/dev/dxg` injection
- WSL D3D12/DXCore library injection
- Vulkan userspace enumeration
- cleanup of Phase 4 test containers

This phase does not claim complete resource-management compatibility or complete GPU acceleration compatibility.

## 2. Runtime Resource Options

`wslc run --help` exposes resource-related options including `--cpus`, `--memory`, `--gpus`, `--shm-size`, and `--ulimit`.

CPU, memory, and GPU behavior were tested in this phase.

## 3. CPU and Memory Limit Configuration

A long-running Alpine container was created with:

```cmd
"C:\Program Files\WSL\wslc.exe" run -d --name wslc-lab-limits --cpus 1.5 --memory 256M alpine sleep 3600
```

WSLC emitted:

```text
wsl: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.
```

The container nevertheless started successfully. Inspection showed:

```json
"HostConfig": {
  "Memory": 268435456,
  "NanoCpus": 1500000000,
  "NetworkMode": "bridge",
  "Ulimits": []
}
```

Results:

| Capability | Result |
|---|---|
| `--cpus 1.5` accepted | PASS |
| `--memory 256M` accepted | PASS |
| CPU limit represented in inspect | PASS |
| Memory limit represented in inspect | PASS |
| Container startup with limits | PASS |
| Swap-limit warning | OBSERVED |

## 4. cgroup Version

Inside the limited container:

```text
cgroup on /sys/fs/cgroup type cgroup2 (ro,nosuid,nodev,noexec,relatime)
```

and `/proc/self/cgroup` returned:

```text
0::/
```

The tested WSLC container therefore exposed a unified cgroup v2 hierarchy.

**Result: cgroup v2 — PASS**

## 5. CPU Limit at cgroup Level

For the container created with `--cpus 1.5`, the cgroup value was:

```text
cpu.max=150000 100000
```

This corresponds to a quota of 150000 microseconds per 100000-microsecond period, i.e. 1.5 CPUs.

Before stress, `cpu.stat` showed:

```text
usage_usec 154641
user_usec 106040
system_usec 48601
nice_usec 0
nr_periods 13
nr_throttled 0
throttled_usec 0
nr_bursts 0
burst_usec 0
```

**Result: CPU quota configuration — PASS**

## 6. CPU Throttling Under Load

Four CPU-bound `yes` processes were started concurrently for approximately ten seconds:

```sh
for i in 1 2 3 4; do yes > /dev/null & done
sleep 10
killall yes
wait 2>/dev/null || true
```

After the workload:

```text
cpu.max=150000 100000
---
usage_usec 15340489
user_usec 15002375
system_usec 338113
nice_usec 0
nr_periods 119
nr_throttled 100
throttled_usec 24997431
nr_bursts 0
burst_usec 0
```

The transition from `nr_throttled 0` to `nr_throttled 100` demonstrates active CPU quota enforcement under contention.

**Result: CPU throttling enforcement — PASS**

## 7. Memory Limit at cgroup Level

A container configured with `--memory 64M` reported:

```text
memory.max=67108864
```

which equals exactly 64 MiB.

**Result: Memory cgroup limit configuration — PASS**

## 8. Initial Memory Allocation Test

A 32 MiB file-backed test completed successfully:

```text
PASS-32M
33554432 /tmp/test
```

This test alone was not treated as proof of anonymous-memory enforcement because filesystem/page-cache behavior can differ from direct process memory allocation.

## 9. Anonymous Memory Test

A Python container was used for a direct allocation test. A 32 MiB allocation under a 64 MiB limit completed successfully:

```text
ALLOC-32M-PASS 33554432
```

**Result: 32 MiB anonymous allocation below 64 MiB limit — PASS**

## 10. Allocation Larger Than memory.max

A 128 MiB Python allocation was attempted with the same 64 MiB memory limit.

The process completed:

```text
UNEXPECTED-ALLOC-PASS 134217728
```

Inspection still reported:

```json
"Memory": 67108864
```

This initially appeared inconsistent with the configured memory limit, so a persistent test was performed to inspect live cgroup state.

## 11. Live Memory and Swap Characterization

A long-running container allocated and touched 128 MiB. Logs confirmed:

```text
ALLOCATED 134217728
```

The cgroup reported:

```text
memory.max=67108864
memory.current=38608896
```

`memory.events` showed:

```text
low 0
high 0
max 4454
oom 0
oom_kill 0
oom_group_kill 0
```

The process had encountered the cgroup maximum repeatedly without being OOM-killed.

## 12. Swap Behavior

The same live container reported:

```text
memory.current=38608896
memory.max=67108864
memory.swap.current=105000960
memory.swap.max=max
```

Process status showed:

```text
VmSize:   141584 kB
VmRSS:     37600 kB
RssAnon:   32332 kB
RssFile:    5268 kB
VmSwap:   102120 kB
```

`/proc/swaps` reported an active `/dev/sdd` swap device.

The 128 MiB allocation therefore did not demonstrate failure of `memory.max`. Instead, resident memory remained constrained while substantial process memory moved to swap. Because `memory.swap.max` was `max`, the process could survive despite allocating substantially more memory than the configured resident-memory ceiling.

Results:

| Capability / Observation | Result |
|---|---|
| `memory.max` configured to 64 MiB | PASS |
| Resident memory constrained by cgroup | PASS |
| Memory pressure recorded in `memory.events` | PASS |
| OOM kill during tested workload | NO |
| Swap used by limited process | OBSERVED |
| `memory.swap.max` | `max` |
| Independent swap limit enforcement | NOT ESTABLISHED |

The tested `--memory` behavior must not be interpreted as a combined RAM+swap ceiling.

## 13. Host GPU

Windows PowerShell reported:

```text
Name                         DriverVersion
----                         -------------
Intel(R) Iris(R) Xe Graphics 32.0.101.7076
```

No NVIDIA GPU was present in the tested system. CUDA/NVIDIA-specific validation was therefore not applicable.

## 14. WSL GPU Baseline

The Ubuntu 24.04 WSL distribution exposed `/dev/dxg`:

```text
crw-rw-rw- 1 root root 10, 258 ... /dev/dxg
```

and `/usr/lib/wsl/lib` contained:

```text
libd3d12.so
libd3d12core.so
libdxcore.so
```

This establishes the host WSL graphics baseline.

## 15. WSLC Container Without `--gpus all`

A normal WSLC Alpine container showed:

```text
---DXG---
ls: /dev/dxg: No such file or directory
---WSL-LIB---
ls: /usr/lib/wsl/lib: No such file or directory
```

Therefore GPU resources were not injected into an ordinary WSLC container by default.

**Result: Default GPU isolation — PASS**

## 16. WSLC `--gpus all`

The same test with `--gpus all` exposed:

```text
/dev/dxg
```

and:

```text
/usr/lib/wsl/lib/libd3d12.so
/usr/lib/wsl/lib/libd3d12core.so
/usr/lib/wsl/lib/libdxcore.so
```

A persistent `--gpus all` container exposed the same resources.

A/B result:

| Resource | Default container | `--gpus all` |
|---|---|---|
| `/dev/dxg` | absent | present |
| `/usr/lib/wsl/lib` | absent | present |
| `libd3d12.so` | absent | present |
| `libd3d12core.so` | absent | present |
| `libdxcore.so` | absent | present |

**Result: WSLC GPU resource injection — PASS**

## 17. D3D12 / DXCore Userspace Libraries

Inside the GPU-enabled Alpine container, `libd3d12.so`, `libd3d12core.so`, and `libdxcore.so` were present. `ldd` was also able to process `libdxcore.so` in the tested Alpine environment.

This confirms library presence and loader visibility. It does not by itself prove successful accelerated rendering or compute execution.

**Result: WSL graphics library injection — PASS**

## 18. GLX Test

An Ubuntu 24.04 GPU-enabled container installed `mesa-utils` and executed `glxinfo -B`.

Result:

```text
Error: unable to open display
```

The test container was headless and no X11/Wayland display was configured.

**Result: GLX hardware test — INCONCLUSIVE / NOT APPLICABLE TO HEADLESS TEST**

This is not treated as evidence that GPU passthrough failed.

## 19. Vulkan Enumeration

A GPU-enabled Ubuntu 24.04 container installed `vulkan-tools` and `mesa-vulkan-drivers`, then executed `vulkaninfo --summary`.

Vulkan initialized successfully:

```text
Vulkan Instance Version: 1.3.275
```

However, the only enumerated device was:

```text
GPU0:
    deviceType = PHYSICAL_DEVICE_TYPE_CPU
    deviceName = llvmpipe (LLVM 20.1.2, 256 bits)
    driverName = llvmpipe
```

Therefore Vulkan userspace was operational, but the tested stack used Mesa's CPU software renderer rather than the Intel Iris Xe hardware GPU.

Results:

| Capability | Result |
|---|---|
| Vulkan loader/runtime | PASS |
| Hardware Vulkan acceleration | NOT ESTABLISHED |
| Observed Vulkan device | llvmpipe CPU |

## 20. Vulkan ICD Inspection

The tested Ubuntu userspace exposed ICD definitions for:

```text
asahi
radeon
intel
nouveau
lvp
virtio
intel_hasvk
gfxstream
```

No tested ICD explicitly provided a DXG/D3D12 Vulkan path for the injected WSL GPU interface.

The presence of `intel_icd.json` is not sufficient to claim Intel Iris Xe acceleration through `/dev/dxg`; actual enumeration selected llvmpipe.

This observation is documented without attributing it to a WSLC defect.

**DXG/D3D12 Vulkan hardware path in tested stock Ubuntu userspace: NOT ESTABLISHED**

## 21. GPU Validation Boundary

The Phase 4 evidence supports the following bounded claim:

> WSLC 3.0.1 accepts `--gpus all` and, on the tested Intel Iris Xe / WSL environment, injects `/dev/dxg` together with WSL D3D12 and DXCore userspace libraries into the container.

The evidence does not support the broader claim that hardware-accelerated Vulkan workloads are fully functional in arbitrary WSLC containers.

The stock Ubuntu 24.04 Vulkan experiment enumerated llvmpipe rather than the Intel hardware adapter.

No CUDA validation was attempted because the test host did not contain an NVIDIA GPU.

## 22. Cleanup

Phase 4 temporary persistent containers were removed:

```text
wslc-lab-gpu
wslc-lab-mem-live
wslc-lab-limits
```

Final `wslc list` showed only the pre-existing baseline container:

```text
wslc-nginx-test
```

**Result: Phase 4 temporary resource cleanup — PASS**

## 23. Phase 4 Result Matrix

| Capability | Result |
|---|---|
| `--cpus` accepted | PASS |
| CPU quota represented by WSLC inspect | PASS |
| cgroup v2 CPU quota | PASS |
| CPU throttling under contention | PASS |
| `--memory` accepted | PASS |
| Memory limit represented by WSLC inspect | PASS |
| `memory.max` configured/enforced | PASS |
| Memory pressure accounting | PASS |
| Anonymous allocation below limit | PASS |
| Swap use under memory pressure | OBSERVED |
| Independent swap ceiling | NOT ESTABLISHED |
| `memory.swap.max` in tested environment | `max` |
| `--gpus all` accepted | PASS |
| Default container excludes `/dev/dxg` | PASS |
| `--gpus all` injects `/dev/dxg` | PASS |
| `--gpus all` injects WSL D3D12/DXCore libraries | PASS |
| GPU injection A/B behavior | PASS |
| Vulkan userspace initialization | PASS |
| Intel hardware Vulkan device enumeration | NOT ESTABLISHED |
| Vulkan software fallback | llvmpipe OBSERVED |
| NVIDIA/CUDA validation | NOT APPLICABLE |
| Phase 4 temporary resource cleanup | PASS |

## 24. Overall Phase 4 Assessment

### PASS — with explicitly bounded GPU and swap claims

WSLC 3.0.1 successfully demonstrated:

- CPU limit configuration
- cgroup v2 CPU quota representation
- measurable CPU throttling
- memory limit configuration
- cgroup v2 memory accounting
- memory pressure behavior
- GPU resource isolation by default
- `/dev/dxg` injection through `--gpus all`
- WSL D3D12/DXCore library injection
- clean lifecycle of Phase 4 test containers

Two important boundaries were observed:

1. A configured memory limit did not act as a total RAM+swap ceiling in the tested environment because `memory.swap.max=max`.
2. GPU plumbing was successfully exposed, but the tested stock Ubuntu Vulkan stack enumerated llvmpipe rather than Intel Iris Xe hardware acceleration.

Accordingly, Phase 4 validates WSLC's tested resource-control and GPU-injection mechanisms without claiming complete swap-control or hardware-GPU workload compatibility.
