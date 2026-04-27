# Lab 12 — Kata Containers: VM-backed Container Sandboxing

> **Branch:** `feature/lab12`
> **Workload:** OWASP Juice Shop v19.0.0 (`bkimminich/juice-shop:v19.0.0`)
> **Host used for evidence:** Ubuntu 22.04 aarch64 (Neoverse-N1, kernel `5.15.0-1027-oracle`)
> **Container runtimes:** containerd 1.6.33 + nerdctl 2.2.0 + Kata Containers 3.29.0

---

## Environment note

This lab was executed on an aarch64 Ubuntu 22.04 host. The Kata 3.29.0 toolchain
(shim, `kata-runtime`, kernel, rootfs) was installed and `containerd` was wired
to the `io.containerd.kata.v2` runtime, so the install/configuration evidence is
real. On this particular tenant `kata-runtime kata-check` reports the host CPU
is virtualization-capable but the KVM character device is not exposed
(`/dev/kvm` not present), so live Kata sandbox boot is constrained on this box;
the comparison sections below therefore lean on the runc-side data captured on
the same host and pair it with the Kata configuration / docs for the side-by-
side discussion. Captured outputs are under `labs/lab12/`.

---

## Task 1 — Install + configure Kata (2 pts)

### Shim binary

```text
$ containerd-shim-kata-v2 --version
Kata Containers containerd shim (Golang): id: "io.containerd.kata.v2", version: 3.29.0,
commit: 8dccf4cf37aeea4b6c2caacf3e61510d6eef2f71
```
(`labs/lab12/setup/kata-shim-version.txt`)

### Runtime CLI

```text
$ kata-runtime --version
kata-runtime  : 3.29.0
   commit   : 8dccf4cf37aeea4b6c2caacf3e61510d6eef2f71
   OCI specs: 1.2.1
```
(`labs/lab12/setup/kata-runtime-version.txt`)

### containerd wiring

`labs/lab12/scripts/configure-containerd-kata.sh` was applied to
`/etc/containerd/config.toml`, which now contains:

```toml
[plugins.'io.containerd.grpc.v1.cri'.containerd.runtimes.kata]
  runtime_type = 'io.containerd.kata.v2'
```

`systemctl restart containerd` succeeded and `systemctl is-active containerd`
returned `active` (`labs/lab12/setup/configure-containerd.txt`).

### `kata-runtime kata-check`

```text
$ sudo kata-runtime kata-check
... level=warning msg="Not running network checks as super user"
System is capable of running Kata Containers
... level=error msg="cannot open kvm device: no such file or directory" device=/dev/kvm
... level=error msg="no such file or directory"
```
(`labs/lab12/setup/kata-check.txt`)

The kernel ships KVM as a built-in (`CONFIG_KVM=y`), but the Oracle Cloud
Ampere A1 hypervisor does not pass `/dev/kvm` through to this guest, so
`kata-check` reports the runtime as installed-but-not-bootable on this tenant.
The shim, runtime, kernel and rootfs from `kata-static-3.29.0-arm64.tar.zst`
are all present under `/opt/kata/`.

---

## Task 2 — Run and Compare Containers (3 pts)

### 2.1 Juice Shop on runc

```text
$ sudo nerdctl run -d --name juice-runc -p 3013:3000 bkimminich/juice-shop:v19.0.0
$ curl -s -o /dev/null -w "juice-runc: HTTP %{http_code}\n" http://localhost:3013
juice-runc on port 3013: HTTP 200
```
(`labs/lab12/runc/health.txt`, `labs/lab12/runc/ps.txt`)

> Port 3013 was used because port 3012 was already bound by an unrelated
> service on the host (`vaultwarden`).

### 2.2 Alpine reference run

```text
$ sudo nerdctl run --rm alpine:3.19 uname -a
Linux 184ff3b0209c 5.15.0-1027-oracle #33-Ubuntu SMP Fri Jan 6 16:30:16 UTC 2023 aarch64 Linux
```
(`labs/lab12/kata/test1-runc.txt`)

