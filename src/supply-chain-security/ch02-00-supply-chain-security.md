# Supply Chain Security

Your application is only as secure as its weakest dependency. Building on your existing knowledge of Dependabot, this module covers advanced supply chain security topics including dependency visibility, build provenance, and automated remediation at scale.

Your instructor will guide you through the following topics:

- [**Dependency Graph**](ch02-01-dependency-graph.md) - How GitHub maps your dependency tree and why visibility is the foundation of supply chain security.
- [**Dependency Submission**](ch02-02-dependency-submission.md) - Extending the dependency graph with build-time and runtime dependencies that aren't captured from manifest files alone.
- [**SBOMs, Attestation, and SLSA**](ch02-03-sboms-attestation-slsa.md) - Generating software bills of materials, signing build artifacts with attestations, and achieving SLSA supply chain integrity levels.
- [**Lab: SBOM, Attestation, and SLSA L3**](ch02-04-lab-sbom-attestation.md) - Hands-on: generating an SBOM, attesting a build artifact, and refactoring to a reusable workflow for SLSA Build L3.
- [**Dependabot Alerts**](ch02-05-dependabot-alerts.md) - Advanced configuration, grouping, and triaging alerts at scale.
- [**Dependabot Security and Version Updates**](ch02-06-dependabot-security-version-updates.md) - Fine-tuning update strategies and managing automated pull requests across an organization.
- [**Dependency Review**](ch02-07-dependency-review.md) - Catching vulnerable dependency changes in pull requests before they reach the default branch.
