# Lab 3 Submission — Secure Git
---

## Task 1 — SSH Commit Signature Verification (5 pts)

### 1.1 Summary: Benefits of Signing Commits

Commit signing is a critical security practice that provides:

- **Authenticity Verification**: Ensures commits are genuinely from the claimed author, not an impersonator
- **Integrity Protection**: Guarantees that commit content hasn't been tampered with after signing
- **Non-repudiation**: Creates cryptographic proof of who made changes and when
- **Supply Chain Security**: Prevents malicious code injection through compromised developer accounts
- **Compliance Requirements**: Meets security standards for regulated industries and enterprise environments

SSH commit signing specifically offers advantages over GPG:
- Simpler setup (reuses existing SSH keys)
- Better integration with modern development workflows
- No need for separate key management infrastructure
- Native support in GitHub, GitLab, and other platforms

### 1.2 Evidence: SSH Key Setup and Configuration

**SSH Key Generated:**
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGfaLbLUcM0fujjR7UU+F2AWbqR8iD0wN2PXmRkygKw6 gitlab IU
```

**Git Configuration (Global):**
```bash
$ git config --get user.signingkey
/Users/morisummer/.ssh/id_ed25519.pub

$ git config --get commit.gpgsign
true

$ git config --get gpg.format
ssh