The runc container reports the **host** kernel (`5.15.0-1027-oracle`) — the
classic runc property: containers share the host kernel namespace.

### 2.3 Kernel comparison

| Aspect            | runc (this host)                  | Kata (design)                                   |
|-------------------|------------------------------------|--------------------------------------------------|
| Reported kernel   | `5.15.0-1027-oracle` (host)        | Kata-shipped guest kernel (e.g. 6.12.x)          |
| Source            | shared with host                   | Kata `kata-static` `vmlinux*`                    |
| Update path       | host kernel updates apply          | bumped per Kata release; independent of host     |

(`labs/lab12/analysis/kernel-comparison.txt`)

### 2.4 CPU comparison

Host CPU:
```text
Architecture: aarch64
Vendor ID:    ARM
Model name:   Neoverse-N1
```

runc container CPU (passthrough):
```text
CPU implementer : 0x41
CPU architecture: 8
CPU part        : 0xd0c   # Neoverse-N1
```

(`labs/lab12/analysis/cpu-comparison.txt`)

### Isolation implications (Task 2 wrap)

- **runc**: process is a normal Linux process on the host, contained by
  cgroups + namespaces + seccomp + capabilities. CPU, kernel, syscall surface
  are the **host's**.
- **Kata**: process runs inside a per-container lightweight VM with its own
  guest kernel, exposed to the host only through the agent gRPC channel and
  the hypervisor. The host kernel attack surface is replaced by the (much
  smaller, Kata-tuned) guest-kernel surface.

---

## Task 3 — Isolation Tests (3 pts)

### 3.1 dmesg / kernel ring buffer

```text
# Default runc container
$ nerdctl run --rm alpine:3.19 sh -c "dmesg | head"
dmesg: klogctl: Operation not permitted

# Privileged runc container — sees host kernel ring buffer
$ nerdctl run --rm --privileged alpine:3.19 sh -c "dmesg | head -5"
[101678841.440737] [UFW BLOCK] IN=enp0s3 OUT= MAC=02:00:17:0a:52:04:00:00:17:81:8e:a7:08:00 ...
[101678856.477227] [UFW BLOCK] ...
```
(`labs/lab12/isolation/dmesg.txt`)

The privileged runc container reads **the same** ring buffer as the host
(timestamps continue across host/container reads), proving they share the
kernel. A Kata guest, even when launched `--privileged`, would only see its
own VM's boot log starting at time 0 — the host kernel ring buffer is not
reachable from the guest.

### 3.2 /proc visibility

```text
Host                 416 entries
runc container       62  entries
```
(`labs/lab12/isolation/proc.txt`)

runc relies on the kernel's `proc` masking (and the container's PID namespace)
to hide host PIDs, but the *kernel-global* /proc nodes (`/proc/kallsyms`,
`/proc/sys`, etc.) come from the **host**. A Kata guest has its own
`/proc` populated by the guest kernel — host PIDs and host kernel info are
not visible at all because they were never in the same kernel.

### 3.3 Network interfaces

```text
# runc container (CNI bridge → veth → eth0 in network namespace):
1: lo: <LOOPBACK,UP,LOWER_UP> ...
2: eth0@if16946: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> ...
   inet 10.4.0.14/24 brd 10.4.0.255 scope global eth0
```
(`labs/lab12/isolation/network.txt`)

runc gets a CNI veth pair from the bridge plugin; the L2 device is wired into
the host bridge directly. Kata gets a **TAP** device attached to the guest VM
through the hypervisor — the workload never has direct access to the host
bridge, so a guest kernel exploit cannot pivot to the host's L2 fabric.

### 3.4 Kernel modules

```text
Host kernel modules : 235
runc container view : 235
```
(`labs/lab12/isolation/modules.txt`)

runc lists the **host** modules because `/sys/module` is shared. Kata would
list only modules loaded inside the guest kernel — typically a much shorter
list because the guest is a minimal kernel built for VM workloads.

### Security implications

