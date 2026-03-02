# Lab 4 Submission — SBOM Generation & Software Composition Analysis

---

## Task 1 — SBOM Generation with Syft and Trivy (4 pts)

### 1.1 Setup

All tools were run as Docker containers against `bkimminich/juice-shop:v19.0.0`:

```bash
docker pull anchore/syft:latest
docker pull aquasec/trivy:latest
docker pull anchore/grype:latest
```

SBOM generation commands produced outputs in both JSON and human-readable table formats for each tool.

### 1.2 Package Type Distribution

| Package Type | Syft Count | Trivy Count |
|---|---:|---:|
| npm / Node.js | 1128 | 1125 |
| deb (OS packages) | 10 | 10 |
| binary | 1 | 0 |
| **Total** | **1139** | **1135** |

Syft detected 1139 total components across 3 package types (npm, deb, binary), while Trivy found 1135 across 2 types. Syft uniquely identified the Node.js runtime binary (`node@22.18.0`) as a separate component, which proved important — Grype later flagged CVEs against it.

### 1.3 Dependency Discovery Analysis

From the package overlap analysis:

| Metric | Count |
|---|---:|
| Packages detected by both tools | 1126 |
| Packages only detected by Syft | 13 |
| Packages only detected by Trivy | 9 |

**Syft-only packages (13):** Included the Node.js binary, OS-level packages with full debian version strings (e.g., `libssl3@3.0.17-1~deb12u2`), and several test/internal npm packages (`baz@UNKNOWN`, `browser_field@UNKNOWN`, `false_main@UNKNOWN`, `invalid_main@UNKNOWN`, `hashids-esm@UNKNOWN`). Syft is more aggressive at cataloging packages even without proper version metadata.

