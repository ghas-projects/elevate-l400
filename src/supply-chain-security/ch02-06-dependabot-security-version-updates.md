# Dependabot Security and Version Updates

> [!Important]
> **📖 Background Reading - Not Part of the Course**
>
> This page covers assumed knowledge and is provided as a reference for self-study. It is up to the instructor's discretion whether this is covered during session. If you are already familiar with supply chain security concepts, feel free to skip ahead to the next section.

The previous chapter covered how Dependabot alerts surface vulnerable dependencies in your repository. Alerts are the detection mechanism - they tell you something is wrong. This chapter covers the remediation mechanisms: Dependabot security updates and Dependabot version updates. Both automatically open pull requests to update dependencies, but they serve fundamentally different purposes, operate on different triggers, and require different configuration approaches.

Understanding the distinction is critical for organizations designing their supply chain security strategy at scale.

## Two Update Mechanisms

| | Security Updates | Version Updates |
|---|---|---|
| **Purpose** | Remediate known vulnerabilities | Keep dependencies current |
| **Trigger** | A Dependabot alert is created for a vulnerable dependency | A scheduled check finds a newer version available |
| **Scope** | Only dependencies with active alerts | All dependencies declared in configured manifests |
| **Configuration** | Enabled via repository/org settings (no config file required) | Requires a `dependabot.yml` configuration file |
| **Pull request target** | Minimum patched version that resolves the vulnerability | Latest available version (configurable) |
| **Prerequisite** | Dependabot alerts must be enabled | Dependency graph must be enabled |

Both features generate pull requests against your repository. The pull requests include changelogs, release notes, commit diffs, and compatibility scores where available giving reviewers the context they need to merge with confidence.

## Dependabot Security Updates

### How They Work

When Dependabot creates an alert for a vulnerable dependency and the advisory includes a patched version, Dependabot security updates will automatically open a pull request that updates the dependency to the **minimum non-vulnerable version**. The update targets the smallest version bump that resolves the vulnerability, minimizing the risk of introducing breaking changes.

The process:

1. A Dependabot alert fires (vulnerability detected in the dependency graph).
2. Dependabot checks whether a patched version exists in the advisory metadata.
3. If a patch is available, Dependabot resolves the dependency update against your manifest and lock files, ensuring compatibility.
4. Dependabot opens a pull request with the version bump, including:
   - A description of the advisory being resolved.
   - Links to the advisory, changelog, and release notes.
   - A compatibility score based on CI pass rates from other repositories that have merged the same update.
5. If the repository has CI configured, the pull request runs your test suite automatically.

### Enabling Security Updates

Dependabot Security Updates can be enables at the Repository, Organization or Enterprise level. They require GitHub actions. 

### Grouped Security Updates

By default, Dependabot opens one pull request per vulnerable dependency. For repositories with many alerts, this can create a flood of individual PRs. Grouped security updates consolidate multiple security updates into a single pull request, reducing review overhead.

Grouped security updates are configured through the `dependabot.yml` file at the repository level by adding a `groups` key within a security update configuration:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      security-patches:
        applies-to: security-updates
        patterns:
          - "*"
```

This groups all npm security updates into a single pull request. You can also create multiple groups based on package name patterns or dependency types to control granularity.

### Security Update Limits and Behavior

- Dependabot will open a maximum of 10 security update pull requests at a time per repository. Once one is merged or closed, the next queued update is opened.
- If the vulnerable dependency cannot be updated without disrupting the dependency graph (e.g., a transitive dependency conflict), Dependabot reports an error on the alert instead of opening a PR. For npm, Dependabot can update parent dependencies or remove unused sub-dependencies to resolve transitive vulnerabilities; for other ecosystems, indirect dependencies that require a parent update cannot be resolved automatically.
- When a `.github/dependabot.yml` file exists, security update PRs inherit applicable settings such as `ignore`, `reviewers`, `assignees`, and `labels`. No config file is required - security updates work out of the box once enabled. See [Customizing pull requests for Dependabot security updates](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/dependabot-security-updates/customizing-dependabot-security-prs).

## Dependabot Version Updates

### How They Work

Dependabtot Version Updates check for newer versions of your dependencies on a configurable schedule and open pull requests to keep your dependencies current regardless of whether a vulnerability exists.

The rationale: staying current reduces the distance to the next security patch. A project on version 2.1.0 of a library can update to 2.1.5 (the patched version) with a minor bump. A project stuck on version 1.3.0 may require a major version migration to reach the patch, which is far more costly and risky.

### The `dependabot.yml` Configuration File

Version updates are configured exclusively through a file at `.github/dependabot.yml` in your repository. This file defines which package ecosystems to monitor, how often to check for updates, and how to customize the resulting pull requests.

#### Minimal Configuration

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

This checks for new versions of all npm dependencies in the root `package.json` every week and opens PRs for any that have updates available.

#### Complete Configuration Example

```yaml
version: 2
registries:
  npm-private:
    type: npm-registry
    url: https://npm.pkg.github.com
    token: ${{ secrets.GHPR_TOKEN }}

updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "America/New_York"
    open-pull-requests-limit: 10
    reviewers:
      - "my-org/frontend-team"
    assignees:
      - "octocat"
    labels:
      - "dependencies"
      - "npm"
    commit-message:
      prefix: "deps"
      include: "scope"
    ignore:
      - dependency-name: "express"
        versions: ["5.x"]
    groups:
      minor-and-patch:
        applies-to: version-updates
        update-types:
          - "minor"
          - "patch"
      major:
        applies-to: version-updates
        update-types:
          - "major"
    registries:
      - npm-private

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      actions:
        patterns:
          - "*"

  - package-ecosystem: "docker"
    directory: "/deploy"
    schedule:
      interval: "monthly"
```

### Key Configuration Options

The `dependabot.yml` file supports a wide range of options for controlling update behavior, including `package-ecosystem`, `directory`, `schedule`, `open-pull-requests-limit`, `ignore`, `groups`, `target-branch`, `reviewers`, `assignees`, `labels`, and many more. For a complete reference of all available options and their usage, see the [Dependabot options reference](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference).

> **Tip:** The `github-actions` ecosystem deserves special attention. It monitors the versions of actions referenced in your workflow files. Keeping actions up to date is a supply chain security concern in its own right - a compromised or outdated action can inject malicious code into your CI pipeline.

### Private Registries

Many organizations host packages in private registries (Artifactory, Nexus, GitHub Packages, AWS CodeArtifact, etc.). Dependabot supports private registry authentication through the `registries` configuration in `dependabot.yml` for both security updates and version updates. For a complete list of supported registry types and their configuration options, see the [Dependabot options reference](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#registries).

Registry credentials are stored as Dependabot secrets (separate from Actions secrets), configured at the repository, organization, or enterprise level. Dependabot secrets are only exposed to Dependabot processes, not to Actions workflows.

```yaml
registries:
  artifactory:
    type: maven-repository
    url: https://artifactory.example.com/maven-releases
    username: ${{ secrets.ARTIFACTORY_USER }}
    password: ${{ secrets.ARTIFACTORY_PASSWORD }}

updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
    registries:
      - artifactory
```

> **Key detail:** Each `updates` entry must explicitly reference the registries it needs via the `registries` key. Registries defined at the top level are not automatically used by all entries.

## Grouped Updates In Depth

Grouped updates are available for both Security and Version Updates. 

### Group Configuration

Groups are defined per ecosystem entry in `dependabot.yml` and support filters such as `applies-to`, `patterns`, `exclude-patterns`, `dependency-type`, and `update-types`. For a full reference of group configuration options, see the [Dependabot options reference](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#groups).


## Managing Dependabot at Scale

### Organization-Level Configuration

For organizations managing hundreds of repositories, configuring `dependabot.yml` in every repository is impractical. Several approaches address this:

#### Security Configurations

Security configurations (available on GitHub Enterprise Cloud) let you bundle Dependabot alerts and security updates as part of a named configuration applied to repositories matching specific criteria. This does not manage `dependabot.yml` for version updates, but it handles the security update side.

#### Programmatic Configuration

For full control, use GitHub's REST API or a configuration-as-code approach to generate and commit `dependabot.yml` files across repositories. This is common in large enterprises where dependency update policies are centrally defined and enforced.

### Monitoring Update Health

Track the health of your Dependabot rollout through:

- Security Overview - Shows which repositories have Dependabot enabled and alert/update remediation trends.
- Dependabot activity log - Available per-repository under the Dependabot tab, showing all update activity, failures, and PR status.
-Webhook events  `dependabot_alert` and `pull_request` events can feed external dashboards for organization-wide visibility.

### Common Failure Modes

| Failure | Cause | Resolution |
|---|---|---|
| PR creation fails | Dependency conflict or lock file resolution error | Review the Dependabot logs; may require manual update |
| Private registry auth fails | Missing or expired Dependabot secret | Rotate the secret in repository/org Dependabot settings |
| PR sits unreviewed | No reviewers assigned, low team bandwidth | Configure `reviewers` in `dependabot.yml`; implement auto-merge for low-risk updates |
| Too many open PRs | High dependency count, low merge rate | Use `groups` and `open-pull-requests-limit` to control volume |
| Updates break CI | Incompatible version bump | Add the dependency to `ignore` temporarily; investigate and adapt |


## Further Reading

- [About Dependabot security updates](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates)
- [Configuring Dependabot security updates](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/dependabot-security-updates/configuring-dependabot-security-updates)
- [About Dependabot version updates](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/dependabot-version-updates/about-dependabot-version-updates)
- [Configuration options for the dependabot.yml file](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file)
- [Managing pull requests for dependency updates](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/working-with-dependabot/managing-pull-requests-for-dependency-updates)
- [Automating Dependabot with GitHub Actions](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/working-with-dependabot/automating-dependabot-with-github-actions)
- [Configuring access to private registries for Dependabot](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/working-with-dependabot/configuring-access-to-private-registries-for-dependabot)
- [Optimizing Dependabot pull requests with grouped updates](https://docs.github.com/en/enterprise-cloud@latest/code-security/dependabot/dependabot-version-updates/optimizing-pr-creation-version-updates)
