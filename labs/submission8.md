# Lab 8 — Software Supply Chain Security: Signing, Verification, and Attestations

## Task 1 — Local Registry, Signing & Verification (4 pts)

### 1.1 Local Registry Setup

Started a local OCI registry (registry:3) on port 5050 (port 5000 was occupied by macOS AirPlay), pulled `bkimminich/juice-shop:v19.0.0`, tagged and pushed it:

```
docker run -d --restart=always -p 5050:5000 --name registry registry:3
docker tag bkimminich/juice-shop:v19.0.0 localhost:5050/juice-shop:v19.0.0
docker push localhost:5050/juice-shop:v19.0.0
```

Digest reference resolved:

```
Using digest ref: localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7
```

### 1.2 Key Pair Generation

Generated a Cosign key pair in `labs/lab8/signing/`:

```
cosign generate-key-pair
# Created: cosign.key (private), cosign.pub (public)
```

### 1.3 Signing and Verification

> **Note:** Cosign v3.0.5 deprecated `--tlog-upload=false`. A signing config without transparency log was used instead (`cosign signing-config create`).

**Signing:**

```
$ cosign sign --yes --allow-insecure-registry \
    --signing-config labs/lab8/signing/signing-config.json \
    --key labs/lab8/signing/cosign.key \
    "localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"

Signing artifact...
```

**Verification:**

```
$ cosign verify --allow-insecure-registry --insecure-ignore-tlog \
    --key labs/lab8/signing/cosign.pub \
    "localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"

Verification for localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key

[{"critical":{"identity":{"docker-reference":"localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"},
"image":{"docker-manifest-digest":"sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"},
"type":"https://sigstore.dev/cosign/sign/v1"},"optional":{}}]
```

### 1.4 Tamper Demonstration

Pushed `busybox:latest` over the same `v19.0.0` tag to simulate a supply chain attack:

```
docker tag busybox:latest localhost:5050/juice-shop:v19.0.0
docker push localhost:5050/juice-shop:v19.0.0
```

The tampered image received a different digest:

```
After tamper digest ref: localhost:5050/juice-shop@sha256:50a3a2fef78c92dee45a3a9b72af5bdcbff6476e685cef49d97f286b6ce6f14a
```

**Verification of tampered image — FAILED (expected):**

```
$ cosign verify --allow-insecure-registry --insecure-ignore-tlog \
    --key labs/lab8/signing/cosign.pub \
    "localhost:5050/juice-shop@sha256:50a3a2fef78c92dee45a3a9b72af5bdcbff6476e685cef49d97f286b6ce6f14a"

Error: no signatures found
```

**Verification of original image — PASSED:**

```
$ cosign verify --allow-insecure-registry --insecure-ignore-tlog \
    --key labs/lab8/signing/cosign.pub \
    "localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"

Verification for localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
```

### Analysis: How Signing Protects Against Tag Tampering

Container image tags (like `v19.0.0`) are mutable pointers — anyone with push access can overwrite a tag to point to a completely different image. This is exactly what happened in the tamper demo: the `v19.0.0` tag was redirected from the real Juice Shop image to `busybox`.

Cosign signatures are bound to the **subject digest** (e.g., `sha256:872efcc...`), not the tag. The "subject digest" is the SHA-256 hash of the image manifest — a cryptographic fingerprint that uniquely identifies the exact image content (layers, config, metadata). Even a single byte change produces a completely different digest.

When we signed the original image, Cosign created a signature over digest `sha256:872efcc...`. After tampering, the tag pointed to digest `sha256:50a3a2f...` (busybox). Verification failed because no signature existed for that digest. Meanwhile, the original digest's signature remained valid. This demonstrates why **digest-based references** are a supply chain security best practice: they are immutable and tied to the actual content, making tag substitution attacks detectable.

---

## Task 2 — Attestations: SBOM & Provenance (4 pts)

### 2.1 SBOM Attestation (CycloneDX)

Converted the Lab 4 Syft SBOM to CycloneDX format:

```
docker run --rm \
  -v "$(pwd)/labs/lab4/syft":/in:ro \
  -v "$(pwd)/labs/lab8/attest":/out \
  anchore/syft:latest \
  convert /in/juice-shop-syft-native.json -o cyclonedx-json=/out/juice-shop.cdx.json
```

Attached the CycloneDX SBOM as an attestation:

```
$ cosign attest --yes --allow-insecure-registry \
    --signing-config labs/lab8/signing/signing-config.json \
    --key labs/lab8/signing/cosign.key \
    --predicate labs/lab8/attest/juice-shop.cdx.json \
    --type cyclonedx \
    "localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"

Using payload from: labs/lab8/attest/juice-shop.cdx.json
Signing artifact...
```

**Verification:**

```
$ cosign verify-attestation --allow-insecure-registry --insecure-ignore-tlog \
    --key labs/lab8/signing/cosign.pub --type cyclonedx \
    "localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"

Verification for localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
```

**Payload inspection (SBOM attestation structure):**

