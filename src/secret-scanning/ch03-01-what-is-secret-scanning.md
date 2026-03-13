# What Is Secret Scanning

> [!Important]
> **📖 Background Reading - Not Part of the Course**
>
> This page covers assumed knowledge and is provided as a reference for self-study. It is up to the instructor's discretion whether this is covered during session. If you are already familiar with secret scanning concepts, feel free to skip ahead to the next section.


Leaked credentials are consistently among the most exploited attack vectors in real-world breaches. An API key committed to a public repository can be harvested by automated scanners within seconds. An internal service token pushed to a private repository can persist unnoticed for months, granting lateral access long after the commit that introduced it has scrolled out of view. The problem is not hypothetical, GitHub has detected and reported over 100 million secret leaks across the platform.

Secret scanning is GitHub's answer to this class of risk. It continuously monitors your repositories for known secret formats - API keys, access tokens, private keys, connection strings, and other credentials - and alerts you when one is detected. When combined with push protection, it can prevent secrets from entering the repository in the first place.

## Why Secret Scanning Matters

Secrets are different from other security findings in one critical way: **the moment a secret is committed, it must be considered compromised.** Even if the commit is immediately reverted or force-pushed away, Git's history retains the value. Anyone who cloned or forked the repository between the push and the remediation has a copy. Automated credential-harvesting bots constantly scan public repositories and can exfiltrate a secret within minutes of it being pushed.

This means that for secrets, detection speed and prevention are paramount. Secret scanning addresses both:

- Detection - Scans every push, every branch, and the full Git history (including historical commits) for patterns matching known secret types.
- Prevention - Push protection blocks pushes that contain recognized secrets before they reach the remote repository, stopping the leak at the source.

## How Secret Scanning Works

### Pattern Matching

Secret scanning uses a library of patterns,regular expressions and validation logic to identify secrets. These patterns fall into three categories:

| Pattern Type | Description | Example |
|---|---|---|
| **Partner patterns** | Patterns developed in collaboration with service providers (e.g., AWS, Azure, Slack, Stripe). When a match is found, GitHub notifies the provider, who can automatically revoke the credential. | `AKIA[0-9A-Z]{16}` (AWS Access Key ID) |
| **Non-provider patterns** | Patterns for generic secret formats that are not tied to a specific provider (e.g., private keys, generic API tokens, HTTP basic auth in URLs). These generate alerts but do not trigger provider notification. | `-----BEGIN RSA PRIVATE KEY-----` |
| **Custom patterns** | Organization- or repository-level patterns you define to detect secrets specific to your environment (e.g., internal service tokens, proprietary key formats). | Defined by you |