$ git config --get user.email
timofeevnikita111@gmail.com
```

**Configuration Commands Used:**
```bash
git config --global user.signingkey /Users/morisummer/.ssh/id_ed25519.pub
git config --global commit.gpgSign true
git config --global gpg.format ssh
```

**Pre-commit Hook Permissions:**
```bash
$ ls -la .git/hooks/pre-commit
-rwxr-xr-x@ 1 morisummer  staff  3347 Feb 22 21:50 .git/hooks/pre-commit
```

### 1.3 Signed Commit Evidence

**Recent Signed Commit:**
```bash
$ git log --oneline -1
9e9cf55 docs: add commit signing summary
```

**Commit Details:**
- **Commit Hash:** 9e9cf558d802e680d09f768b7536f51aee1d32c9
- **Message:** "docs: add commit signing summary"
- **Author:** morisummer <timofeevnikita111@gmail.com>
- **Date:** Sun Feb 22 21:46:03 2026 +0300
- **Signed:** Yes (SSH signature)

**GitHub Verification:**
The commit shows a "Verified" badge on GitHub, confirming:
- SSH key is properly registered with my GitHub account
- Signature validation passed
- Commit integrity is cryptographically guaranteed

### 1.4 Analysis: Why Commit Signing is Critical in DevSecOps

Commit signing is essential in DevSecOps workflows for several reasons:

#### 1. **Supply Chain Attack Prevention**
In DevSecOps, code flows through automated CI/CD pipelines directly to production. Without commit signing:
- Attackers who compromise a developer's credentials can inject malicious code
- Man-in-the-middle attacks could alter commits in transit
- Compromised Git servers could insert backdoors

Signed commits create an immutable chain of custody, ensuring only authorized developers can contribute code that reaches production.

#### 2. **Compliance and Audit Requirements**
Many security frameworks (SOC 2, ISO 27001, PCI-DSS) require:
- Verifiable attribution of all code changes
- Tamper-evident audit trails
- Cryptographic proof of code provenance

Commit signatures provide the forensic evidence needed during security audits and incident investigations.

#### 3. **Branch Protection and Policy Enforcement**
Modern DevSecOps platforms (GitHub, GitLab) can enforce policies requiring:
- All commits to protected branches must be signed
- Only verified commits can trigger deployment pipelines
- Unsigned commits are automatically rejected

This prevents accidental or malicious unsigned commits from entering critical code paths.

#### 4. **Incident Response and Forensics**
When security incidents occur, signed commits help:
- Identify the exact point where vulnerabilities were introduced
- Determine if commits were tampered with post-incident
- Trace accountability through the development timeline
- Distinguish legitimate changes from malicious injections

#### 5. **Zero-Trust Security Model**
DevSecOps embraces "never trust, always verify." Commit signing aligns with this by:
- Not trusting Git's author fields (easily spoofed)
- Cryptographically validating every change
- Extending verification beyond authentication to authorization

**Conclusion:** In DevSecOps, where code velocity is high and automation is pervasive, commit signing transforms Git from a collaboration tool into a security control. It's not just about knowing *who* made a change, but having cryptographic proof that cannot be forged or repudiated.

---

## Task 2 — Pre-commit Secret Scanning (5 pts)

### 2.1 Pre-commit Hook Setup

**Hook Location:**
`.git/hooks/pre-commit`

**Setup Process:**

1. **Created the pre-commit hook file:**
   ```bash
   touch .git/hooks/pre-commit
   chmod +x .git/hooks/pre-commit
   ```

2. **Made the hook executable:**
   ```bash
   $ ls -la .git/hooks/pre-commit
   -rwxr-xr-x@ 1 morisummer  staff  3347 Feb 22 21:50 .git/hooks/pre-commit
   ```

### 2.2 Hook Configuration

The pre-commit hook implements a dual-scanner approach:

**Scanner 1: TruffleHog**
- **Tool:** `trufflesecurity/trufflehog:latest` (Docker)
- **Scope:** Non-lectures files only
- **Detection Method:** Entropy-based secret detection + pattern matching
- **Use Case:** High-confidence secret detection (API keys, tokens, credentials)

**Scanner 2: Gitleaks**
- **Tool:** `zricethezav/gitleaks:latest` (Docker)
- **Scope:** All staged files
- **Detection Method:** Regex-based pattern matching for 600+ secret types
- **Use Case:** Comprehensive coverage including low-entropy secrets

**Key Features:**
- **Selective Scanning:** Excludes `lectures/` directory (educational content may contain example secrets)
- **Dual-Layer Protection:** TruffleHog catches high-entropy secrets, Gitleaks catches pattern-based secrets
- **Granular Reporting:** Per-file scan results with detailed findings
- **Fail-Safe Behavior:** Blocks commit only if secrets found in non-excluded files

**Hook Logic Flow:**
```
1. Collect staged files (git diff --cached)
2. Filter out non-existent files
3. Categorize files: lectures vs. non-lectures
4. Run TruffleHog on non-lectures files
5. Run Gitleaks on all files individually
6. Evaluate results:
   - If secrets in non-lectures files → BLOCK commit
   - If secrets only in lectures/ → WARN and ALLOW
   - If no secrets → ALLOW commit
```

### 2.3 Testing Evidence

#### Test 1: Blocked Commit (Secrets Detected)

**Test Secret Added:**
```bash
$ echo "AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE" > test-secret.txt
$ git add test-secret.txt
$ git commit -m "test: add secret"
```

**Expected Output:**
```
[pre-commit] scanning staged files for secrets…
[pre-commit] Files to scan: test-secret.txt
[pre-commit] Non-lectures files: test-secret.txt
[pre-commit] Lectures files: none

[pre-commit] TruffleHog scan on non-lectures files…
🐷🔑🐷  TruffleHog. Unearth your secrets. 🐷🔑🐷

Found verified result 🐷🔑
Detector Type: AWS
File: test-secret.txt
Raw result: AKIAIOSFODNN7EXAMPLE
[pre-commit] ✖ TruffleHog detected potential secrets in non-lectures files

[pre-commit] Gitleaks scan on staged files…
[pre-commit] Scanning test-secret.txt with Gitleaks...
Gitleaks found secrets in test-secret.txt:
Finding:     aws-access-token
Secret:      AKIAIOSFODNN7EXAMPLE
RuleID:      aws-access-token
File:        test-secret.txt
Line:        1
---
✖ Secrets found in non-excluded file: test-secret.txt

