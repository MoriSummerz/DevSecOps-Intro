# Lab 5 — Security Analysis: SAST & DAST of OWASP Juice Shop

## Task 1 — Static Application Security Testing with Semgrep

### 1.1 SAST Tool Effectiveness

Semgrep was run against the OWASP Juice Shop v19.0.0 source code using two rulesets: `p/security-audit` and
`p/owasp-top-ten`.

| Metric                      | Value |
|-----------------------------|-------|
| Total findings              | 25    |
| Files scanned with findings | 20    |
| ERROR (High) severity       | 7     |
| WARNING (Medium) severity   | 18    |
| Unique rule IDs triggered   | 11    |

**Vulnerability types detected:**

| Vulnerability Type             | Count | CWE     |
|--------------------------------|-------|---------|
| SQL Injection (Sequelize)      | 6     | CWE-89  |
| Unquoted HTML Attributes (XSS) | 4     | CWE-79  |
| Directory Listing Enabled      | 4     | CWE-548 |
| Path Traversal (sendFile)      | 4     | CWE-73  |
| Script Tag XSS                 | 2     | CWE-79  |
| Open Redirect                  | 2     | CWE-601 |
| Code Injection (eval)          | 1     | CWE-94  |
| Hardcoded JWT Secret           | 1     | CWE-798 |
| Raw HTML Formatting            | 1     | CWE-79  |

Semgrep effectively covers OWASP Top 10 categories including Injection (A03:2021), Broken Access Control (A01:2021),
Security Misconfiguration (A05:2021), and Identification & Authentication Failures (A07:2021). Its pattern-based
approach excels at catching code-level anti-patterns such as string concatenation in SQL queries, use of `eval()`, and
hardcoded secrets.

### 1.2 Critical Vulnerability Analysis — Top 5 Findings

#### 1. SQL Injection in Login Route

| Field        | Value                     |
|--------------|---------------------------|
| **Type**     | SQL Injection (Sequelize) |
| **File**     | `src/routes/login.ts`     |
| **Line**     | 34                        |
| **Severity** | ERROR                     |
| **CWE**      | CWE-89                    |

User-supplied email and password are directly concatenated into a Sequelize query string, allowing an attacker to bypass
authentication or extract data via crafted input like `' OR 1=1--`.

#### 2. SQL Injection in Product Search

| Field        | Value                     |
|--------------|---------------------------|
| **Type**     | SQL Injection (Sequelize) |
| **File**     | `src/routes/search.ts`    |
| **Line**     | 23                        |
| **Severity** | ERROR                     |
| **CWE**      | CWE-89                    |

The search query parameter `q` is interpolated directly into a SQL query without parameterization, enabling union-based
and boolean-blind SQL injection.

#### 3. Remote Code Execution via eval()

| Field        | Value                       |
|--------------|-----------------------------|
| **Type**     | Code Injection              |
| **File**     | `src/routes/userProfile.ts` |
| **Line**     | 62                          |
| **Severity** | ERROR                       |
| **CWE**      | CWE-94                      |

User-controllable data is passed to `eval()`, allowing arbitrary JavaScript execution on the server — a critical RCE
vulnerability.

#### 4. SQL Injection in Challenge Code (dbSchemaChallenge)

| Field        | Value                                              |
|--------------|----------------------------------------------------|
| **Type**     | SQL Injection (Sequelize)                          |
| **File**     | `src/data/static/codefixes/dbSchemaChallenge_1.ts` |
| **Line**     | 5                                                  |
| **Severity** | ERROR                                              |
| **CWE**      | CWE-89                                             |

Vulnerable search implementation with direct string interpolation in SQL query.

#### 5. SQL Injection in Union Challenge Code

| Field        | Value                                                       |
|--------------|-------------------------------------------------------------|
| **Type**     | SQL Injection (Sequelize)                                   |
| **File**     | `src/data/static/codefixes/unionSqlInjectionChallenge_1.ts` |
| **Line**     | 6                                                           |
| **Severity** | ERROR                                                       |
| **CWE**      | CWE-89                                                      |

