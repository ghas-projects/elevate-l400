# Secret Scanning

Leaked credentials are one of the most exploited attack vectors in real-world breaches. Building on your existing knowledge of secret scanning, this module covers how GitHub detects secrets across your repositories, prevents them from being pushed in the first place, and how you can extend detection to cover organization-specific credential formats.

Your instructor will guide you through the following topics:

- [**What Is Secret Scanning**](ch03-01-what-is-secret-scanning.md) - How secret scanning detects leaked credentials using partner, non-provider, and custom patterns, the scanning scope across pushes, branches, and full Git history, and how alerts are managed and remediated.
- [**Push Protection**](ch03-02-push-protection.md) - Shifting left by blocking pushes that contain recognized secrets before they reach the repository, eliminating the need for costly post-commit remediation workflows.
- [**Custom Patterns**](ch03-03-custom-patterns.md) - Extending secret scanning to detect organization-specific credentials — internal API tokens, database connection strings, and legacy authentication formats — using custom regex patterns with contextual filters.