```json
{
  "_type": "https://in-toto.io/Statement/v0.1",
  "predicateType": "https://cyclonedx.org/bom",
  "subject": [
    {
      "name": "localhost:5050/juice-shop",
      "digest": {
        "sha256": "872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"
      }
    }
  ],
  "predicate_keys": [
    "$schema",
    "bomFormat",
    "components",
    "dependencies",
    "metadata",
    "serialNumber",
    "specVersion",
    "version"
  ]
}
```

### 2.2 Provenance Attestation

Created a minimal SLSA Provenance v1 predicate and attached it:

```
$ cosign attest --yes --allow-insecure-registry \
    --signing-config labs/lab8/signing/signing-config.json \
    --key labs/lab8/signing/cosign.key \
    --predicate labs/lab8/attest/provenance.json \
    --type slsaprovenance \
    "localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"

Using payload from: labs/lab8/attest/provenance.json
Signing artifact...
```

**Verification and payload inspection:**

```json
{
  "_type": "https://in-toto.io/Statement/v0.1",
  "predicateType": "https://slsa.dev/provenance/v0.2",
  "subject": [
    {
      "name": "localhost:5050/juice-shop",
      "digest": {
        "sha256": "872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"
      }
    }
  ],
  "predicate": {
    "builder": { "id": "student@local" },
    "buildType": "manual-local-demo",
    "invocation": {
      "parameters": {
        "image": "localhost:5050/juice-shop@sha256:872efcc03cc16e8c4e2377202117a218be83aa1d05eb22297b248a325b400bd7"
      }
    },
    "metadata": {
      "buildStartedOn": "2026-03-29T15:28:18Z",
      "completeness": {
        "parameters": true,
        "environment": false,
        "materials": false
      },
      "reproducible": false
    }
  }
}
```

### Analysis: Attestations vs Signatures

| Aspect | Signatures | Attestations |
|--------|-----------|-------------|
| **What they prove** | The image has not been tampered with since signing | Structured claims *about* the image (contents, build process, etc.) |
| **Content** | Cryptographic signature over the image digest | In-toto statement wrapping a typed predicate + signature |
| **Use case** | Integrity and authenticity verification | Policy enforcement, compliance, audit trail |
| **Granularity** | Binary — signed or not | Rich metadata — SBOM, provenance, vulnerability scans |

**SBOM attestation** contains a complete CycloneDX bill of materials (components, dependencies, licenses, metadata) cryptographically bound to the image digest. This allows consumers to verify exactly what software components are inside the image and enforce policies (e.g., block images with known vulnerable dependencies).

**Provenance attestations** provide supply chain traceability by recording *how* an image was built: the builder identity, build type, parameters, timestamps, and completeness information. In production with SLSA, this enables verifying that an image was built by an authorized CI/CD system from the expected source code, preventing unauthorized builds from entering the deployment pipeline.

---

## Task 3 — Artifact (Blob/Tarball) Signing (2 pts)

### Signing and Verification

Created a sample tarball and signed it:

```
echo "sample content $(date -u)" > labs/lab8/artifacts/sample.txt
tar -czf labs/lab8/artifacts/sample.tar.gz -C labs/lab8/artifacts sample.txt

$ cosign sign-blob --yes \
    --signing-config labs/lab8/signing/signing-config.json \
    --key labs/lab8/signing/cosign.key \
    --bundle labs/lab8/artifacts/sample.tar.gz.bundle \
    labs/lab8/artifacts/sample.tar.gz

Using payload from: labs/lab8/artifacts/sample.tar.gz
Signing artifact...
Wrote bundle to file labs/lab8/artifacts/sample.tar.gz.bundle
```

**Verification:**

```
$ cosign verify-blob --key labs/lab8/signing/cosign.pub \
    --bundle labs/lab8/artifacts/sample.tar.gz.bundle \
    --insecure-ignore-tlog \
    labs/lab8/artifacts/sample.tar.gz

Verified OK
```

### Analysis: Blob Signing vs Container Image Signing

**Use cases for signing non-container artifacts:**
- **Release binaries** — Ensure downloaded executables haven't been tampered with (e.g., CLI tools, firmware updates)
- **Configuration files** — Verify that deployment configs (Terraform plans, Ansible playbooks, Helm charts) come from authorized sources
- **Compliance documents** — Sign audit reports, SBOMs, or policy files to prove they were produced by an authorized process
- **Patches and updates** — Validate that software patches are authentic before applying them

**How blob signing differs from container image signing:**

| Aspect | Container Image Signing | Blob Signing |
|--------|------------------------|-------------|
| **Storage** | Signature stored in OCI registry alongside the image | Signature stored as a bundle file (JSON) alongside the artifact |
| **Reference** | Bound to image manifest digest via registry API | Bound to file content hash |
| **Distribution** | Registry-native — signature travels with the image | File-based — bundle must be distributed alongside the artifact |
| **Verification** | `cosign verify` queries the registry for signatures | `cosign verify-blob` reads the local bundle file |
| **Scope** | OCI images and artifacts only | Any file — binaries, tarballs, configs, documents |

Blob signing extends the same cryptographic guarantees (integrity and authenticity) to any file, making it a versatile tool for securing the entire software supply chain beyond just container images.
