# Push Protection

> [!Important]
> **📖 Background Reading - Not Part of the Course**
>
> This page covers assumed knowledge and is provided as a reference for self-study. It is up to the instructor's discretion whether this is covered during session. If you are already familiar with secret scanning concepts, feel free to skip ahead to the next section.

The previous chapter established that a committed secret must be treated as compromised the moment it reaches the remote. Secret scanning detects these leaks and alerts you — but detection is inherently reactive. By the time an alert fires, the secret is already in Git history, potentially cloned, and possibly in the hands of an attacker. Push protection shifts the response left by **blocking the push before the secret ever lands in the repository**.

## Why Prevention Beats Detection

Consider the remediation workflow when a secret is detected after the fact:

1. Rotate the credential.
2. Audit access logs to determine whether the credential was exploited.
3. Update every system that consumed the credential.
4. Purge the secret from Git history (which may require force-pushes, re-signing, and coordination across forks).
5. Notify affected teams and, in regulated environments, potentially file an incident report.

This process can take hours to days. Push protection collapses the entire workflow into a single moment: the developer sees a block, removes the secret, and pushes clean code. No rotation, no audit, no incident.

## How Push Protection Works

### The Blocking Flow

When a developer pushes commits to a repository with push protection enabled, the following sequence occurs:

1. Pre-receive scan - GitHub scans the contents of every commit in the push for patterns matching known secret types (partner patterns, non-provider patterns, and any custom patterns configured with push protection enabled).
2. Match found - If one or more secrets are detected, the push is rejected with a detailed error message identifying:
   - The file and line number where the secret was found.
   - The secret type (e.g., "GitHub Personal Access Token", "AWS Access Key ID").
   - A link to the secret scanning alert page with instructions for remediation.
3. Developer action - The developer must remove the secret from the commit(s) before the push will succeed. This typically means rewriting the commit with `git commit --amend` or `git rebase -i` to excise the secret, then pushing again. 

> **Key detail:** Push protection scans the content of commits, not just the diff. If a secret exists anywhere in the pushed commits  even if it was added and then removed in a subsequent commit within the same push the push will be blocked.

### What Gets Scanned

Push protection scans:

- All commits being pushed to any branch (not just the default branch).
- Content pushed via the Git protocol (HTTPS and SSH).
- Content uploaded through the GitHub web UI (e.g., editing a file or uploading via the browser).
- Commits pushed to any repository where push protection is enabled, regardless of branch protection rules.

Push protection does not scan:

- Commits that already exist in the repository (these are covered by the standard secret scanning historical scan).
- Content in Git LFS objects (large file storage pointers are scanned, but the referenced blobs are not).

### Supported Patterns

Push protection supports a subset of secret scanning patterns — specifically, those with sufficiently low false positive rates to justify blocking a push. Blocking on a high-false-positive pattern would create developer friction without meaningful security benefit.

The supported pattern set includes:

- All partner patterns - These have the highest accuracy because they are co-developed with service providers.
- Select non-provider patterns - High-confidence generic patterns such as private keys and HTTP basic auth credentials.
- Custom patterns  Organization and repository-level custom patterns can be individually opted into push protection when they are created or edited. This gives you control over which of your custom detections are enforced as blocking rules.

The full list of patterns supported for push protection is documented in [Supported secret scanning patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/supported-secret-scanning-patterns#supported-secrets).


## Bypass Mechanisms

Blocking every push that contains a pattern match is the strictest posture, but real-world development sometimes produces legitimate reasons to push content that looks like a secret. Push protection provides controlled bypass mechanisms to handle these cases without disabling protection entirely.

### Developer Self-Bypass

When a push is blocked, the developer can choose to bypass the block by providing a reason.

When a bypass is used:

- The push is allowed through.
- A secret scanning alert is created on the repository, tagged with the bypass reason and the identity of the developer who bypassed. The state of the alert may be open or closed based on the bypass reason.
- The bypass event is recorded in the audit log, providing full traceability for security and compliance teams.

> **Important:** Self-bypass should be treated as an exception, not a routine workflow. A high rate of bypasses is a signal that patterns may need tuning or that developers need additional training on secret management.

### Delegated Bypass (Organization-Level Control)

For organizations that require tighter control over who can push secrets, GitHub supports delegated bypass. When enabled, individual developers cannot self-bypass; instead, they must request a bypass from a designated reviewer or team.

The workflow:

1. The push is blocked.
2. The developer submits a bypass request through the GitHub UI, selecting a reason and providing context.
3. A designated bypass reviewer (configured at the organization level) receives the request.
4. The reviewer approves or denies the request.
5. If approved, the developer can re-push and the push will be allowed through.

Delegated bypass is configured in Organization Settings → Code security → Global settings → Push protection. You specify which users or teams are authorized to review bypass requests.

This model is particularly valuable in regulated environments where a four-eyes principle is required for security exceptions.


## Metrics and Monitoring

### Security Overview

The organization-level Security Overview dashboard provides aggregated push protection metrics.

These metrics help security teams assess the effectiveness of push protection and identify areas where developer education or pattern tuning is needed.

### Audit Log

Every push protection event  blocks, bypasses, and delegated bypass reviews is recorded in the organization and enterprise audit log. Audit log entries include:

- The repository, branch, and commit SHA.
- The secret type detected.
- Whether the push was blocked or bypassed.
- The bypass reason (if applicable).
- The identity of the developer and (for delegated bypass) the reviewer.

Audit logs can be streamed to external SIEM systems for integration with your broader security monitoring infrastructure.

## Operational Considerations

### Rollout Strategy

Enabling push protection across a large organization can initially cause developer friction if teams are not prepared. A phased rollout is recommended:

1. Communicate - Notify development teams before enabling push protection. Explain what it does, why it matters, and how to handle blocks.
2. Pilot - Enable push protection on a small set of high-risk repositories first. Monitor bypass rates and developer feedback.
3. Expand - Gradually roll out to additional repositories or enable organization-wide. Use security configurations to target specific repository groups.
4. Enforce - Once teams are comfortable, enable enterprise-level policy enforcement to prevent opt-out.

### Handling False Positives

If a custom pattern is generating too many false-positive blocks, you can:

- Refine the pattern - Tighten the regular expression or add exclusion conditions.
- Remove push protection from the pattern - Keep the pattern active for alerting but disable its push protection enforcement. This converts it from a blocking rule to a detection-only rule.

For built-in partner and non-provider patterns, false positive rates are already very low. If you encounter a false positive on a built-in pattern, report it through GitHub Support.

## Further Reading

- [About push protection](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/about-push-protection)
- [Working with push protection](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/working-with-push-protection)
- [Push protection for users](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/working-with-push-protection/push-protection-for-users)
- [Delegated bypass for push protection](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/using-advanced-secret-scanning-and-push-protection-features/delegated-bypass-for-push-protection)
- [Supported secret scanning patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/supported-secret-scanning-patterns)
- [Security overview dashboard](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-overview/about-security-overview)