Template literal used in SQL query construction, allowing union-based SQL injection attacks.

---

## Task 2 — Dynamic Application Security Testing with Multiple Tools

### 2.1 Authenticated vs Unauthenticated Scanning

| Metric                    | Unauthenticated | Authenticated | Difference |
|---------------------------|-----------------|---------------|------------|
| Total alerts              | 12              | 14            | +2         |
| High severity             | 0               | 1             | +1         |
| Medium severity           | 2               | 5             | +3         |
| Low severity              | 6               | 4             | -2         |
| Informational             | 4               | 4             | 0          |
| Unique URLs with findings | 15              | 22            | +7 (+47%)  |

**Admin/authenticated endpoints discovered only in authenticated scan:**

- `POST /rest/user/login` — SQL Injection detected on the `email` parameter
- `/rest/admin/application-configuration` — admin panel configuration
- Session-bound endpoints for basket, orders, and payment processing
- Anti-clickjacking issues on authenticated pages

**Why authenticated scanning matters:**

Unauthenticated scanning only tests the public attack surface — login pages, static content, and openly accessible API
endpoints. Authenticated scanning reveals the full application: admin panels, user-specific workflows (shopping cart,
profile, order history), and endpoints protected by session tokens. In this scan, authentication uncovered **1 HIGH
severity SQL Injection** alert that was invisible to the unauthenticated scan, along with 3 additional medium-severity
issues (anti-clickjacking, session ID in URL rewrite, HTTP-only site). The 47% increase in URLs with findings
demonstrates that a significant portion of the attack surface is hidden behind authentication.

### 2.2 Tool Comparison Matrix

| Tool                    | Total Findings    | High | Medium | Low | Info | Best Use Case                                              |
|-------------------------|-------------------|------|--------|-----|------|------------------------------------------------------------|
| **ZAP** (authenticated) | 14 alert types    | 1    | 5      | 4   | 4    | Comprehensive web app scanning with authentication support |
| **Nuclei**              | 0                 | 0    | 0      | 0   | 0    | Fast template-based scanning for known CVEs (not run)      |
| **Nikto**               | 82                | —    | —      | —   | —    | Web server misconfiguration and information disclosure     |
| **SQLmap**              | 1 injection point | 1    | —      | —   | —    | Deep SQL injection analysis and database extraction        |

> **Note:** Nuclei scan was not completed in this lab run. Nikto does not categorize findings by traditional severity
> levels — it reports individual checks as pass/fail.

### 2.3 Tool-Specific Strengths

#### ZAP — Comprehensive Web Application Scanner

ZAP provides the most complete picture of web application security posture. Its strength lies in combining automated
crawling (spider + AJAX spider) with both passive and active scanning, plus built-in authentication support.

**Example findings:**

1. **SQL Injection** (HIGH) — Detected injection vulnerability in `/rest/products/search?q=` by sending `'(` as payload
   and observing a 500 Internal Server Error response
2. **Content Security Policy Header Not Set** (MEDIUM) — Identified missing CSP headers across 5 endpoints, which could
   allow XSS attacks and data exfiltration

#### Nikto — Server Misconfiguration Scanner

Nikto excels at identifying web server-level issues: missing security headers, exposed directories, information
disclosure through server banners, and known server vulnerabilities. It tested 82 checks against the Juice Shop server.

**Example findings:**

1. **Wildcard Access-Control-Allow-Origin** — The server returns `Access-Control-Allow-Origin: *`, allowing any domain
   to make cross-origin requests and potentially steal user data
2. **Exposed `/ftp/` directory** — Listed in robots.txt and returning HTTP 200, exposing potentially sensitive files to
   anyone who requests them

#### SQLmap — SQL Injection Specialist

SQLmap provides the deepest analysis of SQL injection vulnerabilities. While ZAP detected the injection point, SQLmap
confirmed it was exploitable and identified the exact technique and backend database.

**Example findings:**

1. **Boolean-based blind SQL injection** in `/rest/products/search?q=` — Confirmed exploitable with payload
   `') AND 8946=8946 AND ('TZOt' LIKE 'TZOt`. Identified backend DBMS as **SQLite**