**Trivy-only packages (9):** Included `portscanner@2.2.0` and `toposort-class@1.0.1` (npm packages missed by Syft), plus the same OS packages but with truncated version strings (e.g., `libssl3@3.0.17` vs Syft's `libssl3@3.0.17-1~deb12u2`). The version string difference is the primary cause of overlap mismatches for OS packages.

**Verdict:** Syft has a slight edge in breadth — it catalogs binaries and even packages with unknown versions. Trivy focuses on packages it can confidently identify. The OS package version string normalization difference means the overlap count underestimates actual agreement.

### 1.4 License Discovery Analysis

| License Category | Syft | Trivy (OS + Node.js) |
|---|---:|---:|
| Unique license types | 32 | 28 |
| MIT occurrences | 890 | 878 |
| ISC occurrences | 143 | 143 |
| Apache-2.0 occurrences | 15 | 13 |
| BSD-3-Clause | 16 | 14 |
| LGPL-3.0 | 19 | 19 |

Syft found 4 more unique license types than Trivy. The extra licenses include `Apache2` (non-SPDX variant), `BSD` (unqualified), `GPL-1`, and `GPL-1+` — these are non-standard identifiers that Trivy normalizes to SPDX format (e.g., `GPL-1.0-only`, `GPL-1.0-or-later`).

**Verdict:** Syft provides raw license strings as declared in package metadata, while Trivy normalizes them to SPDX identifiers. Trivy's approach is better for automated license compliance checks; Syft's is more faithful to the original metadata.

---

## Task 2 — Software Composition Analysis with Grype and Trivy (3 pts)

### 2.1 SCA Tool Comparison

| Metric | Grype | Trivy |
|---|---:|---:|
| Critical | 11 | 10 |
| High | 88 | 81 |
| Medium | 32 | 34 |
| Low | 3 | 18 |
| Negligible/Unknown | 12 | 0 |
| **Total** | **146** | **143** |

Grype found slightly more total vulnerability matches (146 vs 143). Grype reports a `Negligible` severity category (12 findings for libc6 and gcc CVEs) that Trivy does not use. Trivy classifies more findings as `Low` (18 vs 3), suggesting different severity mapping for the same CVEs.

**CVE overlap:**
- Unique CVEs found by Grype: 95
- Unique CVEs found by Trivy: 91
- Common CVEs (found by both): 26

The low overlap (26 common CVEs out of ~160 unique total) is notable. This is partly because each tool reports different CVE aliases (GHSA vs CVE identifiers), and partly because their vulnerability databases have different coverage. Grype uses the Anchore vulnerability database, while Trivy uses its own aggregated database.

### 2.2 Critical Vulnerabilities Analysis — Top 5

| # | Package | CVE/GHSA | Severity | EPSS | Remediation |
|---|---|---|---|---:|---|
| 1 | `vm2@3.9.17` | GHSA-whpj-8f3w-67p5 | Critical | 69.9% | **Remove entirely** — vm2 is abandoned. Migrate to `isolated-vm` or Node.js `vm` with `--experimental-vm-modules`. |
| 2 | `vm2@3.9.17` | GHSA-g644-9gfx-q4q4 | Critical | 39.2% | Same as above — sandbox escape allowing arbitrary code execution. |
| 3 | `jsonwebtoken@0.1.0` | GHSA-c7hr-j4mj-j2w6 | Critical | 32.5% | Upgrade to `jsonwebtoken@9.0.0+`. The current version (0.1.0) has no signature verification. |
| 4 | `crypto-js@3.3.0` | GHSA-xwcq-pm8m-c4vf | Critical | 0.8% | Upgrade to `crypto-js@4.2.0+`. PBKDF2 implementation uses insecure defaults. |
| 5 | `lodash@2.4.2` | GHSA-jf85-cpcp-j695 | Critical | 1.2% | Upgrade to `lodash@4.17.21+`. Prototype pollution vulnerability. |

The most dangerous finding is **vm2** — it has multiple sandbox escape vulnerabilities and the project is officially unmaintained. The EPSS score of 69.9% means there is a very high probability of active exploitation.

### 2.3 License Compliance Assessment

| License | Risk Level | Package Count | Concern |
|---|---|---:|---|
| LGPL-3.0 | High | 19 | Web3 libraries — copyleft, requires source disclosure if modified |
| GPL-2.0 | High | 6+ | `fuzzball`, OS packages — strong copyleft |
| GPL-3.0 | Medium | 4 | GCC runtime — typically covered by runtime exception |
| WTFPL | Low | 2 | Non-standard, may lack legal clarity |
| Unlicense | Low | 2 | Public domain dedication, generally permissive |

**Recommendations:**
- The 19 LGPL-3.0 packages are all `web3-*` libraries. If the application links dynamically (standard for Node.js), LGPL compliance is manageable, but modifications to those libraries must be shared.
- The `fuzzball@1.4.0` package uses GPL-2.0, which is the most restrictive license in the dependency tree for application-level code. Evaluate if it can be replaced with an MIT/ISC alternative.
- ~95% of dependencies use permissive licenses (MIT, ISC, BSD), which is favorable.

### 2.4 Additional Security Features — Secrets Scanning

Trivy's secrets scanner found **no exposed secrets** in the Juice Shop image. All scanned targets (debian OS layer, node package manifests) returned clean results. This is expected for a published open-source Docker image — secrets would typically be injected at runtime via environment variables.

---

## Task 3 — Toolchain Comparison: Syft+Grype vs Trivy All-in-One (3 pts)

### 3.1 Accuracy Analysis

#### Package Detection

| Metric | Value |
|---|---:|
| Common packages | 1126 |
| Syft-only | 13 |
| Trivy-only | 9 |
| Agreement rate | ~98.1% |

The tools agree on the vast majority of packages. Discrepancies are largely due to:
1. **Version string formatting** — OS packages use different version representations
2. **Binary detection** — Syft detects standalone binaries (Node.js runtime); Trivy does not
3. **Edge cases** — Syft catalogs packages with `UNKNOWN` versions that Trivy skips

#### Vulnerability Detection

| Metric | Grype | Trivy |
|---|---:|---:|
| Unique CVE IDs | 95 | 91 |
| Common CVEs | 26 | 26 |
| Grype-only CVEs | 69 | — |
| Trivy-only CVEs | — | 65 |

The low overlap is significant — running both tools provides substantially better coverage than either alone. The combined unique CVE count (~160) is ~70% higher than either tool individually.

### 3.2 Tool Strengths and Weaknesses

| Dimension | Syft + Grype | Trivy |
|---|---|---|
| **SBOM depth** | Richer metadata, binary detection, raw license strings | Cleaner SPDX-normalized output |
| **Vulnerability coverage** | More CVEs found (95 vs 91), includes EPSS scores and risk ratings | Fewer but well-curated findings, better Low-severity coverage |
| **Severity mapping** | Includes `Negligible` category | Stricter 4-tier severity (CRITICAL/HIGH/MEDIUM/LOW) |
| **Additional scanners** | SBOM-only (pair with Grype for vulns) | Built-in: vuln, secret, license, misconfiguration scanning |
| **Output formats** | Syft-native JSON, CycloneDX, SPDX | JSON, table, SARIF, CycloneDX, SPDX |
| **Speed** | Two separate scans required | Single scan covers everything |
| **Database** | Anchore vulnerability feed | Trivy DB (aggregated from NVD, GitHub Advisory, etc.) |

### 3.3 Use Case Recommendations

**Choose Syft + Grype when:**
- You need detailed SBOM generation as a primary artifact (e.g., for NTIA/EO compliance)
- Binary-level detection matters (e.g., embedded runtimes, compiled dependencies)
- You want EPSS scores and risk ratings in vulnerability output
- You operate in a specialized Anchore ecosystem with policy-as-code

**Choose Trivy when:**
- You need a single tool for multiple security scanning types (vulns, secrets, licenses, IaC)
- CI/CD pipeline simplicity is a priority — one Docker image, one command
- You scan diverse targets beyond container images (filesystems, git repos, Kubernetes)
- Team onboarding speed matters — lower learning curve

**Best practice — use both:**
As demonstrated by the low CVE overlap (only 26 common out of ~160 total), running both tools provides significantly better vulnerability coverage. A practical approach is to use Trivy as the primary CI/CD gate (fast, all-in-one) and Syft+Grype for periodic deep analysis and SBOM compliance artifacts.

### 3.4 Integration Considerations

| Aspect | Syft + Grype | Trivy |
|---|---|---|
| **CI/CD setup** | 2 steps: generate SBOM, then scan | 1 step: scan directly |
| **GitHub Actions** | Separate actions for each tool | Single `aquasecurity/trivy-action` |
| **Policy enforcement** | Grype supports `.grype.yaml` for ignore rules | Trivy supports `.trivyignore` and Rego policies |
| **SBOM storage** | Syft SBOM can be stored/versioned independently | SBOM generation is a secondary feature |
| **Maintenance** | Two tools to update and monitor | One tool to maintain |
| **Community** | Active (Anchore-backed) | Very active (Aqua Security-backed, CNCF project) |