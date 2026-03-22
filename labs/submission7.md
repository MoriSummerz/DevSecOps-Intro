# Lab 7 — Container Security: Image Scanning & Deployment Hardening

**Target image:** `bkimminich/juice-shop:v19.0.0`
**Platform:** linux/arm64
**Image size:** 170 MB | **Packages:** 1004

---

## Task 1 — Image Vulnerability & Configuration Analysis

### 1.1 Vulnerability Scanning Summary

Docker Scout identified **118 vulnerabilities** across 48 packages:

| Severity | Count |
|----------|------:|
| Critical | 11 |
| High | 65 |
| Medium | 30 |
| Low | 5 |
| Unspecified | 7 |

Snyk confirmed similar findings: **53 issues** across 2 projects (OS-level deb packages + npm dependencies), with 6 critical and 47 high-severity vulnerabilities.

### 1.2 Top 5 Critical/High Vulnerabilities

| # | CVE ID | Package | Severity | CVSS | Impact |
|---|--------|---------|----------|------|--------|
| 1 | CVE-2026-22709 | vm2 3.9.17 | Critical | 9.8 | **Protection Mechanism Failure** — allows sandbox escape, enabling arbitrary code execution on the host. The vm2 library is fundamentally broken for security sandboxing. |
| 2 | CVE-2023-37903 | vm2 3.9.17 | Critical | 9.8 | **OS Command Injection** — attacker can break out of the vm2 sandbox and execute arbitrary OS commands. No fix available (library deprecated). |
| 3 | CVE-2019-10744 | lodash 2.4.2 | Critical | 9.1 | **Prototype Pollution** — attacker can modify `Object.prototype`, leading to property injection across the entire application, potentially enabling RCE or DoS. |
| 4 | CVE-2015-9235 | jsonwebtoken 0.4.0 | Critical | N/A | **Improper Input Validation** — allows authentication bypass by crafting malicious JWT tokens, enabling unauthorized access. |
| 5 | CVE-2023-46233 | crypto-js 3.3.0 | Critical | 9.1 | **Broken Cryptographic Algorithm** — uses weak PBKDF2 with MD5 by default, making encrypted data trivially decryptable. |

### 1.3 Snyk vs Docker Scout Comparison

| Aspect | Docker Scout | Snyk |
|--------|-------------|------|
| Total issues | 118 across 48 packages | 53 across 2 projects |
| Scan scope | npm dependencies only | OS packages + npm |
| OS-level findings | Not reported | 1 High (openssl CVE-2025-69421) |
| Unique insight | More granular per-version breakdown | Dependency tree paths + upgrade advice |
| Remediation | Links to advisory pages | Specific upgrade paths (e.g., "Upgrade multer@1.4.5-lts.2 to multer@2.1.1") |

Snyk provides actionable remediation paths while Docker Scout gives more comprehensive CVE coverage. Using both tools together provides the most complete picture.

### 1.4 Dockle Configuration Findings

Dockle reported minimal configuration issues for this image:

| Level | Check | Finding | Security Concern |
|-------|-------|---------|-----------------|
| SKIP | DKL-LI-0001 | Avoid empty password — failed to detect etc/shadow | Scanner could not verify password file; image may lack standard shadow file, making user auditing difficult. |
| INFO | CIS-DI-0005 | Docker Content Trust not enabled | Without content trust, pulled images are not cryptographically verified, risking supply-chain attacks via tampered images. |
| INFO | CIS-DI-0006 | No HEALTHCHECK statement found | Without a HEALTHCHECK, Docker cannot detect if the application inside the container has become unresponsive, reducing availability. |
| INFO | DKL-LI-0003 | Unnecessary files: `.DS_Store` in node_modules | Leaked macOS metadata files indicate build environment artifacts were included, increasing attack surface and image bloat. |

### 1.5 Security Posture Assessment

**Does the image run as root?**
The Dockle scan did not flag a `CIS-DI-0001` (non-root user) warning, but inspecting the Juice Shop Dockerfile shows the application uses a `juice-shop` user. However, the base Node.js image initially runs setup as root before switching users, and the container could still be started as root if `USER` is overridden.

