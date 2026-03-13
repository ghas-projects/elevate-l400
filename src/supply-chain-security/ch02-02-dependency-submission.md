# Dependency Submission

The dependency graph's static manifest and lock file analysis covers the majority of common ecosystems, but it has a fundamental limitation: it only sees what is committed to the repository. Dependencies resolved at build time, vendored without a package manager, or managed by build systems that GitHub does not natively parse are invisible to the graph and therefore invisible to dependabot alerts, dependency review, and SBOM exports.

The Dependency Submission API exists to close this gap. It lets you programmatically submit dependency snapshots to the dependency graph from any source, ensuring that every dependency your project actually uses is tracked, scanned, and auditable.

## Why This Matters

Recall the principle from the dependency graph chapter: if a dependency is not in the graph, none of GitHub's supply chain security features will detect it. Some real life examples of where there could be an incomplete dependency graph are:

- A C++ project with vendored libraries has zero vulnerability coverage unless those libraries are submitted to the graph.
- A Bazel monorepo that resolves hundreds of transitive dependencies at build time will show an incomplete dependency tree from static analysis alone.
- A Maven project pulling artifacts from an internal Nexus registry may have transitive dependencies that only materialize during `mvn compile`.

The Dependency Submission API turns the dependency graph from a static parser into a flexible ingestion layer - any system that can produce a list of packages can feed the graph.

## How It Works