For the full list of supported patterns, see [Supported secret scanning patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/supported-secret-scanning-patterns) (partner patterns) and [Non-provider patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/supported-secret-scanning-patterns#non-provider-patterns).

Partner patterns are the highest-fidelity signal. Because the patterns are co-developed with the service provider, false positive rates are extremely low, and the provider can take immediate remediation action (often automatic revocation) upon notification in public repositories.

### Scanning Scope

Secret scanning operates across multiple dimensions:

- Push scanning - Every push to any branch is scanned in near real-time. Detected secrets generate alerts within seconds.
- Historical scanning - When secret scanning is first enabled on a repository, the full Git history is scanned, including all branches and all commits. This surfaces secrets that were committed before scanning was turned on.
- Issue and pull request scanning - Titles, descriptions, and comments on issues and pull requests are also scanned for secrets.
- Discussions scanning - Titles, descriptions, and comments in GitHub Discussions are scanned.
- Wiki scanning - If the repository has a wiki, its content is scanned as well.
- Secret gists - Secret (non-public) gists are also scanned for leaked credentials.


> **Key detail:** Secret scanning operates on the **content** of commits, not just the diff. This means that even if a secret was introduced in an old commit and subsequently removed, the historical scan will still flag it — because the secret existed in the repository's history and must be treated as compromised.



## Enabling Secret Scanning

Secret scanning can be enabled at the repository, organization, or enterprise level.

## Alert Lifecycle

### Alert States

Secret scanning alerts move through defined states - from open through to closure with a specific reason (e.g., revoked, false positive, used in tests, won't fix). For the full list of states and their meanings, see [Managing alerts from secret scanning](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/managing-alerts-from-secret-scanning).

> **Governance tip:** Ensure developers across your organization have a shared, documented understanding of what each closure reason means and when it should be used. Inconsistent use of dismissal reasons (e.g., closing a genuine secret as "false positive" to clear triage queues) undermines the integrity of your security data, distorts metrics in the security overview, and can mask real risk. Define these expectations before rolling out secret scanning and reinforce them in onboarding and training.

### Delegated Alert Dismissal

Just as with Dependabot alerts, secret scanning supports delegated alert dismissal. When enabled, developers cannot directly close alerts - instead, their dismissal creates a request that must be reviewed and approved by an organization owner or security manager. This prevents premature or inappropriate closure and ensures every dismissal is audited.

Delegated dismissal can be enabled at the repository, organization, or enterprise level through security configurations by enabling `Prevent direct alert dismissal` under secret scanning settings. See [Enabling delegated alert dismissal for secret scanning](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/using-advanced-secret-scanning-and-push-protection-features/enabling-delegated-alert-dismissal-for-secret-scanning).

> **Operational note:** Delegated dismissal can create a bottleneck if only organization owners and security managers can approve requests. Consider creating a [custom organization role](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-peoples-access-to-your-organization-with-roles/about-custom-organization-roles) with permission to review dismissal requests, allowing you to distribute triage across a broader group without granting full security manager access.

### Validity Checks

For partner patterns, GitHub can verify whether a detected secret is still active. The validity status is shown on the alert:

- Active - The token was verified as valid by the partner. Immediate revocation is critical.
- Inactive -The token has been revoked or expired. The alert remains for audit purposes.
- Unknown - GitHub was unable to verify the token's status (e.g., the partner's verification endpoint was unavailable).

Validity checks reduce triage burden by letting you prioritize active credentials over already-revoked ones.

### AI-Powered Generic Secret Detectiong

Partner patterns and non-provider patterns cover structured secrets with recognizable formats. But many real-world credentials are unstructured - hardcoded passwords, database connection strings with inline credentials, or ad-hoc tokens that don't follow a standard pattern. These fall through the cracks of regex-based detection.

This uses a large language model (LLM) to detect unstructured secrets - specifically passwords - in your source code. It analyzes the context around strings to determine whether they are likely credentials, even when they don't match any known pattern.

Key characteristics:

- Separate alert list - AI-detected secrets appear in a dedicated "Generic" list on the secret scanning alerts page, distinct from partner and non-provider alerts. This separation acknowledges the higher false positive rate and encourages more scrutiny during triage.
- Passwords only (currently) - Generic detection currently focuses on passwords in git content. It does not scan non-git content (issues, PRs, wikis) or detect other unstructured secret types.
- Higher false positive rate - Because the detection is probabilistic rather than pattern-based, expect more false positives than with partner patterns. Closing false positives as "False positive" helps improve the model over time.
- Limitations by design - The LLM skips obviously fake or test passwords, low-entropy strings, generated/vendored files, encrypted files, and certain file types (SVG, PNG, CSV, etc.). It also skips test files (paths containing "test", "mock", or "spec") for supported languages.
- No Copilot subscription required - This feature is part of GitHub Secret Protection, not GitHub Copilot. It is available for organization-owned repositories on GitHub Team or GitHub Enterprise Cloud with Secret Protection enabled.

Copilot secret scanning must be enabled separately - it is not turned on by default when secret scanning is enabled. For GitHub Enterprise Cloud, an enterprise policy must allow generic detection before organizations can enable it. See [Responsible detection of generic secrets with Copilot secret scanning](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/copilot-secret-scanning/responsible-ai-generic-secrets).

## Operational Considerations

### Remediation Is More Than Rotation

When a secret scanning alert fires, the response should follow a structured lifecycle. Simply generating a new credential is not enough - the leaked credential remains valid until explicitly revoked, and may already have been exploited.

#### 1. Assess the alert

Review the alert to understand the type of secret, where it was detected, and the potential blast radius. Check the validity status - if the secret is already **Inactive**, the risk is lower but the alert still requires proper closure.

#### 2. Revoke the leaked credential

Immediately invalidate the compromised secret at its source (the provider's console, API, or admin panel). Do not wait until a replacement is ready - an active leaked credential is an open door.

#### 2. Audit usage

Before rotating, check access logs and audit trails at the provider to determine whether the secret was used by an unauthorized party during the exposure window. If unauthorized access is found, escalate to your incident response process.

#### 3. Rotate and store properly

Generate a new credential and store it in a secrets manager (e.g., Azure Key Vault, AWS Secrets Manager, HashiCorp Vault, or GitHub Actions secrets). Never commit secrets directly to source code. The new credential should be provisioned through environment variables, CI/CD secrets, or a secrets manager integration - not hardcoded in configuration files.

#### 4. Update all consumers

Ensure every system, service, and pipeline that uses the credential is updated to the new value. This includes CI/CD workflows, deployed applications, and any shared configurations.

#### 5. Remove from history

If the repository is public, consider using `git filter-repo` or BFG Repo-Cleaner to purge the secret from Git history. Note that this does not guarantee safety - any clone or fork made before the purge retains the secret.

#### 6. Close the alert

Once the credential has been revoked and rotated, close the secret scanning alert with the appropriate reason (e.g., Revoked). This keeps your alert backlog clean and provides an accurate audit trail. Do not close alerts before remediation is complete - open alerts should reflect genuine outstanding risk.


## Further Reading

- [About secret scanning](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/about-secret-scanning)
- [Supported secret scanning patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/supported-secret-scanning-patterns)
- [Secret scanning partner program](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/supported-secret-scanning-patterns#supported-secrets)
- [Managing alerts from secret scanning](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/managing-alerts-from-secret-scanning)
- [Push protection for repositories and organizations](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/about-push-protection)
