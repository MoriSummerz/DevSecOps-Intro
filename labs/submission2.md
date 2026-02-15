# Lab 2 — Threat Modeling with Threagile

## Task 1 — Threagile Baseline Model (6 pts)

### 1.1 Generated Outputs

Baseline threat model was generated using:

```bash
docker run --rm -v "$(pwd)":/app/work threagile/threagile \
  -model /app/work/labs/lab2/threagile-model.yaml \
  -output /app/work/labs/lab2/baseline \
  -generate-risks-excel=false -generate-tags-excel=false
```

### 1.2 Verified Output Files

| File | Description |
|---|---|
| `report.pdf` | Full PDF report with embedded diagrams |
| `data-flow-diagram.png` | Data-flow diagram |
| `data-asset-diagram.png` | Data-asset diagram |
| `risks.json` | Machine-readable risk list (23 risks) |
| `stats.json` | Risk severity statistics |
| `technical-assets.json` | Technical asset inventory |

### 1.3 Risk Statistics Summary

| Severity | Count |
|---|---:|
| Critical | 0 |
| Elevated | 4 |
| High | 0 |
| Medium | 14 |
| Low | 5 |
| **Total** | **23** |

### 1.4 Risk Ranking Methodology

Composite scores are calculated using the following weights:

| Factor | Scale |
|---|---|
| Severity | critical (5), elevated (4), high (3), medium (2), low (1) |
| Likelihood | very-likely (4), likely (3), possible (2), unlikely (1) |
| Impact | high (3), medium (2), low (1) |

**Composite Score** = `Severity × 100 + Likelihood × 10 + Impact`

This weighting ensures severity dominates the ranking, with likelihood and impact serving as tiebreakers. A higher composite score indicates a more urgent risk.

### 1.5 Top 5 Risks

| Rank | Risk | Severity | Category | Asset | Likelihood | Impact | Score |
|---:|---|---|---|---|---|---|---:|
| 1 | Unencrypted Communication — Direct to App (auth data) | Elevated | unencrypted-communication | User Browser → Juice Shop | Likely | High | **433** |
| 2 | Unencrypted Communication — To App (proxy path) | Elevated | unencrypted-communication | Reverse Proxy → Juice Shop | Likely | Medium | **432** |
| 3 | Missing Authentication — To App | Elevated | missing-authentication | Reverse Proxy → Juice Shop | Likely | Medium | **432** |
| 4 | Cross-Site Scripting (XSS) | Elevated | cross-site-scripting | Juice Shop Application | Likely | Medium | **432** |
| 5 | Cross-Site Request Forgery (CSRF) | Medium | cross-site-request-forgery | Juice Shop Application | Very-likely | Low | **241** |

### 1.6 Analysis of Critical Security Concerns

**1. Unencrypted Communication (Score 433/432):** The highest-scoring risks both involve HTTP traffic in the clear. The direct browser-to-app link exposes authentication credentials (tokens, session IDs) to network-level attackers. The proxy-to-app link similarly transmits data without TLS, making the reverse proxy path equally vulnerable to eavesdropping within the internal network.

**2. Missing Authentication on Reverse Proxy Path (Score 432):** The communication link from the reverse proxy to the Juice Shop application lacks authentication. This means an attacker with network access could bypass the proxy entirely and access the application directly, defeating any access control the proxy provides.

**3. Cross-Site Scripting (Score 432):** Juice Shop is a web application that accepts user input and renders it in the browser. Without proper output encoding and input sanitization, it is vulnerable to stored and reflected XSS, which can lead to session hijacking and data theft.

**4. Cross-Site Request Forgery (Score 241):** Despite a lower composite score, CSRF is flagged with "very-likely" exploitation likelihood. This reflects Juice Shop's lack of CSRF tokens, allowing attackers to craft forged requests that execute actions on behalf of authenticated users.

### 1.7 Diagrams

The generated diagrams are located in `labs/lab2/baseline/`:

- **Data-Flow Diagram** (`data-flow-diagram.png`): Shows User Browser, Reverse Proxy, Juice Shop Application, Persistent Storage, and Webhook Endpoint with communication links between them.
- **Data-Asset Diagram** (`data-asset-diagram.png`): Maps data assets (Customer Orders, Credentials, Tokens & Sessions) to their processing and storage locations.

---

## Task 2 — HTTPS Variant & Risk Comparison (4 pts)