**Recommended improvements:**
1. **Upgrade or replace vm2** — it is deprecated and fundamentally insecure; migrate to `isolated-vm` or `vm2`'s successor
2. **Update all outdated dependencies** — lodash 2.x, jsonwebtoken 0.x, and crypto-js 3.x are critically vulnerable
3. **Add a HEALTHCHECK** instruction to the Dockerfile for container orchestration resilience
4. **Enable Docker Content Trust** (`export DOCKER_CONTENT_TRUST=1`) to verify image signatures
5. **Remove unnecessary files** (`.DS_Store`) via `.dockerignore` to reduce attack surface

---

## Task 2 — Docker Host Security Benchmarking

### 2.1 CIS Docker Benchmark Summary

**Docker Bench for Security v1.3.4** was run against the Docker host (Docker 28.5.2).

| Result | Count |
|--------|------:|
| PASS | 19 |
| WARN | 16 |
| INFO | 34 |
| NOTE | 5 |
| **Total checks** | **74** |
| **Score** | **8** |

### 2.2 Analysis of Warnings/Failures

#### Section 1 — Host Configuration

| Check | Issue | Security Impact | Remediation |
|-------|-------|-----------------|-------------|
| 1.5 | Auditing not configured for Docker daemon | Docker daemon activity (start/stop, config changes) is not logged, making incident investigation difficult | Install `auditd` and add rule: `auditctl -w /usr/bin/dockerd -k docker` |
| 1.6 | No auditing for `/var/lib/docker` | Changes to container storage (image layers, volumes) go unrecorded | Add audit rule: `auditctl -w /var/lib/docker -k docker` |
| 1.7 | No auditing for `/etc/docker` | Configuration changes are untracked | Add audit rule: `auditctl -w /etc/docker -k docker` |
| 1.11 | No auditing for `daemon.json` | Daemon config modifications are invisible to security monitoring | Add audit rule: `auditctl -w /etc/docker/daemon.json -k docker` |

#### Section 2 — Docker Daemon Configuration

| Check | Issue | Security Impact | Remediation |
|-------|-------|-----------------|-------------|
| 2.1 | Unrestricted inter-container traffic on default bridge | Containers on the default bridge can communicate freely, enabling lateral movement if one is compromised | Set `"icc": false` in `daemon.json` |
| 2.6 | Docker daemon listening on TCP without TLS | Remote API calls are unencrypted and unauthenticated — anyone with network access can control Docker | Configure TLS certificates for the Docker daemon or disable TCP socket |
| 2.8 | User namespaces not enabled | Containers share UID/GID space with the host — a container root escape becomes a host root compromise | Enable `userns-remap` in `daemon.json` |
| 2.11 | No authorization plugin for Docker client | Any user with Docker socket access has full administrative control | Deploy an authorization plugin (e.g., Open Policy Agent) |
| 2.12 | No centralized/remote logging | Container logs are only stored locally and can be lost or tampered with | Configure a logging driver (e.g., `syslog`, `fluentd`) pointing to a remote log aggregator |
| 2.14 | Live restore not enabled | Containers stop when the Docker daemon restarts, causing unnecessary downtime | Set `"live-restore": true` in `daemon.json` |
| 2.15 | Userland proxy not disabled | Uses a userland proxy for port forwarding instead of kernel iptables, adding attack surface | Set `"userland-proxy": false` in `daemon.json` |
| 2.18 | Containers not restricted from acquiring new privileges | Processes inside containers can escalate privileges via setuid binaries or capabilities | Set `"no-new-privileges": true` in `daemon.json` |

#### Section 3 — Docker Daemon Configuration Files

| Check | Issue | Security Impact | Remediation |
|-------|-------|-----------------|-------------|
| 3.15 | Wrong ownership for `/var/run/docker.sock` | The Docker socket is not owned by `root:docker`, potentially allowing unauthorized users to control Docker | Run `chown root:docker /var/run/docker.sock` |

#### Section 4 — Container Images and Build Files

| Check | Issue | Security Impact | Remediation |
|-------|-------|-----------------|-------------|
| 4.5 | Docker Content Trust not enabled | Images are pulled without signature verification | `export DOCKER_CONTENT_TRUST=1` |
| 4.6 | No HEALTHCHECK in many images | Orchestrators cannot detect unresponsive containers | Add `HEALTHCHECK` instructions to Dockerfiles |