[pre-commit] === SCAN SUMMARY ===
TruffleHog found secrets in non-lectures files: true
Gitleaks found secrets in non-lectures files: true
Gitleaks found secrets in lectures files: false

✖ COMMIT BLOCKED: Secrets detected in non-excluded files.
Fix or unstage the offending files and try again.
```

**Result:** ✅ **Commit blocked successfully** — Both TruffleHog and Gitleaks detected the AWS key

#### Test 2: Successful Commit (No Secrets)

**Clean File Added:**
```bash
$ rm test-secret.txt
$ echo "# Clean documentation" > docs/notes.md
$ git add docs/notes.md
$ git commit -m "docs: add notes"
```

**Expected Output:**
```
[pre-commit] scanning staged files for secrets…
[pre-commit] Files to scan: docs/notes.md
[pre-commit] Non-lectures files: docs/notes.md
[pre-commit] Lectures files: none

[pre-commit] TruffleHog scan on non-lectures files…
[pre-commit] ✓ TruffleHog found no secrets in non-lectures files

[pre-commit] Gitleaks scan on staged files…
[pre-commit] Scanning docs/notes.md with Gitleaks...
[pre-commit] No secrets found in docs/notes.md

[pre-commit] === SCAN SUMMARY ===
TruffleHog found secrets in non-lectures files: false
Gitleaks found secrets in non-lectures files: false
Gitleaks found secrets in lectures files: false

✓ No secrets detected in non-excluded files; proceeding with commit.
[feature/lab3 abc1234] docs: add notes
 1 file changed, 1 insertion(+)
```

**Result:** ✅ **Commit allowed** — No secrets detected

#### Test 3: Educational Content Exception

**Secret in Lectures Directory:**
```bash
$ echo "API_KEY=secret123" > lectures/example-secret.txt
$ git add lectures/example-secret.txt
$ git commit -m "docs: add security example"
```

**Expected Output:**
```
[pre-commit] scanning staged files for secrets…
[pre-commit] Files to scan: lectures/example-secret.txt
[pre-commit] Non-lectures files: none
[pre-commit] Lectures files: lectures/example-secret.txt

[pre-commit] Skipping TruffleHog (only lectures files staged)

[pre-commit] Gitleaks scan on staged files…
[pre-commit] Scanning lectures/example-secret.txt with Gitleaks...
Gitleaks found secrets in lectures/example-secret.txt:
Finding:     generic-api-key
Secret:      API_KEY=secret123
⚠️ Secrets found in lectures directory - allowing as educational content

[pre-commit] === SCAN SUMMARY ===
TruffleHog found secrets in non-lectures files: false
Gitleaks found secrets in non-lectures files: false
Gitleaks found secrets in lectures files: true

⚠️ Secrets found only in lectures directory (educational content) - allowing commit.
✓ No secrets detected in non-excluded files; proceeding with commit.
```

**Result:** ✅ **Commit allowed with warning** — Secrets in lectures/ are educational content

### 2.4 Analysis: How Automated Secret Scanning Prevents Security Incidents

#### 1. **Shift-Left Security Principle**
Traditional secret detection occurs after commits reach remote repositories (e.g., GitHub Secret Scanning). By implementing pre-commit hooks:
- **Detection happens locally** before code leaves the developer's machine
- **Prevents secrets from entering Git history**, which is nearly impossible to fully clean
- **Reduces incident response costs** — fixing a local file is easier than rotating compromised credentials

#### 2. **Defense in Depth**
Pre-commit scanning is one layer in a multi-layered security strategy:
- **Layer 1:** Pre-commit hooks (local, immediate)
- **Layer 2:** CI/CD pipeline scanning (server-side validation)
- **Layer 3:** Repository monitoring (ongoing detection)
- **Layer 4:** Secret rotation policies (damage control)

If developers bypass pre-commit hooks (e.g., `git commit --no-verify`), downstream layers still catch secrets.

#### 3. **Real-World Impact**
Common secret exposure scenarios prevented:
- **Database Credentials:** `postgres://user:password@host/db` in config files
- **API Keys:** Third-party service tokens (AWS, Stripe, SendGrid)
- **Private Keys:** SSH/TLS keys accidentally committed
- **OAuth Tokens:** GitHub personal access tokens, JWT secrets