### 2.1 Model Changes

The following security controls were applied in `labs/lab2/threagile-model.secure.yaml`:

| Change | Location | Baseline | Secure |
|---|---|---|---|
| Enable HTTPS (browser → app) | User Browser → communication_links → Direct to App | `protocol: http` | `protocol: https` |
| Enable HTTPS (proxy → app) | Reverse Proxy → communication_links → To App | `protocol: http` | `protocol: https` |
| Enable storage encryption | Persistent Storage | `encryption: none` | `encryption: transparent` |

### 2.2 Secure Variant Generation

```bash
docker run --rm -v "$(pwd)":/app/work threagile/threagile \
  -model /app/work/labs/lab2/threagile-model.secure.yaml \
  -output /app/work/labs/lab2/secure \
  -generate-risks-excel=false -generate-tags-excel=false
```

Secure variant risk statistics: **20 risks** (2 elevated, 13 medium, 5 low) — down from 23 in baseline.

### 2.3 Risk Category Delta Table

| Category | Baseline | Secure | Δ |
|---|---:|---:|---:|
| container-baseimage-backdooring | 1 | 1 | 0 |
| cross-site-request-forgery | 2 | 2 | 0 |
| cross-site-scripting | 1 | 1 | 0 |
| missing-authentication | 1 | 1 | 0 |
| missing-authentication-second-factor | 2 | 2 | 0 |
| missing-build-infrastructure | 1 | 1 | 0 |
| missing-hardening | 2 | 2 | 0 |
| missing-identity-store | 1 | 1 | 0 |
| missing-vault | 1 | 1 | 0 |
| missing-waf | 1 | 1 | 0 |
| server-side-request-forgery | 2 | 2 | 0 |
| **unencrypted-asset** | **2** | **1** | **-1** |
| **unencrypted-communication** | **2** | **0** | **-2** |
| unnecessary-data-transfer | 2 | 2 | 0 |
| unnecessary-technical-asset | 2 | 2 | 0 |

**Total risk count: 23 → 20 (Δ = -3)**

### 2.4 Delta Run Explanation

#### Changes Made

1. **HTTPS on communication links**: Both the direct browser-to-app link and the reverse-proxy-to-app link were switched from `http` to `https`.
2. **Transparent encryption on Persistent Storage**: The database/file store encryption was changed from `none` to `transparent`.

#### Observed Results

- **unencrypted-communication** dropped from 2 → 0 (Δ = -2): Both unencrypted communication risks were fully eliminated by enabling HTTPS on both links. This removed the two highest-scoring risks from the baseline (scores 433 and 432).
- **unencrypted-asset** dropped from 2 → 1 (Δ = -1): Enabling transparent encryption on Persistent Storage resolved its unencrypted-asset risk. The Juice Shop Application itself remains flagged because application-level encryption was not configured.
- All other categories remained unchanged — HTTPS and storage encryption do not address application-layer vulnerabilities (XSS, CSRF, SSRF) or architectural gaps (missing authentication, missing WAF, missing vault).

#### Why These Changes Reduced Risks

- **HTTPS** provides transport-layer encryption (TLS), which directly addresses the "unencrypted-communication" category. Threagile recognizes the `https` protocol as satisfying the encrypted transport requirement, so both flagged communication links were cleared.
- **Transparent encryption** on the storage asset means data at rest is encrypted, which resolves the "unencrypted-asset" finding for that specific asset. However, Juice Shop's in-memory/process-level data remains unencrypted, so one unencrypted-asset risk persists.
- The remaining 20 risks require different mitigations (input validation for XSS/CSRF, authentication controls, WAF deployment, secret management) that are outside the scope of the transport and storage encryption changes.

### 2.5 Diagram Comparison

| Aspect | Baseline | Secure |
|---|---|---|
| Data-flow diagram | `labs/lab2/baseline/data-flow-diagram.png` | `labs/lab2/secure/data-flow-diagram.png` |
| Data-asset diagram | `labs/lab2/baseline/data-asset-diagram.png` | `labs/lab2/secure/data-asset-diagram.png` |

Key visual differences in the secure variant diagrams:
- Communication links between User Browser → Juice Shop and Reverse Proxy → Juice Shop now show as encrypted (HTTPS) connections
- Persistent Storage is marked with encryption enabled
- The overall risk coloring/indicators on affected assets reflect the reduced risk posture