---

## Task 3 — Deployment Security Configuration Analysis

### 3.1 Configuration Comparison Table

| Setting | Default | Hardened | Production |
|---------|---------|----------|------------|
| **Capabilities dropped** | None | ALL | ALL |
| **Capabilities added** | Default set (~14 caps) | None | NET_BIND_SERVICE |
| **SecurityOpt** | None | no-new-privileges | no-new-privileges |
| **Seccomp profile** | Default | Default | Default (explicit) |
| **Memory limit** | Unlimited (8.78 GiB host) | 512 MiB | 512 MiB |
| **Memory swap** | Unlimited | Unlimited | 512 MiB (swap disabled) |
| **CPU limit** | Unlimited | 1.0 CPU | 1.0 CPU |
| **PID limit** | Unlimited | Unlimited | 100 |
| **Restart policy** | no | no | on-failure:3 |
| **HTTP status** | 200 | 200 | 200 |
| **Memory used** | 97.64 MiB / 8.78 GiB (1.09%) | 86.21 MiB / 512 MiB (16.84%) | 85.15 MiB / 512 MiB (16.63%) |

All three profiles returned HTTP 200, confirming that hardening did not break application functionality.

### 3.2 Security Measure Analysis

#### a) `--cap-drop=ALL` and `--cap-add=NET_BIND_SERVICE`

**Linux capabilities** are fine-grained privilege units that split the traditional all-or-nothing root model into ~40 distinct permissions (e.g., `CAP_NET_RAW`, `CAP_SYS_ADMIN`, `CAP_CHOWN`). By default, Docker grants containers about 14 capabilities.

**Dropping ALL capabilities** prevents the container from performing any privileged kernel operations: mounting filesystems, modifying network settings, loading kernel modules, changing file ownership, etc. This blocks an attacker who gains code execution from escalating to host-level operations.

**Adding back NET_BIND_SERVICE** allows the process to bind to privileged ports (< 1024). This is necessary if the application needs to listen on ports like 80 or 443 inside the container. In this case, Juice Shop listens on port 3000 (unprivileged), so NET_BIND_SERVICE is not strictly necessary but is included as a common production pattern.

**Trade-off:** Some applications legitimately need certain capabilities (e.g., `CAP_NET_RAW` for health checks using `ping`). Dropping all capabilities and selectively adding back only what is needed follows the principle of least privilege.

#### b) `--security-opt=no-new-privileges`

This flag sets the `PR_SET_NO_NEW_PRIV` bit on the container process, which prevents any child process from gaining more privileges than its parent — even through setuid/setgid binaries or file capabilities.

**Attack prevented:** Without this flag, an attacker who gains shell access inside a container can execute setuid binaries (like a setuid-root `su` or a custom setuid binary) to escalate to root. With `no-new-privileges`, even if a setuid binary exists, it cannot elevate privileges.

**Downsides:** Some legitimate operations fail — for example, `su` or `sudo` inside the container will not work, and applications that rely on setuid helpers (e.g., some `ping` implementations) will break. For most web applications, this has no impact.

#### c) `--memory=512m` and `--cpus=1.0`

**Without resource limits**, a single container can consume all host memory and CPU, starving other containers and the host OS itself.

**Memory limiting prevents:**
- **Denial of Service via memory exhaustion** — a memory leak or intentional memory bomb in a compromised container is contained within its 512 MiB allocation. The kernel OOM killer will terminate the container process rather than destabilizing the host.
- **Cryptomining attacks** — limits the resources available to an attacker for resource-intensive operations.

**Risk of setting limits too low:** The application may be OOM-killed during legitimate load spikes. The Juice Shop uses ~85-97 MiB at baseline, so 512 MiB provides adequate headroom (~5x). Monitoring actual usage before setting limits is essential.

#### d) `--pids-limit=100`

**A fork bomb** is a process that recursively creates copies of itself: `:(){ :|:& };:`. Each new process spawns two more, causing exponential growth that consumes all available PIDs on the system, making it unresponsive.

**PID limiting** caps the total number of processes inside a container. With `--pids-limit=100`, a fork bomb can only create 100 processes before the kernel denies further `fork()` calls, containing the attack within the container.

