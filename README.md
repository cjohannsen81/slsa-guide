# Ultimate Guide to SLSA
### Supply-chain Levels for Software Artifacts

A technical reference for understanding, implementing, and verifying SLSA across packages, containers, and binary artifacts.

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
11. [AI and SLSA](#11-ai-and-slsa)
    - [Why AI Introduces New Supply Chain Risks](#why-ai-introduces-new-supply-chain-risks)
    - [AI Artifacts vs. Traditional Software Artifacts](#ai-artifacts-vs-traditional-software-artifacts)
    - [SLSA Applied to AI: Level by Level](#slsa-applied-to-ai-level-by-level)
    - [Emerging Standards and Tooling](#emerging-standards-and-tooling)
    - [AI as a Builder: Risks of AI-Generated Code](#ai-as-a-builder-risks-of-ai-generated-code)
12. [References](#12-references)

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

## 11. AI and SLSA

Artificial intelligence introduces a new class of artifact into the software supply chain — one that is larger, more opaque, more expensive to produce, and harder to reproduce than any traditional software binary. The SLSA framework was designed for deterministic builds of source code; applying it to AI requires both adaptation of existing concepts and introduction of entirely new provenance dimensions.

This section examines AI's impact on SLSA from two directions: AI artifacts (models and datasets) as objects that need supply chain protection, and AI as a builder (code generation tools, autonomous agents) that introduces new risk into the build process itself.

---

### Why AI Introduces New Supply Chain Risks

Traditional software supply chain attacks target the build pipeline — the transformation of source code into a binary. The source code is human-authored, version-controlled, and relatively compact. An attacker who wants to tamper with the artifact must either modify the source, compromise the build, or tamper with the distributed artifact.

AI models introduce compounding risk at every layer:

| Layer | Traditional Software | AI Model |
|-------|---------------------|----------|
| **Source** | Source code in a VCS (Git) | Training code + training data (often terabytes, distributed, dynamic) |
| **Build** | Compiler + linker (deterministic, minutes) | Training run (GPU cluster, days to weeks, non-deterministic) |
| **Artifact** | Binary / package (MBs) | Model weights (GBs to hundreds of GBs) |
| **Dependencies** | npm/pip packages, pinned | Pretrained base models, datasets, fine-tuning adapters |
| **Reproducibility** | Bit-for-bit reproducible (ideally) | Rarely reproducible without identical hardware, seeds, and data order |
| **Tampering surface** | Build steps, registry | Weights files, quantization step, adapter merging, GGUF conversion |

A further dimension is introduced by the **non-deterministic nature of training**: even if two training runs use identical code, data, and hyperparameters, floating-point non-determinism across GPU kernels means the resulting weight tensors will differ. This fundamentally challenges the hermetic-build and reproducibility assumptions that underpin SLSA Level 3.

Additionally, AI supply chains surface a new category of attack not addressed by classic SLSA: **data poisoning** — the deliberate injection of malicious or biased examples into a training dataset to influence the model's learned behaviour at inference time, without any tampering of the artifact after training completes.

---

### AI Artifacts vs. Traditional Software Artifacts

Before mapping SLSA levels to AI, it is important to characterise what an AI artifact actually is and why provenance for it is structurally different.

**An AI artifact typically consists of:**

- **Model weights** — the primary artifact; large binary files (`.safetensors`, `.gguf`, `.pt`, `.bin`) encoding learned parameters
- **Configuration files** — architecture definition (`config.json`), tokenizer configuration, generation defaults
- **Tokenizer** — vocabulary and tokenization rules; altering this changes model behaviour without touching weights
- **Adapter layers** — LoRA, QLoRA, or other parameter-efficient fine-tuning (PEFT) adapters that modify a base model
- **Quantized variants** — post-training quantized versions (INT4, INT8) produced by a separate, auditable conversion step

Each of these components can be independently tampered with. A compromised tokenizer that maps certain tokens to unexpected IDs, or a malicious LoRA adapter merged with a legitimate base model, can produce harmful outputs without the base model weights ever being touched.

**Provenance for AI must therefore capture additional metadata not required for traditional software:**

| Provenance Field | Traditional Software | AI Model |
|-----------------|---------------------|----------|
| Source repository + commit | ✓ Required | ✓ Required (training code) |
| Build environment | ✓ Required | ✓ Required (framework versions, CUDA, hardware) |
| Artifact digest | ✓ Required | ✓ Required (per-file for all components) |
| Input dataset identity | — Not applicable | ✓ Critical (dataset name, version, digest) |
| Base model identity | — Not applicable | ✓ Critical for fine-tuned models |
| Hyperparameters | — Not applicable | Recommended (learning rate, epochs, batch size) |
| Hardware attestation | — Not applicable | Recommended (GPU type affects numerical outputs) |
| Evaluation results | — Not applicable | Recommended (benchmark scores, safety eval results) |
| Model card | — Not applicable | Strongly recommended |

---

### SLSA Applied to AI: Level by Level

#### AI at Level 0 — No Guarantees

**Current state of most publicly distributed AI models.**

A model uploaded directly to Hugging Face from a researcher's workstation, with no CI/CD pipeline, no provenance attestation, and no cryptographic binding between the weights and any training run, is SLSA Level 0. The consumer must trust the uploader unconditionally.

**Real-world risk:** A malicious actor registers a similar-looking Hugging Face organization name (`mistral-community` vs. `mistralai`) and uploads weights that have been backdoored during a manual merge step. No provenance exists to distinguish the legitimate model from the tampered one. This mirrors the typosquatting attacks seen in npm and PyPI, but the payload is a multi-gigabyte weights file rather than a package tarball.

```
# What SLSA L0 looks like for an AI model publish
python train.py --config config.yaml   # Runs on a researcher's A100
# ... days later ...
huggingface-cli upload my-org/my-model ./checkpoints/final/
# No attestation, no digest record, no build log
```

#### AI at Level 1 — Provenance Exists

**Goal:** Establish a machine-readable record of the training run and publish it alongside the model weights.

At Level 1, the training process is scripted and automated (no manual interventions that produce the final artifact), and a provenance document is emitted recording the training run metadata. The provenance need not be signed; its primary value is auditability and tamper-detection after the fact.

**Minimum provenance fields for an AI artifact at L1:**

```json
{
  "predicateType": "https://slsa.dev/provenance/v1",
  "subject": [
    {
      "name": "model.safetensors",
      "digest": { "sha256": "a1b2c3..." }
    },
    {
      "name": "tokenizer.json",
      "digest": { "sha256": "d4e5f6..." }
    }
  ],
  "predicate": {
    "buildDefinition": {
      "buildType": "https://example.com/ai-training/v1",
      "externalParameters": {
        "trainingScript": "train.py",
        "configFile": "config.yaml",
        "configDigest": "sha256:7890ab...",
        "baseModel": "meta-llama/Meta-Llama-3-8B",
        "baseModelDigest": "sha256:cdef01...",
        "dataset": "HuggingFaceH4/ultrachat_200k",
        "datasetRevision": "dc715f4"
      }
    },
    "runDetails": {
      "builder": { "id": "https://github.com/my-org/my-model/actions/runs/9876543210" },
      "metadata": {
        "invocationId": "run-2024-06-01-001",
        "startedOn": "2024-06-01T02:00:00Z",
        "finishedOn": "2024-06-03T14:37:00Z"
      }
    }
  }
}
```

**What Level 1 provides for AI:** A consumer can verify that the downloaded `model.safetensors` matches the digest in the provenance. If the file has been tampered with after training, the hash will not match. The consumer can also inspect which dataset and base model were declared as inputs — though they cannot yet verify that these declarations are truthful.

**Example — automated training pipeline with provenance emission:**

```yaml
# .github/workflows/train.yml
jobs:
  train:
    runs-on: [self-hosted, gpu]
    steps:
      - uses: actions/checkout@v4

      - name: Run training
        run: python train.py --config config.yaml --output-dir ./output

      - name: Compute artifact digests and emit provenance
        run: |
          python scripts/emit_provenance.py \
            --artifacts ./output/model.safetensors ./output/tokenizer.json \
            --base-model "meta-llama/Meta-Llama-3-8B" \
            --dataset "HuggingFaceH4/ultrachat_200k" \
            --run-id "$GITHUB_RUN_ID" \
            --output provenance.json

      - name: Upload model and provenance to Hugging Face
        run: |
          huggingface-cli upload my-org/my-model ./output/
          huggingface-cli upload my-org/my-model provenance.json
```

#### AI at Level 2 — Hosted Build Service

**Goal:** Make the training provenance tamper-evident and independently verifiable. The provenance is generated and signed by the training infrastructure, not by the researcher's workflow code.

This is architecturally more complex for AI than for traditional software, for two reasons:

1. **Training infrastructure is often self-hosted** — GPU clusters are rarely provided by a third-party build service in the same way GitHub Actions provides compute for software builds. This means the "trusted builder" must be the organization's own training platform, which must be hardened and attested independently.

2. **Training is long-running and stateful** — unlike a 5-minute software build, a training run may take days. The provenance service must remain available and consistent across the entire training duration.

**What changes at L2 for AI:**

- The training orchestration system (Kubeflow Pipelines, Metaflow, MLflow, or a custom training platform) — not the training script itself — generates and signs the provenance
- The signing key is inaccessible to the training job
- Provenance is uploaded to a transparency log (Rekor) upon training completion
- The base model's own provenance is resolved and linked as a dependency

**Example — container image for model serving with L2 provenance:**

The model inference container is a traditional software artifact and can achieve L2 straightforwardly using standard Cosign tooling:

```yaml
# The serving container is a normal container build — full L2 achievable
- name: Build and sign serving container
  run: |
    docker build \
      --build-arg MODEL_DIGEST=sha256:a1b2c3... \
      -t ghcr.io/my-org/my-model-server:v1.0.0 .
    docker push ghcr.io/my-org/my-model-server:v1.0.0
    cosign sign --yes \
      --annotations="model.digest=sha256:a1b2c3..." \
      --annotations="model.name=my-org/my-model" \
      ghcr.io/my-org/my-model-server:v1.0.0
```

By embedding the model weights digest as a Cosign annotation on the serving container, a consumer can verify both the container's provenance (L2) and that it was built with a specific, known model artifact.

**Signing raw model weights at L2:**

The OpenSSF Model Signing (OMS) specification — released in 2025 — extends the Sigstore bundle format to cover multi-file AI artifacts:

```bash
# Sign a Hugging Face model directory (all files in a single OMS signature)
pip install model-signing

# Sign using keyless OIDC (from a CI environment with an OIDC token)
python -m model_signing.sign \
  --model_path ./my-model/ \
  --sig_out ./my-model.sig.json \
  --signing_key sigstore

# Verify
python -m model_signing.verify \
  --model_path ./my-model/ \
  --sig_path ./my-model.sig.json \
  --certificate_identity "https://github.com/my-org/my-model/.github/workflows/train.yml@refs/heads/main" \
  --certificate_oidc_issuer "https://token.actions.githubusercontent.com"
```

#### AI at Level 3 — Hardened Builds

**Goal:** Make training provenance non-falsifiable, the training environment hermetic, and the artifact verifiably linked to an auditable, tamper-resistant training run.

Level 3 is the most challenging level for AI workloads and represents the current frontier of research and engineering effort in the field. The core difficulty is that **hermetic, reproducible builds** — the cornerstone of SLSA L3 for software — are not directly achievable for neural network training with current hardware.

**The reproducibility gap:** Floating-point arithmetic on GPUs is not fully associative. The order in which partial sums are accumulated across GPU threads varies with parallelism strategy, producing different rounding errors across runs. Even with fixed random seeds, identical data order, and identical hyperparameters, two training runs on the same hardware type will often produce weights that differ in the last few bits. Across different hardware generations or vendors, divergence can be more significant.

**What L3 means in practice for AI — current best achievable posture:**

| L3 Software Requirement | AI Equivalent |
|------------------------|---------------|
| Hermetic build (no network mid-build) | Training data fully pre-fetched and hash-verified before training begins; no dynamic data downloads during training |
| Ephemeral environment | Training job runs in a freshly provisioned container/VM with no persistent state from prior runs |
| Isolated build | Training job cannot communicate with or influence concurrent training jobs |
| Non-falsifiable provenance | Training orchestrator generates and signs provenance in a privileged sidecar inaccessible to the training process; signed with a hardware-rooted key (TPM or HSM) |
| Reproducible artifact | Not achievable for neural network training — mitigated by hardware attestation and comprehensive environment capture |

**Hardware attestation as the L3 substitute for reproducibility:**

Because exact reproducibility is not achievable, L3-equivalent assurance for AI relies on **hardware attestation** — a cryptographic proof from the compute hardware itself (GPU or TEE) that a specific computation was executed on attested hardware in an unmodified environment. NVIDIA's Hopper architecture (H100) supports Confidential Computing mode, enabling GPU attestation via the NVIDIA Attestation Service. An L3-posture training run would:

1. Boot the training VM in a Trusted Execution Environment (TEE) or with attestable GPU compute
2. Obtain a hardware attestation report from the GPU before training begins
3. Run training with all data pre-staged and network disabled
4. Have a privileged orchestration sidecar sign the provenance — including the hardware attestation report — using an HSM-backed key
5. Upload the signed provenance + hardware attestation to Rekor

```json
{
  "predicate": {
    "buildDefinition": {
      "systemParameters": {
        "hardwareAttestation": {
          "type": "nvidia-hopper-cc",
          "report": "base64-encoded-attestation-report...",
          "attestationServiceVerificationURI": "https://nras.nvidia.com/v1/attestation/gpu"
        },
        "computeEnvironment": {
          "containerDigest": "sha256:abc...",
          "cudaVersion": "12.4",
          "driverVersion": "550.54.15",
          "gpuCount": 8,
          "gpuModel": "NVIDIA H100 SXM5 80GB"
        }
      }
    }
  }
}
```

This does not prove the weights are numerically identical to a reference run, but it does prove that the weights were produced by a specific, attested computation on unmodified hardware — which is the strongest integrity claim currently achievable for neural network training.

---

### Emerging Standards and Tooling

The ecosystem for AI supply chain security is maturing rapidly. The following initiatives are directly relevant to applying SLSA concepts to AI artifacts:

| Standard / Tool | Description | SLSA Relevance |
|----------------|-------------|----------------|
| [OpenSSF Model Signing (OMS)](https://openssf.org/blog/2025/06/25/an-introduction-to-the-openssf-model-signing-oms-specification/) | Sigstore-compatible signing format for multi-file AI model artifacts; supports keyless OIDC signing | L2 signing for model weights |
| [sigstore/model-transparency](https://github.com/sigstore/model-transparency) | Reference implementation of model signing; integrates with Hugging Face and OCI registries | L1/L2 provenance emission and verification |
| [Hugging Face model cards](https://huggingface.co/docs/hub/model-cards) | Structured metadata format for datasets, training details, evaluation results; not cryptographically signed but foundational for L1 provenance | L1 metadata |
| [NVIDIA Hopper Confidential Computing](https://www.nvidia.com/en-us/data-center/solutions/confidential-computing/) | Hardware attestation for GPU training workloads; enables cryptographic proof of execution environment | L3 hardware attestation |
| [GUAC](https://guac.sh) | Graph for Understanding Artifact Composition; ingests SLSA provenance, SBOMs, and OSV data to model supply chain relationships | Verification and policy |
| [MLflow model registry](https://mlflow.org) | Experiment tracking and model versioning with lineage metadata; can emit provenance data | L1 provenance |
| [DVC (Data Version Control)](https://dvc.org) | Git-based versioning for large datasets and model files with content-addressed storage | Dataset provenance |
| [MLBOM / AI BOM](https://owasp.org/www-project-ai-security-and-privacy-guide/) | Machine Learning Bill of Materials — SBOM analogue for AI; captures datasets, base models, frameworks, and evaluation results | Complements SLSA provenance |

---

### AI as a Builder: Risks of AI-Generated Code

The preceding discussion treats AI models as *artifacts to be protected*. There is a second, increasingly important dimension: AI as an active participant in the software build process itself — specifically, AI coding assistants and autonomous agents that write, modify, and commit source code.

When a developer uses GitHub Copilot, Cursor, or Claude to generate code that is committed to a repository and built into a release artifact, a question arises: **does SLSA provenance capture the AI's participation in authoring the source?**

Under current SLSA v1.0, provenance captures the transformation of committed source code into an artifact. It does not capture how that source code came to be. The commit SHA in the provenance attests that the artifact was built from a specific commit — it does not attest to the authorship of that commit's content.

**This creates a provenance gap for AI-generated code:**

- A SLSA L3 artifact can have fully verified, non-falsifiable build provenance while being built from source code that was entirely generated by an AI system, with no human review
- The supply chain integrity of the build process is fully assured; the integrity of the *authorship* process is not captured at all
- An autonomous AI agent that commits malicious code to a repository — whether due to a prompt injection attack, a compromised model, or deliberate misuse — produces a SLSA-attested artifact with valid provenance

**Risk scenarios specific to AI-assisted development:**

| Scenario | SLSA Detection | Notes |
|----------|---------------|-------|
| Developer uses AI to write code; reviews and commits it | ✗ Not detected — normal commit | Acceptable; human review is the control |
| AI agent autonomously commits code without human review | ✗ Not detected by SLSA | Control must be at source track level (branch protection, required reviews) |
| Prompt injection causes AI agent to commit a backdoor | ✗ Not detected by SLSA | SLSA attests the build, not the commit's semantic content |
| AI generates a malicious dependency in `package.json` | ✗ Not detected at L1/L2 | Dependency provenance verification (SLSA for deps) can surface this |
| AI coding assistant suggests a supply chain attack payload | ✗ Not detected | Out of scope for SLSA entirely |

**Mitigations at the source track level** (complements SLSA build track):

SLSA's **source track** (currently in draft) begins to address this gap. It introduces requirements for source integrity — including two-person review requirements and version control platform controls — that would apply to AI-authored commits. Until the source track matures, the primary controls are:

1. **Branch protection rules** — require human review and approval of all commits before merge, regardless of authorship
2. **Commit signing** — require GPG or SSH-signed commits; an AI agent acting autonomously cannot produce a valid human signature
3. **AI authorship disclosure** — emerging tools (e.g., GitHub's AI code attribution) can annotate commits as AI-assisted; this is auditable but not yet cryptographically enforced
4. **Agent permission scoping** — restrict autonomous coding agents to read-only repository access; require human approval before any commit is accepted

```yaml
# .github/branch_protection.yml (conceptual — configure via API or UI)
# Enforce human review even when AI agents participate in PRs
required_pull_request_reviews:
  required_approving_review_count: 1
  dismiss_stale_reviews: true
  require_code_owner_reviews: true
  # Note: GitHub Copilot-generated PR suggestions still require human approval
restrict_pushes:
  allow_actors: []   # No direct push to main — all changes via reviewed PR
```

**The forward trajectory:** As AI agents become first-class participants in software development — writing code, running tests, opening PRs, and merging changes — the distinction between "source authored by a human" and "source generated by an AI" will require explicit provenance representation. The SLSA community and OpenSSF AI/ML Working Group are actively developing extensions to address this. The most likely near-term approach is extending the `externalParameters` field to include AI tool identity and version, and adding optional `aiAssistance` metadata to provenance predicates.

---

## 12. References

**SLSA Core**

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

**AI Supply Chain Security**

- [OpenSSF Model Signing (OMS) Specification](https://openssf.org/blog/2025/06/25/an-introduction-to-the-openssf-model-signing-oms-specification/) — Sigstore-compatible signing standard for ML model artifacts
- [sigstore/model-transparency](https://github.com/sigstore/model-transparency) — Reference implementation of AI model signing and provenance
- [Google: Securing the AI Software Supply Chain](https://research.google/pubs/securing-the-ai-software-supply-chain/) — Google's approach to adapting SLSA for AI pipelines
- [Google Cloud: AI Supply Chain Security Guidance](https://cloud.google.com/transform/same-same-but-also-different-google-guidance-ai-supply-chain-security/) — Practical guidance on AI provenance and SLSA adaptation
- [GUAC — Graph for Understanding Artifact Composition](https://guac.sh) — Supply chain knowledge graph ingesting SLSA, SBOM, and OSV data
- [NVIDIA Hopper Confidential Computing](https://www.nvidia.com/en-us/data-center/solutions/confidential-computing/) — GPU hardware attestation for training workloads
- [DVC — Data Version Control](https://dvc.org) — Dataset and model versioning with content-addressed storage
- [Hugging Face Model Cards](https://huggingface.co/docs/hub/model-cards) — Structured metadata specification for AI models
- [OWASP AI Security and Privacy Guide](https://owasp.org/www-project-ai-security-and-privacy-guide/) — AI BOM and threat modelling for AI systems
- [OpenSSF AI/ML Working Group](https://github.com/ossf/ai-ml-security) — Standards development for AI supply chain security

---

*Specification reference: SLSA v1.0 · Last reviewed: 2025 · Maintained at [github.com/cjohannsen81/slsa-guide](https://github.com/cjohannsen81/slsa-guide)*