[The API accepts snapshots.](https://docs.github.com/en/enterprise-cloud@latest/rest/dependency-graph/dependency-submission?apiVersion=2022-11-28#create-a-snapshot-of-dependencies-for-a-repository) These are payloads that describe a set of dependencies observed at a specific point in time. Each snapshot includes:

- Detector metadata - the name and version of the tool that produced the snapshot, so you can trace provenance.
- Correlator - a unique identifier that links the snapshot to its source (e.g., a specific workflow run or build step). When a new snapshot is submitted with the same correlator, it replaces the previous one.
- Manifests - one or more manifest objects, each containing a list of resolved packages with their names, versions, and relationship type (direct or indirect).

Once submitted, these dependencies appear in the dependency graph alongside those discovered by static analysis. They are matched against the GitHub Advisory Database just like any other dependency, meaning they will trigger Dependabot alerts if vulnerabilities are found.

> **Key detail:** The dependency graph uses a deduplication and precedence model when the same manifest appears from multiple sources. User-submitted snapshots (via the API) take highest priority, followed by automatic submissions, with static analysis results used as a fallback. This means API submissions will override not duplicate static analysis results for the same manifest.

## Pre-Built Actions

The simplest way to use the Dependency Submission API is through [pre-built GitHub Actions](https://docs.github.com/en/enterprise-server@3.20/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/using-the-dependency-submission-api#using-pre-made-actions) that handle dependency resolution and snapshot formatting for you. These actions run as part of your CI workflow, resolve dependencies using the ecosystem's native tooling, and submit the results to the API.


## Automatic Dependency Submission

For supported ecosystems, GitHub can handle the entire submission workflow for you without requiring you to author or maintain a GitHub Actions workflow. Automatic dependency submission is a built-in feature that watches for pushes to the default branch, resolves transitive dependencies using the ecosystem's native tooling, and submits the results to the dependency graph automatically.

### Prerequisites

- The dependency graph must be enabled for the repository.
- GitHub Actions must be enabled for the repository (the automatic submission runs as a managed Actions workflow).

### Enabling It

Automatic Dependency Submission can be enabled at the repository, organization or enterprise level. Each scope carries different trade-offs around control, rollout speed, and policy enforcement. These are covered in the sections that follow.


Once enabled, GitHub will:

- Watch for pushes to the default branch.
- Run the ecosystem-appropriate dependency resolution action for any manifests in the repository.
- Submit the resolved dependency tree to the graph.

You can view the workflow runs in the repository's Actions tab.


### Private Registries and Self-Hosted Runners

If your dependencies are hosted in private registries (e.g., an internal Nexus or Artifactory instance), automatic submission on GitHub-hosted runners will fail because it cannot reach your registry. Depending on the ecosystem, you may need to fork the underlying action and extend it to support your registry or proxy logic.

For ecosystems where self-hosted runner support is sufficient, you can run the automatic submission on self-hosted runners:

1. Provision self-hosted runners (Linux or macOS) at the repository or organization level.
2. Assign the `dependency-submission` label to each runner.
3. In the repository's Advanced Security settings, set Automatic dependency submission to Enabled for labeled runners.

#### Ecosystem-Specific Caveats

- Maven / Gradle: For projects pulling from private Maven registries, you will also need to configure the runner's Maven `settings.xml` (or Gradle equivalent) with credentials for your private registry.
- .NET and Python: Self-hosted runners still require public internet access to download the latest [`microsoft/component-detection`](https://github.com/microsoft/component-detection/) release used by the autosubmission action.
- Python: Autosubmission does not currently support private packages. Packages in `requirements.txt` that are not publicly available will cause the action to fail. For Python projects with private dependencies, use a custom action built with the Dependency Submission Toolkit instead.


### Deduplication Behavior

When a repository uses both static analysis and dependency submission (manual or automatic), the dependency graph deduplicates manifest entries using the following precedence:

1. User submissions(via the API or pre-built actions) — highest priority, as they are produced during actual builds.
2. Automatic submissions — second priority, also produced during builds but not explicitly curated.
3. Static analysis — used as a fallback when no submission data exists.

This means you do not need to worry about duplicate entries when combining approaches.

## Submitting SBOMs as Snapshots

If your organization already generates Software Bills of Materials (SBOMs) through external tooling, you can submit them to the dependency graph. GitHub's native SBOM support uses **SPDX**. This is the format used by the dependency graph export and the REST API. CycloneDX SBOMs can be ingested via third-party actions but are not natively generated or exported by GitHub.

Several actions can bridge external SBOM tooling with the dependency graph:

- [SPDX Dependency Submission Action](https://github.com/marketplace/actions/spdx-dependency-submission-action) - generates an SPDX SBOM and submits it.
- [Anchore SBOM Action](https://github.com/marketplace/actions/anchore-sbom-action) - generates and submits SBOMs using Syft (supports both SPDX and CycloneDX output).
- [SBOM Dependency Submission Action](https://github.com/marketplace/actions/sbom-dependency-submission-action) - uploads an existing CycloneDX SBOM to the Dependency Submission API.

This is particularly useful for organizations that already have SBOM generation as part of their release pipeline. Rather than building a separate dependency submission workflow, you can reuse your existing SBOM output.

## The Dependency Submission Toolkit

For ecosystems or internal build systems where no pre-built action exists, GitHub provides the [Dependency Submission Toolkit](https://github.com/github/dependency-submission-toolkit) - a TypeScript library for building your own GitHub Action that submits dependencies to the API.

The toolkit handles:

- Constructing the snapshot JSON payload in the correct format.
- Authenticating with the Dependency Submission API.
- Submitting and validating the snapshot.

Your custom action only needs to:

1. Generate a list of dependencies (by invoking your build tool, parsing a lock file, or querying a registry).
2. Map each dependency to a package URL (purl) with name and version.
3. Pass the list to the toolkit, which formats and submits the snapshot.

This is the path to follow when you need to bring ecosystems like Bazel, Buck, Pants, or proprietary internal package managers into the dependency graph.



## Operational Considerations

### Caching

Automatic dependency submission uses the [GitHub Actions Cache](https://github.com/marketplace/actions/cache) to speed up package resolution between runs. For self-hosted runners where you manage your own cache infrastructure, you can disable the built-in caching by setting the environment variable `GH_DEPENDENCY_SUBMISSION_SKIP_CACHE` to `true`.

### API Rate Limits and Payload Size

The Dependency Submission API is subject to standard GitHub API rate limits. For repositories with extremely large dependency trees, consider batching submissions across multiple manifests rather than submitting a single massive snapshot.

### When to Use Which Approach

| Scenario | Recommended Approach |
|---|---|
| Supported ecosystem with simple build | Automatic dependency submission — zero configuration |
| Supported ecosystem needing customization | Pre-built action in a workflow you control |
| Unsupported ecosystem or internal build system | Custom action built with the Dependency Submission Toolkit |
| Existing SBOM pipeline | SBOM submission via SPDX or Anchore action |

## Further Reading

- [Using the Dependency Submission API](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/using-the-dependency-submission-api)
- [Configuring automatic dependency submission](https://docs.github.com/en/enterprise-cloud@latest/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configuring-automatic-dependency-submission-for-your-repository)
- [Dependency Submission Toolkit (GitHub)](https://github.com/github/dependency-submission-toolkit)
- [REST API endpoints for dependency submission](https://docs.github.com/en/enterprise-cloud@latest/rest/dependency-graph/dependency-submission)
- [Dependency graph supported package ecosystems](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/dependency-graph-supported-package-ecosystems)