**Determining the right limit:** Monitor the container under peak load to find the maximum process count, then add a safety margin (e.g., 2-3x). For a Node.js application like Juice Shop, which is single-threaded with a small worker pool, 100 PIDs is generous.

#### e) `--restart=on-failure:3`

This policy automatically restarts the container when it exits with a non-zero exit code, up to a maximum of 3 attempts.

**When auto-restart is beneficial:**
- Transient failures (out-of-memory kills, temporary dependency unavailability) are recovered automatically
- Improves availability without manual intervention

**When auto-restart is risky:**
- A container in a crash loop can repeatedly trigger resource-intensive startup operations
- A compromised container keeps restarting, giving the attacker repeated execution windows
- Can mask underlying issues that need investigation

**`on-failure` vs `always`:**
- `on-failure:3` only restarts on crashes (non-zero exit) and gives up after 3 attempts — this prevents infinite restart loops
- `always` restarts even on clean exits (exit code 0) and never stops retrying — appropriate for critical services but can waste resources if the failure is permanent

### 3.3 Critical Thinking Questions

#### 1. Which profile for DEVELOPMENT? Why?

**Default** profile is best for development. Developers need unrestricted access for debugging: attaching debuggers, inspecting processes, installing packages, and running diagnostic tools. Resource limits can interfere with profiling and testing. Security hardening in development adds friction without meaningful protection since the environment is already trusted and isolated.

#### 2. Which profile for PRODUCTION? Why?

**Production** profile is the clear choice. It implements defense-in-depth with multiple independent security layers: capability dropping prevents privilege escalation, `no-new-privileges` blocks setuid attacks, resource limits prevent DoS, PID limits stop fork bombs, and the restart policy provides resilience. The fact that all three profiles returned HTTP 200 proves that these restrictions do not impact functionality.

#### 3. What real-world problem do resource limits solve?

Resource limits solve the **noisy neighbor problem** and prevent **denial-of-service**. In a shared environment (e.g., Kubernetes cluster, multi-tenant Docker host), one misbehaving or compromised container can consume all available memory and CPU, causing other containers and the host to become unresponsive. Resource limits provide isolation guarantees — each container gets a predictable resource budget, and failures are contained within that budget. This is essential for SLA compliance and cost predictability.

#### 4. If an attacker exploits Default vs Production, what actions are blocked in Production?

In the **Default** profile, an attacker with code execution can:
- Escalate to root via setuid binaries
- Use capabilities like `CAP_NET_RAW` to sniff network traffic
- Use `CAP_SYS_PTRACE` to attach to other processes
- Fork bomb to crash the host
- Consume all host memory to DoS other services
- Allocate unlimited CPU for cryptomining
- Mount filesystems or modify network configuration

In the **Production** profile, all of the above are blocked:
- `--cap-drop=ALL` removes all kernel capabilities
- `--security-opt=no-new-privileges` prevents setuid escalation
- `--memory=512m --memory-swap=512m` caps memory usage (swap disabled prevents bypassing memory limit)
- `--cpus=1.0` limits CPU usage
- `--pids-limit=100` prevents fork bombs
- The attacker is confined to a minimal, resource-constrained sandbox

#### 5. What additional hardening would you add?

1. **`--read-only`** — Mount the root filesystem as read-only to prevent attackers from modifying application code or dropping malicious files. Use `tmpfs` mounts for directories that need writes (e.g., `/tmp`).
2. **`--user=1000:1000`** — Explicitly run as a non-root user to reduce impact of container escape vulnerabilities.
3. **`--network=custom-bridge`** with explicit network policies — Isolate the container on a dedicated network and control ingress/egress.
4. **Custom seccomp profile** — The default seccomp profile blocks ~44 syscalls, but a custom profile tailored to Node.js can block many more unnecessary syscalls (e.g., `mount`, `reboot`, `kexec_load`).
5. **`--tmpfs /tmp:rw,noexec,nosuid,size=64m`** — If using `--read-only`, mount `/tmp` as a size-limited tmpfs with `noexec` to prevent binary execution from temporary files.
6. **Health checks** — Add `--health-cmd` to detect application-level failures, enabling the orchestrator to replace unresponsive containers.
