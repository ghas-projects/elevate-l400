# SBOMs, Attestation, and SLSA
The previous chapters covered how GitHub discovers and tracks your dependencies  - through static analysis, the Dependency Submission API, and automatic dependency submission. This chapter addresses what happens after you know what your software is made of: proving it to others.

A Software Bill of Materials (SBOM) answers the question **"what is in this software?"** An artifact attestation answers **"where and how was this software built?"** Together, they form the evidence layer that auditors, regulators, and downstream consumers increasingly require. The SLSA framework provides a maturity model to measure how trustworthy that evidence is.

## SBOM Generation

### What Is an SBOM?

A Software Bill of Materials is a machine-readable inventory of every component in a software artifact - direct dependencies, transitive dependencies, versions, package identifiers, licenses, and relationships. Think of it as a nutritional label for software.

Two formats dominate the industry:

- SPDX (Software Package Data Exchange) — an ISO/IEC standard (ISO/IEC 5962:2021) maintained by the Linux Foundation. This is the format GitHub uses natively for SBOM export (SPDX via the dependency graph UI and REST API).
- CycloneDX — an OWASP standard designed for security use cases, with native support for vulnerability and service metadata. GitHub does not natively generate or export CycloneDX, but CycloneDX SBOMs can be submitted to the dependency graph via third-party actions.

### Why SBOMs Matter

SBOMs are no longer optional for many organizations. Regulatory frameworks increasingly require them:

- The US Executive Order 14028(2021) mandates SBOM delivery for software sold to the federal government.
- The EU Cyber Resilience Act requires manufacturers to document software components for products sold in the EU.
- FedRAMP, NIST SP 800-218, and industry-specific standards (e.g., FDA for medical devices) all reference SBOMs as a baseline expectation.

Beyond compliance, SBOMs enable rapid incident response. When a zero-day like Log4Shell drops, an organization with SBOMs for every deployed artifact can answer "are we affected?" in minutes instead of days.

### Generating SBOMs on GitHub

GitHub provides three ways to generate SBOMs:

#### 1. Export from the Dependency Graph (UI or REST API)

The simplest path. Navigate to your repository's Insights → Dependency graph and click Export SBOM, or call the [SBOM REST API endpoint](https://docs.github.com/en/enterprise-cloud@latest/rest/dependency-graph/sboms#export-a-software-bill-of-materials-sbom-for-a-repository). This exports the current state of the dependency graph in SPDX format.

**Limitations:** This export reflects whatever the dependency graph currently knows. If you haven't enriched the graph with dependency submissions, the SBOM will only contain statically analyzed dependencies.

#### 2. Generate from GitHub Actions

For SBOMs that need to be produced as part of your CI/CD pipeline (e.g., attached to a release or submitted for audit), use an action:

| Action | Format | Description |
|---|---|---|
| [SPDX Dependency Submission Action](https://github.com/marketplace/actions/spdx-dependency-submission-action) | SPDX 2.2 | Uses Microsoft's SBOM Tool |
| [Anchore SBOM Action](https://github.com/marketplace/actions/anchore-sbom-action) | SPDX / CycloneDX | Uses Syft for multi-ecosystem scanning |
| [SBOM Dependency Submission Action](https://github.com/marketplace/actions/sbom-dependency-submission-action) | CycloneDX | Uploads a CycloneDX SBOM to the Dependency Submission API |

These actions can both generate the SBOM and submit the dependency data back to the graph, closing the loop between SBOM production and vulnerability monitoring.

#### 3. External Tooling

If your organization already produces SBOMs through tools like Syft, Trivy, or Microsoft's SBOM Tool, you can submit those SBOMs to the dependency graph using the Dependency Submission API (covered in the previous chapter) and attest them using the process described below.

## Artifact Attestations

### The Problem Attestations Solve

An SBOM tells you what is in your software. But how do you know the SBOM is accurate? How do you know the binary you are running was actually produced by the CI pipeline you trust, from the source code you reviewed? Without a cryptographic link between an artifact and its build process, any of these claims can be forged.

Artifact attestations solve this by creating cryptographically signed, unfalsifiable claims about how an artifact was built. Each attestation includes:

- A link to the GitHub Actions workflow that produced the artifact.
- The repository, organization, environment, commit SHA, and triggering event.
- OIDC-based identity from the build platform, establishing provenance without managing long-lived signing keys.

### How GitHub Implements Attestations

GitHub uses [Sigstore](https://www.sigstore.dev/), an open-source project for signing and verifying software artifacts:

- **Public repositories** use the Sigstore Public Good Instance. Attestations are written to an immutable transparency log that is publicly readable.
- **Private repositories** use GitHub's own Sigstore instance, which uses the same codebase but does not write to a public transparency log. Attestations are stored with GitHub.

In both cases, signing is ephemeral - short-lived certificates are issued based on the OIDC identity of the GitHub Actions workflow run. There are no long-lived keys to rotate or protect.

### Two Types of Attestations

GitHub supports two attestation types through first-party actions:

#### Build Provenance Attestation

The [`actions/attest-build-provenance`](https://github.com/actions/attest-build-provenance) action generates a signed statement that a specific artifact (binary or container image) was produced by a specific workflow run.

```yaml
- name: Generate artifact attestation
  uses: actions/attest-build-provenance@v3
  with:
    subject-path: 'path/to/artifact'
```

#### SBOM Attestation

The [`actions/attest-sbom`](https://github.com/actions/attest-sbom) action generates a signed statement linking an SBOM to the artifact it describes. This proves that a particular SBOM was produced for a particular artifact during a particular build.

```yaml
- name: Generate SBOM attestation
  uses: actions/attest-sbom@v2
  with:
    subject-path: 'path/to/artifact'
    sbom-path: 'path/to/sbom.spdx.json'
```

Both actions require the following workflow permissions:

```yaml
permissions:
  id-token: write
  contents: read
  attestations: write
```

For container images, add `packages: write` and use `subject-name` / `subject-digest` instead of `subject-path`.

### Verifying Attestations

Consumers verify attestations using the GitHub CLI:

```bash
# Verify build provenance for a binary
gh attestation verify path/to/artifact -R org/repo

# Verify build provenance for a container image
gh attestation verify oci://ghcr.io/org/image:tag -R org/repo

# Verify an SBOM attestation (SPDX format)
gh attestation verify path/to/artifact -R org/repo \
  --predicate-type https://spdx.dev/Document/v2.3
```

Verification confirms that the attestation's signature is valid, that it was signed by the expected repository's workflow, and that the artifact's digest matches.

> **Important:** Attestations are not a guarantee that an artifact is secure. They guarantee that the artifact was built where and how the attestation claims. It is up to consumers to evaluate whether that build process meets their security requirements.

### When to Attest

Attest artifacts that others will consume - release binaries, published packages, container images. Do not attest ephemeral build outputs like test binaries or intermediate artifacts. The goal is to create a verifiable chain of custody for software that leaves your organization.

### Availability

Artifact attestations are available for all current GitHub plans for public repositories. For private and internal repositories, they require GitHub Enterprise Cloud.

## SLSA and GitHub

### What Is SLSA?

[SLSA](https://slsa.dev/) (Supply-chain Levels for Software Artifacts, pronounced "salsa") is an industry framework maintained by the OpenSSF that defines increasing levels of supply chain security guarantees. Version 1.0 focuses on a single Build track with four levels:

| Level | Name | What It Proves | Threats Addressed |
|---|---|---|---|
| Build L0 | No guarantees | Nothing | None |
| Build L1 | Provenance exists | How the package was built | Mistakes, documentation |
| Build L2 | Hosted build platform | Signed provenance from a hosted platform | Tampering after the build |
| Build L3 | Hardened builds | Provenance from a hardened, isolated build platform | Tampering during the build |

### Where GitHub Fits

GitHub Actions with artifact attestations maps directly to the SLSA Build track:

- **SLSA Build L1** is achieved by using GitHub Actions to build your software and generating any form of provenance (even unsigned build logs).
- **SLSA Build L2** is achieved by using `actions/attest-build-provenance` in a GitHub Actions workflow. The attestation is signed via Sigstore and tied to GitHub's hosted build infrastructure.
- **SLSA Build L3** is achieved by combining artifact attestations with **reusable workflows**. Reusable workflows provide isolation between the build process and the calling workflow - the caller cannot modify the build steps, and the signing happens inside the reusable workflow. This meets SLSA's requirement that the build platform prevents runs from influencing one another and keeps signing material inaccessible to user-defined steps.

### Achieving SLSA Build L3 on GitHub

The path to Build L3 requires two components:

1. A reusable workflow that contains your build and attestation steps. This workflow lives in a dedicated repository (often shared across your organization) and is called by individual project workflows.

2. Attestation generation inside the reusable workflow, so the signing identity is tied to the reusable workflow rather than the caller.

When verifying, consumers can use the `--signer-repo` and `--signer-workflow` flags to confirm the attestation was produced by the expected reusable workflow:

```bash
gh attestation verify path/to/artifact \
  -o my-org \
  --signer-repo my-org/build-workflows \
  --signer-workflow my-org/build-workflows/.github/workflows/build-and-attest.yml
```

This proves not just that the artifact came from your organization, but that it was built using your organization's vetted, centrally managed build process.

### Connecting the Pieces

The following diagram shows how the supply chain security features covered in this course relate to each other and map to the SLSA framework:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Source Code                              │
│         (manifests, lock files, source committed to repo)       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │    Dependency Graph      │ ◄── Static analysis
              │    + Submissions         │ ◄── Dependency Submission API
              └────────────┬────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    Dependabot        Dependency       SBOM Export
      Alerts           Review
                                           │
                           ┌───────────────▼───────────────┐
                           │        Build Pipeline         │
                           │     (GitHub Actions)          │
                           └───────┬───────────┬───────────┘
                                   │           │
                                   ▼           ▼
                             Build Prov.   SBOM Attestation
                             Attestation
                                   │           │
                                   ▼           ▼
                           ┌───────────────────────────────┐
                           │   Signed, Verifiable Artifact │
                           │   (SLSA Build L2 / L3)        │
                           └───────────────────────────────┘
```

## Operational Considerations

### Attestation Lifecycle

Attestations accumulate over time. GitHub provides UI and APIs to manage their lifecycle - delete attestations that are no longer relevant (e.g., for deprecated releases) to keep your attestation store manageable. See [Managing the lifecycle of artifact attestations](https://docs.github.com/en/enterprise-cloud@latest/actions/how-tos/security-for-github-actions/using-artifact-attestations/managing-the-lifecycle-of-artifact-attestations).

### Offline Verification

For air-gapped or restricted environments, `gh attestation verify` supports offline verification. See [Verifying attestations offline](https://docs.github.com/en/enterprise-cloud@latest/actions/security-guides/verifying-attestations-offline).

### Linked Artifacts

GitHub Enterprise Cloud users can upload attested assets to a linked artifacts page, which displays build history, deployment records, and storage details. This connects vulnerable artifacts to their owning team, source code, and build run for faster triage. See [About linked artifacts](https://docs.github.com/en/enterprise-cloud@latest/code-security/concepts/supply-chain-security/linked-artifacts).

### Adoption Strategy

For organizations adopting these capabilities incrementally:

| Maturity Stage | Actions to Take |
|---|---|
| **Foundation** | Enable the dependency graph and dependency submission across all repositories. Export SBOMs on demand. |
| **Provenance** | Add `attest-build-provenance` to release workflows. Achieve SLSA Build L2. |
| **Integrity** | Add `attest-sbom` to release workflows. Attach SBOMs to releases alongside attestations. |
| **Hardened builds** | Move build and attestation steps into shared reusable workflows. Achieve SLSA Build L3. |
| **Enforcement** | Require `gh attestation verify` in deployment pipelines. Reject unattested or untrusted artifacts. |

## Further Reading

- [Exporting a software bill of materials for your repository](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/exporting-a-software-bill-of-materials-for-your-repository)
- [Artifact attestations (concepts)](https://docs.github.com/en/enterprise-cloud@latest/actions/concepts/security/artifact-attestations)
- [Using artifact attestations to establish provenance for builds](https://docs.github.com/en/enterprise-cloud@latest/actions/security-guides/using-artifact-attestations-to-establish-provenance-for-builds)
- [Using artifact attestations and reusable workflows to achieve SLSA v1 Build Level 3](https://docs.github.com/en/enterprise-cloud@latest/actions/security-guides/using-artifact-attestations-and-reusable-workflows-to-achieve-slsa-v1-build-level-3)
- [SLSA Specification v1.0 — Security Levels](https://slsa.dev/spec/v1.0/levels)
- [REST API endpoints for SBOMs](https://docs.github.com/en/enterprise-cloud@latest/rest/dependency-graph/sboms)
- [Sigstore](https://www.sigstore.dev/)