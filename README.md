# Ultimate Guide to SLSA
### Supply-chain Levels for Software Artifacts

> A technical reference for understanding, implementing, and verifying SLSA across packages, containers, and binary artifacts.

---

## Table of Contents

1. [What is SLSA?](#1-what-is-slsa)
2. [Why Supply Chain Security Matters](#2-why-supply-chain-security-matters)
3. [Core Concepts](#3-core-concepts)
4. [SLSA Levels](#4-slsa-levels)
   - [Level 0 — No Guarantees](#level-0--no-guarantees)
   - [Level 1 — Provenance Exists](#level-1--provenance-exists)
   - [Level 2 — Hosted Build Service](#level-2--hosted-build-service)
   - [Level 3 — Hardened Builds](#level-3--hardened-builds)
5. [Level Comparison Matrix](#5-level-comparison-matrix)
6. [Provenance In Depth](#6-provenance-in-depth)
7. [Adoption Guide](#7-adoption-guide)
8. [Verification](#8-verification)
9. [Ecosystem & Tools](#9-ecosystem--tools)
10. [Common Misconceptions](#10-common-misconceptions)
11. [References](#11-references)

---

## 1. What is SLSA?

**SLSA** *(pronounced "salsa")* stands for **Supply-chain Levels for Software Artifacts**. It is an open security framework — originally developed by Google and now stewarded by the [Open Source Security Foundation (OpenSSF)](https://openssf.org) — that defines a graduated set of requirements for hardening the integrity of software build and release pipelines.

SLSA does not describe the security of the code itself. It describes the **verifiable trustworthiness of the process** by which that code is transformed into a distributable artifact — be it an npm package, a Python wheel, a Go binary, a container image, or any other software deliverable.

The framework answers a fundamental question a software consumer cannot otherwise answer:

> *"How do I know this artifact is actually what the author intended to release, built from the source code I can inspect, without tampering along the way?"*

SLSA answers that question through **provenance** — a signed, machine-readable attestation that records the exact inputs and environment of a build.

---

## 2. Why Supply Chain Security Matters

Modern software is rarely built in isolation. Applications depend on hundreds or thousands of third-party packages. Each dependency is a potential attack surface — not just for vulnerabilities in its code, but for **integrity attacks** targeting the build and distribution pipeline.

### Notable Supply Chain Attacks

| Year | Incident | Attack Vector | Impact |
|------|----------|---------------|--------|
| 2020 | **SolarWinds SUNBURST** | Malicious code injected into the build process | ~18,000 organisations compromised |
| 2021 | **Codecov bash uploader** | Artifact replaced in storage after build | Secrets leaked from thousands of CI pipelines |
| 2021 | **ua-parser-js** | Maintainer account hijacked; malicious version published | Cryptominer and credential stealer distributed via npm |
| 2022 | **PyTorch nightlies** | Dependency confusion attack via PyPI | Malicious `torchtriton` served to researchers |
| 2024 | **XZ Utils (CVE-2024-3094)** | Multi-year backdoor merged into source; shipped in distro packages | Targeted SSH daemon in affected Linux distributions |

In each case, the attack succeeded not because the source code review was inadequate, but because **the integrity of the build or distribution pipeline was not verifiable**. A consumer could not distinguish the legitimate artifact from the tampered one.

SLSA directly addresses this gap.

---

## 3. Core Concepts

### Artifact

Any distributable unit of software output: an npm package tarball, a Python wheel (`.whl`), a compiled Go binary, a container image, a Maven JAR, a Debian package, or a Helm chart. SLSA applies to any artifact a consumer might retrieve and trust.

### Provenance

A **provenance attestation** is a signed, structured document that records:

- **What** was built — the artifact's cryptographic digest (SHA-256)
- **Where** it was built — the build platform and its configuration
- **From what** — the exact source repository, branch, and commit
- **How** — the build steps, toolchain, and environment
- **When** — start and end timestamps with a unique invocation identifier

Provenance is the evidentiary basis of SLSA. Without it, trust is based on convention; with it, trust is based on verifiable evidence.

### Builder

The system responsible for executing the build and generating provenance. In SLSA terminology, a **trusted builder** is a build platform that generates provenance autonomously — meaning the developer cannot forge or alter the provenance document, because they did not produce it.

Examples: GitHub Actions (with OIDC), Google Cloud Build, Tekton with Chains, GitLab CI with runner attestation.

### Attestation

A cryptographically signed envelope containing a provenance statement. SLSA uses the [in-toto Attestation Framework](https://in-toto.io) and [DSSE (Dead Simple Signing Envelope)](https://github.com/secure-systems-lab/dsse) as the envelope format.

### Verification

The act of checking, at consumption time, that an artifact's provenance attestation:

1. Is cryptographically valid (signature checks out)
2. Was produced by a trusted builder
3. Links to the expected source repository and commit
4. Matches the artifact's digest exactly

---

## 4. SLSA Levels

SLSA defines four levels of assurance, numbered **0 through 3**. Each level inherits the requirements of all prior levels and adds new constraints. The levels progress along two axes: **provenance quality** and **build environment hardening**.

---

### Level 0 — No Guarantees

**Threat model:** No claims are made about how the artifact was produced.

Level 0 is not a SLSA level per se — it represents the baseline state of most software today: no formal build process requirements, no provenance, no verifiable chain of custody.

#### Characteristics

- Build may be manual, undocumented, or developer-machine-local
- No provenance document produced
- No cryptographic link between source and artifact
- Consumer must trust the publisher unconditionally

#### Example — npm package published from a developer laptop

A maintainer clones their repository, runs `npm run build`, and publishes directly:

```bash
# Developer's local machine
npm run build
npm publish
```

The published tarball on npmjs.com has no attestation. A consumer downloading `my-lib@1.2.0` cannot verify whether the tarball was built from the tagged commit or from a locally modified working directory, cannot verify the build environment was clean, and cannot detect whether the registry artifact was swapped after publication.

---

### Level 1 — Provenance Exists

**Threat model:** Prevents accidental or after-the-fact tampering by establishing a documented, auditable build process. Does **not** prevent a malicious or compromised developer from generating false provenance.

#### Requirements

| Requirement | Detail |
|-------------|--------|
| **Scripted build** | The build process must be fully automated; no manual steps that produce the final artifact |
| **Provenance available** | A provenance document must be generated and made available alongside the artifact |
| **Provenance content** | Must identify the builder, the build invocation, and the artifact digest |

#### What it does NOT require

- The provenance need not be signed
- The provenance may be generated by the developer's own tooling (not an independent build service)
- The build environment does not need to be isolated or ephemeral

#### Example — Python package with self-generated provenance

A CI workflow builds a Python wheel and emits a minimal provenance JSON:

```yaml
# .github/workflows/publish.yml
- name: Build wheel
  run: python -m build

- name: Generate basic provenance
  run: |
    echo '{
      "artifact": "dist/my_lib-1.2.0-py3-none-any.whl",
      "sha256": "'$(sha256sum dist/my_lib-1.2.0-py3-none-any.whl | cut -d' ' -f1)'",
      "source": "https://github.com/example/my-lib",
      "ref": "'$GITHUB_SHA'",
      "builder": "github-actions",
      "runId": "'$GITHUB_RUN_ID'"
    }' > provenance.json
```

The provenance is published with the release. A consumer can verify the artifact digest matches. However, because the developer's workflow generated the provenance, a compromised developer could produce false provenance claiming a different (clean) commit.

**Real-world adoption:** PyPI Trusted Publishers, npm `--provenance` flag (when using a supported CI provider) — these reach L1 or push toward L2.

---

### Level 2 — Hosted Build Service

**Threat model:** Prevents a compromised developer from forging provenance. The provenance is generated and signed by an independent, trusted build service — not the developer's code. An attacker would need to compromise the build service itself to produce false provenance.

#### Requirements

| Requirement | Detail |
|-------------|--------|
| **All Level 1 requirements** | Scripted build, provenance available |
| **Service-generated provenance** | The build service — not the developer's workflow code — generates the provenance |
| **Authenticated provenance** | The provenance must be signed by the build service using a verifiable key |
| **Source reference** | Provenance must include the source repository URI and commit identifier |
| **Unique invocation ID** | Must include a reference to the specific build run for audit trail purposes |

#### The key distinction from Level 1

At Level 1, the developer's code can write anything into the provenance document. At Level 2, the provenance is produced by a **privileged component of the build service** that runs outside the developer's workflow code and signs with a key the developer cannot access.

On GitHub Actions, this is achieved via **OIDC tokens** — the runner platform itself signs an attestation using an ephemeral key backed by GitHub's identity infrastructure, before the workflow's user-defined steps can modify the output.

#### Example — Container image with Sigstore-signed provenance (GitHub Actions)

```yaml
# .github/workflows/build-container.yml
name: Build and Push Container

on:
  push:
    tags: ["v*"]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write       # Required for OIDC-based signing

    steps:
      - uses: actions/checkout@v4

      - name: Build container image
        run: docker build -t ghcr.io/example/myapp:${{ github.ref_name }} .

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Push image
        run: docker push ghcr.io/example/myapp:${{ github.ref_name }}

      - name: Sign image and generate provenance (keyless via OIDC)
        run: |
          cosign sign --yes ghcr.io/example/myapp:${{ github.ref_name }}
          cosign attest --yes \
            --predicate provenance.json \
            --type slsaprovenance \
            ghcr.io/example/myapp:${{ github.ref_name }}
```

The `cosign sign` step uses GitHub's OIDC token to request an ephemeral Fulcio certificate, signs the image digest, and records the entry in the Rekor transparency log. The developer's workflow code cannot forge this signature because it does not hold the signing key — Sigstore's Fulcio CA issued it based on the OIDC identity of the GitHub Actions run.

**Real-world adoption:** npm provenance (L2 via GitHub Actions OIDC), PyPI Trusted Publishers (approaching L2), GitHub's native artifact attestations (`gh attestation`).

---

### Level 3 — Hardened Builds

**Threat model:** Prevents a compromised build service *worker node* or *build step* from tampering with the artifact or provenance undetected. Builds are hermetic (reproducible, network-isolated), ephemeral (no state between runs), and provenance is non-falsifiable even by an insider on the build platform.

#### Requirements

| Requirement | Detail |
|-------------|--------|
| **All Level 2 requirements** | Service-generated, authenticated, source-referenced provenance |
| **Hermetic build** | All build dependencies must be declared upfront and fetched before the build begins; no network access during the build itself |
| **Ephemeral environment** | Each build runs in a freshly provisioned environment; no filesystem, memory, or process state is shared between builds |
| **Isolated build** | The build cannot influence other builds on the same platform |
| **Non-falsifiable provenance** | The signing key used for provenance is inaccessible to the build steps themselves — the build cannot sign its own provenance |

#### What hermetic means in practice

A hermetic build resolves and snapshots all dependencies *before* the build begins. During the build, no outbound network calls are permitted. This eliminates a class of attacks where a dependency is replaced mid-build (e.g., a `pip install` pulling a malicious version at build time).

**Non-hermetic (vulnerable):**
```dockerfile
# Dockerfile — fetches dependencies at build time (network access required)
RUN pip install requests==2.31.0   # Version resolved at build time; if index is compromised, different bytes arrive
```

**Hermetic (L3-compatible):**
```
All dependency wheels pre-fetched and hash-verified in a lockfile.
Build runs with --no-index --find-links=./vendor/
Network interface disabled in the build sandbox.
```

#### Example — Go binary at SLSA L3 using slsa-github-generator

The [`slsa-github-generator`](https://github.com/slsa-framework/slsa-github-generator) project provides reusable GitHub Actions workflows that are pre-certified to produce SLSA L3 provenance. The key mechanism: the provenance is generated and signed in a **separate, isolated job** that the caller workflow cannot influence — achieving non-falsifiability.

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ["v*"]

jobs:
  build:
    permissions:
      id-token: write
      contents: read
      actions: read

    # This reusable workflow is the certified SLSA L3 builder.
    # It runs in an isolated context; caller code cannot tamper with provenance.
    uses: slsa-framework/slsa-github-generator/.github/workflows/builder_go_slsa3.yml@v2.0.0
    with:
      go-version: "1.22"
      config-file: .slsa-goreleaser.yml
```

The reusable workflow:
1. Compiles the binary in an ephemeral, network-controlled environment
2. Generates the provenance document from inputs the caller cannot modify
3. Signs the provenance using an OIDC-backed ephemeral key
4. Uploads the signed provenance to Rekor
5. Attaches both artifact and provenance to the GitHub release

The calling repository's code is entirely excluded from steps 2–5.

---

## 5. Level Comparison Matrix

| Requirement | L0 | L1 | L2 | L3 |
|-------------|:--:|:--:|:--:|:--:|
| Scripted / automated build | ✗ | ✓ | ✓ | ✓ |
| Provenance document available | ✗ | ✓ | ✓ | ✓ |
| Provenance includes artifact digest | ✗ | ✓ | ✓ | ✓ |
| Provenance includes source ref (commit) | ✗ | ✗ | ✓ | ✓ |
| Provenance generated by build service | ✗ | ✗ | ✓ | ✓ |
| Provenance cryptographically signed | ✗ | ✗ | ✓ | ✓ |
| Signed by key inaccessible to build | ✗ | ✗ | ✗ | ✓ |
| Hermetic build (no network mid-build) | ✗ | ✗ | ✗ | ✓ |
| Ephemeral build environment | ✗ | ✗ | ✗ | ✓ |
| Isolated build (no cross-build state) | ✗ | ✗ | ✗ | ✓ |
| Non-falsifiable provenance | ✗ | ✗ | ✗ | ✓ |

---

## 6. Provenance In Depth

### The in-toto Attestation Format

SLSA provenance is expressed as an **in-toto attestation** wrapped in a **DSSE envelope**. The structure has three layers:

```
DSSE Envelope (signed)
└── Statement (in-toto v0.1)
    ├── _type
    ├── subject[]         ← artifact(s) and their digests
    └── predicate         ← the SLSA provenance payload
        ├── buildDefinition
        │   ├── buildType
        │   ├── externalParameters   ← what the caller controlled
        │   └── systemParameters     ← what the build service controlled
        └── runDetails
            ├── builder.id
            └── metadata (timestamps, invocation ID)
```

### Annotated SLSA v1.0 Provenance Document

```json
{
  "_type": "https://in-toto.io/Statement/v0.1",
  "predicateType": "https://slsa.dev/provenance/v1",

  "subject": [
    {
      "name": "myapp-v1.2.0-linux-amd64.tar.gz",
      "digest": {
        "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
      }
    }
  ],

  "predicate": {
    "buildDefinition": {
      "buildType": "https://slsa.dev/container-based-build/v0.1",

      "externalParameters": {
        "repository": "https://github.com/example/myapp",
        "ref": "refs/tags/v1.2.0"
      },

      "systemParameters": {
        "GITHUB_ACTOR_ID": "12345678",
        "GITHUB_EVENT_NAME": "push",
        "GITHUB_REF": "refs/tags/v1.2.0",
        "GITHUB_REPOSITORY": "example/myapp",
        "GITHUB_REPOSITORY_ID": "987654321",
        "GITHUB_RUN_ATTEMPT": "1",
        "GITHUB_SHA": "aabbccdd1234567890abcdef1234567890abcdef",
        "GITHUB_WORKFLOW_REF": "example/myapp/.github/workflows/release.yml@refs/tags/v1.2.0"
      }
    },

    "runDetails": {
      "builder": {
        "id": "https://github.com/slsa-framework/slsa-github-generator/.github/workflows/builder_go_slsa3.yml@refs/tags/v2.0.0"
      },
      "metadata": {
        "invocationId": "https://github.com/example/myapp/actions/runs/9876543210/attempts/1",
        "startedOn": "2024-06-01T10:00:00Z",
        "finishedOn": "2024-06-01T10:04:37Z"
      }
    }
  }
}
```

#### Field-by-field explanation

| Field | Purpose | SLSA Level Required |
|-------|---------|-------------------|
| `subject[].digest.sha256` | Cryptographically binds the attestation to a specific artifact byte-for-byte | L1 |
| `builder.id` | Identifies the build platform; consumers use this to determine which signing key to trust | L1 |
| `externalParameters.repository` | The source repository the build claims to originate from | L2 |
| `externalParameters.ref` | The specific tag or commit ref | L2 |
| `systemParameters` | Values set by the build platform itself, not the developer — these cannot be forged by the caller | L2 |
| `invocationId` | Unique URL to the specific build run; enables cross-referencing with audit logs | L2 |
| `GITHUB_WORKFLOW_REF` | Identifies the exact builder workflow version used — critical for verifying builder provenance at L3 | L3 |

---

## 7. Adoption Guide

### Recommended Progression

SLSA adoption is intentionally incremental. Reaching Level 1 delivers immediate value. Each subsequent level adds meaningful resistance to a concrete threat class.

```
Current state        Target           Effort    Primary gain
─────────────────────────────────────────────────────────────
Manual / ad-hoc  →   Level 1        Low       Auditability
CI build only    →   Level 1        Low       Provenance trail
Level 1          →   Level 2        Medium    Tamper-evident provenance
Level 2          →   Level 3        High      Non-falsifiable provenance
```

---

### Reaching Level 1

**For any ecosystem:** Migrate builds to CI. Emit a provenance document and attach it to the release.

**npm packages** — use the built-in `--provenance` flag (requires GitHub Actions or other OIDC-enabled CI):

```bash
npm publish --provenance
```

This generates and uploads an in-toto attestation to npmjs.com automatically. Consumers can inspect it via `npm audit signatures`.

**Python packages (PyPI)** — configure a Trusted Publisher:

1. On PyPI, go to your project → *Publishing* → *Add a new publisher*
2. Select GitHub Actions, enter your repo and workflow file name
3. In your workflow, publish without a password — PyPI issues a short-lived token via OIDC:

```yaml
- name: Publish to PyPI
  uses: pypa/gh-action-pypi-publish@release/v1
  # No api_token needed — OIDC trust established via Trusted Publisher config
```

---

### Reaching Level 2

**Container images** — use Cosign with keyless (OIDC) signing:

```yaml
- name: Install Cosign
  uses: sigstore/cosign-installer@v3

- name: Build and push image
  run: |
    docker build -t ghcr.io/${{ github.repository }}:${{ github.ref_name }} .
    docker push ghcr.io/${{ github.repository }}:${{ github.ref_name }}

- name: Sign image (keyless)
  run: |
    cosign sign --yes \
      ghcr.io/${{ github.repository }}:${{ github.ref_name }}
```

The signature is transparently logged in Rekor. No long-lived key material is stored in the repository.

**GitHub native attestations** (GA as of 2024) — a simpler L2-compatible alternative:

```yaml
- name: Attest build provenance
  uses: actions/attest-build-provenance@v1
  with:
    subject-path: dist/myapp-linux-amd64
```

Attestations are stored in the GitHub Attestations API and verifiable with `gh attestation verify`.

---

### Reaching Level 3

Use a **pre-certified SLSA L3 builder**. These are reusable workflows that have been independently verified to satisfy Level 3 requirements.

**Go binaries:**
```yaml
uses: slsa-framework/slsa-github-generator/.github/workflows/builder_go_slsa3.yml@v2.0.0
```

**Generic artifacts / containers:**
```yaml
uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2.0.0
```

Other certified L3 builders: **Google Cloud Build** (with SLSA integration), **Tekton with Tekton Chains** (on GKE/Cloud Run), **GitLab Ultimate** (with runner attestation and isolation policies).

---

## 8. Verification

Provenance is only valuable if consumers verify it. Unverified provenance provides auditability but not security guarantees.

### slsa-verifier

The reference CLI for verifying SLSA provenance attestations:

```bash
# Install
go install github.com/slsa-framework/slsa-verifier/v2/cli/slsa-verifier@latest

# Verify a Go binary release
slsa-verifier verify-artifact myapp-v1.2.0-linux-amd64 \
  --provenance-path myapp-v1.2.0-linux-amd64.intoto.jsonl \
  --source-uri github.com/example/myapp \
  --source-tag v1.2.0

# Verify a container image
slsa-verifier verify-image ghcr.io/example/myapp:v1.2.0 \
  --source-uri github.com/example/myapp
```

On success, `slsa-verifier` prints the verified source URI and builder identity. On failure, it exits non-zero with a descriptive error.

### Cosign — container image verification

```bash
# Verify signature and retrieve attestation
cosign verify \
  --certificate-identity-regexp "^https://github.com/example/myapp/" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/example/myapp:v1.2.0

# Verify SLSA provenance attestation specifically
cosign verify-attestation \
  --type slsaprovenance \
  --certificate-identity-regexp "^https://github.com/example/myapp/" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/example/myapp:v1.2.0 | jq .payload | base64 -d | jq .
```

### GitHub CLI — native attestation verification

```bash
# Verify an artifact against GitHub's attestation store
gh attestation verify myapp-v1.2.0-linux-amd64 \
  --repo example/myapp

# Verify a container image
gh attestation verify oci://ghcr.io/example/myapp:v1.2.0 \
  --repo example/myapp
```

### npm — package signature verification

```bash
# Audit signatures of all installed packages
npm audit signatures

# Output example for a SLSA-attested package:
# myapp@1.2.0: Verified attestation (SLSA provenance)
#   Source: https://github.com/example/myapp
#   Workflow: .github/workflows/publish.yml
```

### Enforcement in CI/CD pipelines

Verification should be integrated into the **consuming** pipeline, not treated as a manual step.

```yaml
# Example: Verify a downloaded artifact before using it
- name: Download release artifact
  run: |
    gh release download v1.2.0 --repo example/myapp \
      --pattern "myapp-linux-amd64*"

- name: Verify SLSA provenance before use
  run: |
    slsa-verifier verify-artifact myapp-linux-amd64 \
      --provenance-path myapp-linux-amd64.intoto.jsonl \
      --source-uri github.com/example/myapp \
      --source-tag v1.2.0
    echo "Provenance verified — proceeding"

- name: Use artifact
  run: ./myapp-linux-amd64 --version
```

---

## 9. Ecosystem & Tools

| Tool | Role | Supported Levels |
|------|------|-----------------|
| [slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator) | Certified reusable builder workflows for GitHub Actions | L1, L2, L3 |
| [slsa-verifier](https://github.com/slsa-framework/slsa-verifier) | Reference CLI for provenance verification | All |
| [Sigstore / Cosign](https://sigstore.dev) | Keyless artifact signing and verification infrastructure | L2, L3 |
| [Rekor](https://rekor.sigstore.dev) | Immutable public transparency log for signed attestations | L2, L3 |
| [Fulcio](https://github.com/sigstore/fulcio) | OIDC-backed certificate authority for ephemeral signing keys | L2, L3 |
| [Tekton Chains](https://tekton.dev/docs/chains/) | Kubernetes-native provenance generation for Tekton pipelines | L1, L2, L3 |
| [in-toto](https://in-toto.io) | Attestation framework and predicate format underlying SLSA | All |
| [witness](https://github.com/in-toto/witness) | Pluggable attestation collection for any CI environment | L1, L2 |
| [Kyverno / OPA Gatekeeper](https://kyverno.io) | Kubernetes admission controllers that enforce provenance at deploy time | Verification |
| npm `--provenance` | Built-in provenance generation for npm publish | L1/L2 |
| PyPI Trusted Publishers | OIDC-based provenance for Python packages | L1/L2 |
| `gh attestation` | GitHub CLI for native attestation generation and verification | L2 |

---

## 10. Common Misconceptions

### "SLSA guarantees my software has no vulnerabilities."

SLSA makes **no claims about the content of the code**. A SLSA L3 artifact may contain critical CVEs. SLSA ensures only that the artifact was built from the stated source, by the stated builder, without tampering in transit. Vulnerability scanning (Grype, Trivy, Dependabot, osv-scanner) is complementary and independent.

### "Only large companies can achieve Level 3."

Level 3 is accessible to any open-source project on GitHub today, using the `slsa-github-generator` reusable workflows at no cost. The engineering effort is a one-time workflow addition, not ongoing infrastructure work.

### "Level 1 is barely worth doing."

Level 1 establishes a provenance trail — any post-release tampering of the artifact becomes detectable by comparing the attested digest to the downloaded artifact. This stops opportunistic registry-swap attacks, which have been the mechanism in several real incidents.

### "Provenance means the supply chain is fully secured."

SLSA addresses the **build and distribution** integrity layer. It does not cover: vulnerabilities in source code, compromised developer machines before code is committed, compromised source repository access controls, or runtime security. A complete supply chain security posture requires SLSA alongside SBOM generation, dependency review, VEX statements, and access control policies.

### "I only need to verify provenance once at release time."

Provenance should be verified at **consumption time** — when your CI pipeline downloads a dependency or base image. A package can be resigned with a different key or have provenance retroactively added without changing the artifact. Verifying at the point of use, against a pinned expected builder identity, is the correct model.

---

## 11. References

- [SLSA Specification v1.0](https://slsa.dev/spec/v1.0) — Official framework specification
- [slsa.dev](https://slsa.dev) — SLSA home page, getting started guides
- [OpenSSF SLSA GitHub](https://github.com/slsa-framework) — Source for generator, verifier, and specification
- [in-toto Attestation Framework](https://github.com/in-toto/attestation) — Predicate format specification
- [DSSE Specification](https://github.com/secure-systems-lab/dsse) — Signing envelope format
- [Sigstore Documentation](https://docs.sigstore.dev) — Cosign, Rekor, Fulcio reference
- [SLSA GitHub Generator](https://github.com/slsa-framework/slsa-github-generator) — Reusable L3 builder workflows
- [SLSA Verifier](https://github.com/slsa-framework/slsa-verifier) — Reference verification CLI
- [Tekton Chains](https://tekton.dev/docs/chains/) — Kubernetes-native SLSA provenance
- [npm Provenance](https://docs.npmjs.com/generating-provenance-statements) — npm registry provenance documentation
- [PyPI Trusted Publishers](https://docs.pypi.org/trusted-publishers/) — PyPI provenance documentation
- [GitHub Artifact Attestations](https://docs.github.com/en/actions/security-guides/using-artifact-attestations-to-establish-provenance-for-builds) — GitHub native attestation documentation

---

*Specification reference: SLSA v1.0 · Last reviewed: 2024 · Maintained at [github.com/cjohannsen81/slsa-guide](https://github.com/cjohannsen81/slsa-guide)*