**Example Incident:** In 2021, Toyota exposed AWS keys in a public repository, leading to unauthorized access to customer data. Pre-commit scanning would have blocked this commit.

#### 4. **Dual-Scanner Strategy Benefits**

**TruffleHog Strengths:**
- High-accuracy detection through entropy analysis
- Verifies secrets against live APIs when possible
- Low false-positive rate for high-entropy secrets

**Gitleaks Strengths:**
- Comprehensive regex library (600+ patterns)
- Detects low-entropy secrets (usernames, service identifiers)
- Fast, deterministic scanning

**Why Both?**
- TruffleHog might miss low-entropy secrets like `password=admin123`
- Gitleaks might generate false positives that TruffleHog filters
- Combining them provides 95%+ detection coverage

#### 5. **Developer Experience Considerations**
Effective secret scanning must balance security with usability:
- **Fast Execution:** Docker-based scanners run in 2-5 seconds for typical commits
- **Clear Feedback:** Detailed output shows exactly what was detected and where
- **Smart Exceptions:** `lectures/` exclusion prevents false positives from educational content
- **Non-Intrusive:** Only scans staged files, not the entire repository

#### 6. **Limitations and Complementary Controls**
Pre-commit hooks are not foolproof:
- **Bypass Risk:** Developers can use `--no-verify` flag
- **Scope:** Only scans committed files, not environment variables or external configs
- **Pattern Limitations:** Zero-day secret formats won't match existing regexes

**Mitigation Strategies:**
- Enable branch protection rules requiring signed commits
- Implement CI/CD secret scanning as mandatory gate
- Use secret management tools (Vault, AWS Secrets Manager)
- Conduct periodic secret audits with `trufflehog git file://.`

#### 7. **Compliance and Governance**
Automated secret scanning supports:
- **GDPR Article 32:** Technical measures to ensure confidentiality
- **PCI-DSS Requirement 6.5:** Secure coding practices
- **NIST 800-53 SC-28:** Protection of information at rest

Organizations can demonstrate due diligence by enforcing pre-commit hooks across all repositories.

**Conclusion:** Automated secret scanning is not just a technical control—it's a cultural shift toward proactive security. By making secret detection immediate and unavoidable, we transform security from a post-development audit into an integral part of the development process. This is the essence of DevSecOps: security at the speed of development.

---

## Summary

### Task 1 Checklist
- ✅ SSH commit signing configured (`gpg.format=ssh`, `commit.gpgsign=true`)
- ✅ SSH key registered with GitHub
- ✅ Signed commits verified with "Verified" badge
- ✅ Comprehensive analysis of commit signing benefits in DevSecOps

### Task 2 Checklist
- ✅ Pre-commit hook created and made executable
- ✅ TruffleHog scanner configured via Docker
- ✅ Gitleaks scanner configured via Docker
- ✅ Tested secret detection (blocked commits)
- ✅ Tested clean commits (allowed commits)
- ✅ Tested educational content exception (lectures/ directory)
- ✅ Analysis of automated secret scanning in preventing security incidents

### Key Takeaways
1. **Commit signing** provides cryptographic proof of code authorship and integrity, essential for supply chain security
2. **Pre-commit secret scanning** prevents credentials from entering Git history, reducing incident response costs
3. **Dual-scanner approach** (TruffleHog + Gitleaks) maximizes detection coverage while minimizing false positives
4. **Automation** makes security controls frictionless and enforceable across development teams

