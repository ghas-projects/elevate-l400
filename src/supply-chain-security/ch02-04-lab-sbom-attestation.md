# Lab: SBOM Generation, Attestation, and SLSA Build L3

This hands-on exercise walks you through the full lifecycle of supply chain evidence on GitHub - from generating an SBOM, to attesting it alongside your build artifact, to refactoring the workflow into a reusable workflow that achieves SLSA Build L3.

You will work with [**fzf**](https://github.com/junegunn/fzf), a popular open-source command-line fuzzy finder written in Go. It has real transitive dependencies, making its SBOM meaningful. The concepts apply to any ecosystem.

---

## Part 1 - Generate an SBOM

**Goal:** Produce an SPDX SBOM for a build artifact and upload it as a workflow artifact.

### 1.1 Navigate to the repository

1. Navigate to the `fzf` repository in your lab organization: `ghas-labs-2026-03-16-<HANDLE>/fzf`.

> **Why fzf?** It's a small, well-maintained Go project with real transitive dependencies. The SBOM it produces will contain dozens of packages that are representative of a real-world project rather than a toy example.

### 1.2 Create the build-and-sbom workflow

Create `.github/workflows/build-and-sbom.yml` with a workflow that:

1. Checks out the repository.
2. Sets up Go using the version from `go.mod`.
3. Builds the binary (`go build -o fzf .`).
4. Generates an SPDX SBOM using [`anchore/sbom-action`](https://github.com/anchore/sbom-action) and submits the dependencies to the dependency graph (using `dependency-snapshot: true`).
5. Uploads both the binary and the SBOM as workflow artifacts.

> **Note:** Submitting the SBOM's dependencies to the dependency graph enables Dependabot alerts for any vulnerabilities found in the SBOM. This requires `contents: write` in the workflow permissions.

The workflow should trigger on pushes to `main` and support `workflow_dispatch`.

<details>
<summary>Hint: Which action generates the SBOM?</summary>

Use `anchore/sbom-action@v0` with the following inputs:
- `path: .`
- `format: spdx-json`
- `output-file: sbom.spdx.json`
- `dependency-snapshot: true`

</details>

<details>
<summary>Solution: Complete workflow file</summary>

```yaml
name: Build and Generate SBOM

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: go.mod

      - name: Build binary
        run: go build -o fzf .

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          path: .
          format: spdx-json
          output-file: sbom.spdx.json
          dependency-snapshot: true

      - name: Upload binary
        uses: actions/upload-artifact@v4
        with:
          name: fzf
          path: fzf

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json
```

</details>

### 1.3 Run and verify

1. Commit and push to `main` (or use the Run workflow button on the Actions tab if you only added the file via the UI).
2. Open the Actions tab and inspect the completed run.
3. Download the `sbom` artifact and open `sbom.spdx.json`. You should see fzf's Go module along with its transitive dependencies listed in SPDX format.

<details>
<summary>Git Commands</summary>

```bash
git add .github/workflows/build-and-sbom.yml
git commit -m "Add SBOM generation workflow"
git push origin main
```

</details>

> **Checkpoint:** You now have a reproducible SBOM generated in CI. At this point, anyone can claim this SBOM belongs to your artifact — there is no cryptographic proof. That's what Part 2 solves.

---

## Part 2 - Attest the Build and SBOM

**Goal:** Add cryptographically signed attestations for both build provenance and the SBOM, then verify them locally.

### 2.1 Update the workflow

Update your workflow to add artifact attestations. You need to:

1. Add to `permissions` block with `id-token: write`, `attestations: write` and keep `contents: write`.
2. After the build and SBOM generation steps, add two attestation steps using [`actions/attest`](https://github.com/actions/attest):
   - One for build provenance (just `subject-path`).
   - One for SBOM attestation (both `subject-path` and `sbom-path`).

> **Hint:** `actions/attest@v4` automatically detects the attestation mode. Without `sbom-path`, it generates SLSA build provenance. With `sbom-path`, it generates an SBOM attestation. Both are signed via Sigstore.

<details>
<summary>Hint: What permissions are required?</summary>

```yaml
permissions:
  id-token: write      # Mint OIDC token for Sigstore signing
  contents: read        # Checkout the repository
  attestations: write   # Persist attestations to the GitHub API
```

</details>

<details>
<summary>Hint: Attestation step syntax</summary>

```yaml
- name: Attest build provenance
  uses: actions/attest@v4
  with:
    subject-path: fzf

- name: Attest SBOM
  uses: actions/attest@v4
  with:
    subject-path: fzf
    sbom-path: sbom.spdx.json
```

</details>

<details>
<summary>Solution: Complete workflow file</summary>

```yaml
name: Build, SBOM, and Attest

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: write
  attestations: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: go.mod

      - name: Build binary
        run: go build -o fzf .

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          path: .
          format: spdx-json
          output-file: sbom.spdx.json
          dependency-snapshot: true

      - name: Attest build provenance
        uses: actions/attest@v4
        with:
          subject-path: fzf

      - name: Attest SBOM
        uses: actions/attest@v4
        with:
          subject-path: fzf
          sbom-path: sbom.spdx.json

      - name: Upload binary
        uses: actions/upload-artifact@v4
        with:
          name: fzf
          path: fzf

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json
```

</details>

### 2.2 Run the workflow

Commit and push. After the workflow completes, navigate to the run summary. You should see an Attestations section listing both the provenance and SBOM attestations.

### 2.3 Optional: Verify locally

Download the `fzf` artifact, then verify both attestations using the GitHub CLI. If you do not have the CLI installed locally, see the [installation instructions](https://github.com/cli/cli#installation).

Try to:
1. Verify the build provenance attestation.
2. Verify the SBOM attestation (you'll need to specify the SPDX predicate type).
3. Inspect the full SBOM predicate from the attestation as JSON.

<details>
<summary>Solution: Verification commands</summary>

```bash
# Verify build provenance
gh attestation verify fzf -R <your-username>/fzf

# Verify the SBOM attestation (specify the SPDX predicate type)
gh attestation verify fzf -R <your-username>/fzf \
  --predicate-type https://spdx.dev/Document/v2.3

# Inspect the full SBOM predicate from the attestation
gh attestation verify fzf -R <your-username>/fzf \
  --predicate-type https://spdx.dev/Document/v2.3 \
  --format json \
  --jq '.[].verificationResult.statement.predicate'
```

</details>

Each `verify` command confirms that:
- The attestation signature is valid (signed by Sigstore).
- The signing identity matches the expected repository and workflow.
- The artifact digest matches the attested subject.

> **Checkpoint:** You now have SLSA Build L2 signed provenance from a hosted build platform. The attestation is tied to this specific workflow file in your repository. However, anyone with write access could modify the workflow steps. Part 3 addresses this.

---

## Part 3 - Refactor to a Reusable Workflow (SLSA Build L3)

**Goal:** Move the build and attestation logic into a reusable workflow so that the signing identity is tied to the reusable workflow, not the caller. This achieves SLSA Build L3 - callers cannot tamper with the build or signing steps.

### 3.1 Create the reusable workflow

In a separate repository (e.g., `<your-org>/build-workflows`), create a reusable workflow at `.github/workflows/build-attest-go.yml` that:

1. Uses `workflow_call` as its trigger.
2. Accepts inputs for `go-version-file` (default: `go.mod`) and `binary-name` (default: `my-app`).
3. Declares the same `permissions` as Part 2.
4. Contains all the build, SBOM generation, attestation, and upload steps from your current workflow.

Think about: Why does putting attestation inside the reusable workflow matter for SLSA L3?

<details>
<summary>Hint</summary>

```yaml
on:
  workflow_call:
    inputs:
      go-version-file:
        description: 'Path to go.mod for version detection'
        required: false
        type: string
        default: 'go.mod'
      binary-name:
        description: 'Name of the output binary'
        required: false
        type: string
        default: 'my-app'
```

Make sure your GitHub action workflow can be accessed from other repositories in the Organization.

</details>

<details>
<summary>Solution: Complete reusable workflow</summary>

**`<your-org>/build-workflows/.github/workflows/build-attest-go.yml`**

```yaml
name: Build, SBOM, and Attest (Reusable)

on:
  workflow_call:
    inputs:
      go-version-file:
        description: 'Path to go.mod for version detection'
        required: false
        type: string
        default: 'go.mod'
      binary-name:
        description: 'Name of the output binary'
        required: false
        type: string
        default: 'my-app'

permissions:
  id-token: write
  contents: write
  attestations: write

jobs:
  build-and-attest:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout caller repository
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: ${{ inputs.go-version-file }}

      - name: Build binary
        run: go build -o ${{ inputs.binary-name }} .

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          path: .
          format: spdx-json
          output-file: sbom.spdx.json
          dependency-snapshot: true

      - name: Attest build provenance
        uses: actions/attest@v4
        with:
          subject-path: ${{ inputs.binary-name }}

      - name: Attest SBOM
        uses: actions/attest@v4
        with:
          subject-path: ${{ inputs.binary-name }}
          sbom-path: sbom.spdx.json

      - name: Upload binary
        uses: actions/upload-artifact@v4
        with:
          name: ${{ inputs.binary-name }}
          path: ${{ inputs.binary-name }}

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json
```

</details>

### 3.2 Update the caller workflow

Back in your fzf fork, replace the workflow with a thin caller that delegates to the reusable workflow. The caller should:

1. Trigger on push to `main` and `workflow_dispatch`.
2. Declare the same `permissions` block.
3. Use `uses:` to call the reusable workflow, passing `binary-name: fzf`.

<details>
<summary>Solution: Caller workflow</summary>

**`.github/workflows/build-and-sbom.yml`**

```yaml
name: Build, SBOM, and Attest

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: write
  attestations: write

jobs:
  build:
    uses: <your-org>/build-workflows/.github/workflows/build-attest-go.yml@main
    with:
      binary-name: fzf
```

</details>

> **Key point:** The caller workflow cannot modify the build or attestation steps. The signing identity recorded in the attestation will reference the reusable workflow (`<your-org>/build-workflows/.github/workflows/build-attest-go.yml`), not the caller. This is what elevates trust from L2 to L3.

### 3.3 Run and verify with signer constraints

Commit and push. After the workflow completes, verify the attestation specifying the expected signer. Use the `--signer-repo` and `--signer-workflow` flags to confirm the attestation was produced by the reusable workflow.

<details>
<summary>Solution: Verification commands</summary>

```bash
# Verify provenance, constraining the signer to the reusable workflow repo
gh attestation verify fzf \
  -o <your-org> \
  --signer-repo <your-org>/build-workflows

# Verify with the exact workflow file
gh attestation verify fzf \
  -o <your-org> \
  --signer-workflow <your-org>/build-workflows/.github/workflows/build-attest-go.yml
```

</details>

The `--signer-repo` flag confirms the attestation was signed by a workflow in the expected repository. The `--signer-workflow` flag goes further and confirms the exact workflow file. If someone bypassed the reusable workflow and built directly, verification would fail.

### 3.4 Pin the reusable workflow to a SHA

For production use, pin the reusable workflow reference to a commit SHA rather than a branch.

<details>
<summary>Solution: SHA-pinned caller</summary>

```yaml
jobs:
  build:
    uses: <your-org>/build-workflows/.github/workflows/build-attest-go.yml@<commit-sha>
    with:
      binary-name: fzf
```

</details>

This ensures the caller cannot silently pick up changes to the reusable workflow. Combined with branch protection on the `build-workflows` repository, this gives you a tamper-resistant build pipeline.

> **Checkpoint:** You now have SLSA Build L3 hardened builds with provenance from a reusable workflow that callers cannot modify. Consumers can verify not just that the artifact came from your organization, but that it was built using your organization's vetted, centrally managed build process.

---

## Summary

| Part | What You Built | SLSA Level Achieved |
|---|---|---|
| Part 1 | SBOM generation in CI | Build L1 (provenance exists as a workflow artifact) |
| Part 2 | Signed build provenance + SBOM attestation | Build L2 (signed provenance from hosted platform) |
| Part 3 | Reusable workflow with attestation | Build L3 (hardened, isolated build process) |

### Discussion Points

- Why is it important to attest the SBOM separately from build provenance? What does each attestation prove?
- What would happen if a developer modified the caller workflow to skip the reusable workflow and attest directly? How would verification catch this?
- How would you enforce that only reusable-workflow-built artifacts are deployed?
- What are the trade-offs of pinning reusable workflows to a SHA vs. a branch?

### What to Explore Next

- Container images: Adapt the workflow to build and push a Docker image to GHCR, using `subject-name` and `subject-digest` instead of `subject-path`. Add `packages: write` to the permissions.
- Deployment gates: Add `gh attestation verify` as a step in your deployment pipeline to reject unattested artifacts.
- Attestation lifecycle: Use `gh attestation delete` to clean up attestations for deprecated releases.
- Offline verification: Explore `gh attestation verify --bundle` for air-gapped environments.
