# Dependency Graph

The dependency graph is the foundational data layer for all of GitHub's supply chain security features. Every Dependabot alert, every dependency review in a pull request, and every exported SBOM ultimately traces back to the dependency graph. Understanding how it works and where its boundaries are is essential for anyone responsible for securing a GitHub environment at scale.

## Know Your Environment

Modern software projects routinely pull in hundreds or thousands of open-source packages. A single direct dependency can introduce a deep tree of transitive dependencies, each of which represents an additional surface area for vulnerabilities, license risk, and supply chain attacks. You cannot secure what you cannot see.

The dependency graph solves this visibility problem. It parses the manifest and lock files committed to your repository (e.g. `package-lock.json`, `pom.xml` etc.), and constructs a complete inventory of every package your project depends on, including:

- Direct dependencies declared in your manifest files.
- Transitive (indirect) dependencies resolved through lock files, where [supported by the ecosystem.](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/supply-chain-security/dependency-graph-supported-package-ecosystems#supported-package-ecosystems) 
- Version and license metadata for each dependency.
- Vulnerability status cross-referenced against the [GitHub Advisory Database](https://github.com/advisories) if dependabot is enabled. 

For public repositories, the graph also tracks dependents, other repositories and packages that depend on yours, enabling you to understand your downstream blast radius.

## How the Dependency Graph Is Built

The graph is constructed from two data sources:

1. Manifest and lock file analysis - On every push to the repository, GitHub parses supported manifest and lock files on the default branch. The parsing is read-only; it never executes package manager commands or runs arbitrary code.

2. Dependency Submission API - For ecosystems or build systems that aren't natively supported (e.g., vendored C/C++ libraries, internal package registries, or monorepo build tools like Bazel), you can submit dependency snapshots programmatically via the [Dependency Submission API](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/using-the-dependency-submission-api). Dependencies submitted this way are tagged with the detector that produced them and the timestamp of submission, giving you full provenance.

> **Key detail:** The graph distinguishes between dependencies discovered from manifest analysis and those submitted via the API. When triaging alerts or reviewing SBOMs, this distinction tells you whether a dependency was declared in source or injected at build time.

## Enabling the Dependency Graph

The dependency graph can be enabled at the repository, organization, or enterprise level. Each scope carries different trade-offs around control, rollout speed, and policy enforcement. These are covered in the sections that follow.


## What the Dependency Graph Powers

The dependency graph is not a standalone feature, it is the data backbone for several downstream capabilities:

| Feature | How It Uses the Dependency Graph |
|---|---|
| **Dependabot Alerts** | Matches dependencies in the graph against known vulnerabilities in the GitHub Advisory Database. |
| **Dependabot Security Updates** | Uses the graph to determine which dependency version to upgrade to and generates a pull request. |
| **Dependency Review** | Compares the dependency graph between the base and head of a pull request to surface added/removed/changed dependencies and their vulnerability status. |
| **SBOM Export** | Generates a SPDX-format Software Bill of Materials from the graph for audit, compliance, and regulatory purposes. |
| **Organization Dependency Insights** | Aggregates dependency data across all repositories in an organization for a centralized view. |
| **Security Overview** | Rolls up dependency alert data at the organization and enterprise level for security managers. |

Understanding this relationship is critical: **if a dependency is not in the graph, none of these features will detect it.** This is why the Dependency Submission API exists - it closes the gap for dependencies that static file parsing cannot discover.

## Operational Considerations

### Data Freshness

The graph updates on every push to the default branch. For non-default branches, dependency data is computed on-demand when a pull request is opened (for dependency review). There is no periodic polling; the graph is event-driven.

### Permissions and Access

- Enabling the graph requires admin access to the repository, or organization owner / security manager role for org-wide enablement.
- The dependency graph itself is visible to anyone with read access to the repository.

### Scale and Limits

Repositories with very large numbers of manifests or extremely deep dependency trees may experience longer initial parse times. In practice, even large monorepos with thousands of dependencies typically complete parsing within a few minutes.

## Further Reading

- [About the dependency graph](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph)
- [Enabling the dependency graph](https://docs.github.com/en/enterprise-cloud@latest/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/enabling-the-dependency-graph)
- [Exploring the dependencies of a repository](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/exploring-the-dependencies-of-a-repository)
- [Using the Dependency Submission API](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/using-the-dependency-submission-api)
- [Dependency graph supported package ecosystems](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/dependency-graph-supported-package-ecosystems)
- [Exporting a software bill of materials (SBOM)](https://docs.github.com/en/enterprise-cloud@latest/code-security/how-tos/secure-your-supply-chain/establish-provenance-and-integrity/exporting-a-software-bill-of-materials-for-your-repository)