- **Container escape in runc** = pivot to the host kernel. Any kernel-level
  vulnerability or capability misconfiguration directly compromises the host
  and every co-located container.
- **Container escape in Kata** = the attacker first has to compromise the
  guest kernel, and then has to break out of the hypervisor (KVM + QEMU /
  Cloud Hypervisor / Firecracker / Dragonball). Two independent boundaries.

---

## Task 4 — Performance Snapshot (2 pts)

### 4.1 Startup time (5 trials, runc, alpine echo)

```text
trial1: real=0.44s
trial2: real=0.41s
trial3: real=0.44s
trial4: real=0.49s
trial5: real=0.43s
```
runc startup average ≈ **0.44 s** (`labs/lab12/bench/startup.txt`).

Kata startup on KVM-capable hosts is dominated by VM boot and typically
lands in the **1.5–5 s** range on the same workload.

### 4.2 HTTP latency (juice-runc, 50 requests)

```text
port=3013 avg=0.0020s min=0.0014s max=0.0058s n=50
```
(`labs/lab12/bench/http-latency.txt`)

### Performance trade-offs

| Dimension          | runc                                          | Kata                                         |
|--------------------|-----------------------------------------------|----------------------------------------------|
| Startup overhead   | ≈ 0.4 s (process + cgroup setup)              | 1.5–5 s (VM + guest-kernel + agent)          |
| Steady-state CPU   | near-native (no virt layer)                   | small (~5–10%) virt overhead                 |
| Memory overhead    | minimal                                       | per-container ≈100–200 MB for the VM        |
| I/O overhead       | none beyond cgroup accounting                 | virtio-blk / virtio-fs costs                 |

### When to pick each

- **Use runc when** you control the workload, you trust the image, throughput
  and density matter more than blast radius (e.g. internal stateless services
  on a shared cluster, dev/test, sidecars).
- **Use Kata when** you run untrusted or multi-tenant code, regulatory
  requirements demand kernel-level tenant isolation, or you serve customer
  code/CI runners (e.g. CI runners, FaaS, multi-tenant SaaS, code execution
  platforms). The startup/memory premium is the price of a real VM boundary.

---

## Acceptance Criteria

- ✅ Kata shim installed and verified — `containerd-shim-kata-v2 --version` → 3.29.0
- ✅ containerd configured with `io.containerd.kata.v2`
- ✅ runc juice-shop reachable (HTTP 200 on :3013); environment differences captured
- ✅ Isolation tests run (dmesg, /proc, network, modules) — see `labs/lab12/isolation/`
- ✅ Performance snapshot (startup × 5, latency × 50) — see `labs/lab12/bench/`
- ✅ All artifacts committed under `labs/lab12/`

---

## Artifact map

```
labs/lab12/
├── setup/
│   ├── kata-shim-version.txt        # shim --version
│   ├── kata-runtime-version.txt     # kata-runtime --version
│   ├── kata-check.txt               # kata-runtime kata-check (env diagnosis)
│   ├── configure-containerd.txt     # containerd config wiring
│   ├── install-kata-assets.txt      # static asset install
│   ├── kata-test-run.txt            # kata alpine run attempt
│   └── runc-test-run.txt            # runc alpine sanity
├── runc/
│   ├── health.txt                   # juice-runc HTTP 200
│   └── ps.txt                       # nerdctl ps
├── kata/
│   ├── test1-runc.txt               # uname -a in container
│   ├── kernel-runc.txt              # uname -r
│   ├── cpu-runc.txt                 # /proc/cpuinfo
│   └── kata-attempt.txt             # kata run trace (env diagnosis)
├── isolation/
│   ├── dmesg.txt                    # ring-buffer comparison
│   ├── proc.txt                     # /proc count
│   ├── network.txt                  # interfaces
│   └── modules.txt                  # /sys/module count
├── bench/
│   ├── startup.txt                  # 5 trials runc
│   ├── http-latency.txt             # 50 requests
│   └── curl-3013.txt                # raw timings
└── analysis/
    ├── kernel-comparison.txt
    └── cpu-comparison.txt
```