2. **Database extraction capability** — SQLmap's `--dump` flag enables extraction of full database contents including
   user tables with emails and bcrypt password hashes

#### Nuclei — Template-Based CVE Scanner

Nuclei uses community-maintained templates to rapidly scan for known CVEs and misconfigurations. While not successfully
run in this session, it typically excels at:

- Fast detection of known vulnerabilities using signature-based matching
- Low false-positive rate due to verification-based templates
- Broad coverage of CVE databases with regular template updates

---

## Task 3 — SAST/DAST Correlation and Security Assessment

### 3.1 SAST vs DAST Comparison

| Approach | Tool(s)              | Total Findings                    |
|----------|----------------------|-----------------------------------|
| SAST     | Semgrep              | 25 code-level findings            |
| DAST     | ZAP + Nikto + SQLmap | 97 runtime findings (14 + 82 + 1) |

While DAST produced a higher raw count (driven largely by Nikto's 82 server-level checks), SAST findings were generally
higher severity — 7 ERROR-level issues vs 1 HIGH from ZAP and 1 confirmed injection from SQLmap.

### Vulnerabilities Found ONLY by SAST

1. **Hardcoded JWT Secret** (`src/lib/insecurity.ts`) — A static secret key used for JWT signing was embedded directly
   in source code. This cannot be detected at runtime because the application functions correctly with the hardcoded
   value; only source code review reveals the practice is insecure.

2. **Remote Code Execution via `eval()`** (`src/routes/userProfile.ts`) — Semgrep identified that user-controllable
   input reaches an `eval()` call. DAST tools did not trigger this code path during scanning because it requires a
   specific sequence of user profile update actions.

3. **Path Traversal in File Serving** (`src/routes/fileServer.ts`, `keyServer.ts`, `logfileServer.ts`,
   `quarantineServer.ts`) — Semgrep detected 4 instances of `res.sendFile()` with user-controlled paths. DAST tools did
   not test these specific endpoints with traversal payloads during automated scanning.

### Vulnerabilities Found ONLY by DAST

1. **Missing Security Headers** (CSP, X-Content-Type-Options, Anti-clickjacking) — These are deployment-time
   configuration issues that only manifest when the application is running. Source code analysis cannot determine what
   headers the HTTP server will actually send in production.

2. **Wildcard CORS Configuration** (`Access-Control-Allow-Origin: *`) — Nikto detected that the running server accepts
   cross-origin requests from any domain. This runtime behavior depends on Express middleware configuration that
   Semgrep's pattern matching did not flag.

3. **Session Management Issues** (Session ID in URL rewrite, private IP disclosure) — These are runtime behaviors
   observable only by interacting with the live application. ZAP detected session tokens appearing in URLs and internal
   IP addresses leaking in HTTP responses.

### Why Each Approach Finds Different Things

**SAST** analyzes source code without executing it. It excels at finding code-level anti-patterns — dangerous function
calls (`eval`), insecure data flows (user input → SQL query), and embedded secrets. However, it cannot observe how the
application behaves in production: what headers the server sends, how sessions are managed, or what the actual HTTP
responses contain.

**DAST** interacts with the running application as an attacker would. It detects configuration and deployment issues (
missing headers, CORS misconfig), runtime behaviors (session handling, error messages leaking information), and confirms
whether code-level vulnerabilities are actually exploitable. However, it only tests code paths it can reach through its
crawling — it may miss deeply nested logic, rarely triggered code paths, or vulnerabilities that require specific
preconditions.

**Complementary coverage:** The SQL injection in `/rest/products/search` was found by both approaches — Semgrep
identified the vulnerable code pattern in `search.ts:23`, and both ZAP and SQLmap confirmed it was exploitable at
runtime. This overlap validates both tools, while the non-overlapping findings demonstrate why a comprehensive DevSecOps
pipeline needs both SAST (shift-left, early developer feedback) and DAST (pre-production validation of deployed security
posture).
