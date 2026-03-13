# Dependency Review

> [!Important]
> **📖 Background Reading - Not Part of the Course**
>
> This page covers assumed knowledge and is provided as a reference for self-study. It is up to the instructor's discretion whether this is covered during session. If you are already familiar with supply chain security concepts, feel free to skip ahead to the next section.

Every supply chain security feature covered so far has been reactive. The dependency graph inventories what is already in your codebase. Dependabot alerts notify you about vulnerabilities in dependencies you have already adopted. Security updates remediate vulnerabilities that already exist on your default branch. Version updates keep dependencies current after they have fallen behind.

All of these operate on the default branch - they respond to problems that have already been merged into your codebase. The damage is done; you are remediating.

Dependency review flips the model. It is the proactive layer in GitHub's supply chain security stack. It evaluates dependency changes before they are merged, at the pull request stage. Instead of cleaning up after a vulnerable dependency enters your codebase, dependency review prevents it from entering in the first place.

## How Dependency Review Works

When a pull request is opened, dependency review compares the dependency graph of the base branch against the head branch (the PR branch). It identifies:

- Added dependencies - packages that are new in the PR.
- Removed dependencies - packages that the PR eliminates.
- Changed dependencies - packages whose version has changed (upgraded or downgraded).

For each added or changed dependency, dependency review cross-references the version against the GitHub Advisory Database. If the new or updated dependency has known vulnerabilities, the review surfaces them directly in the pull request,

This is the same advisory database used by Dependabot alerts, but the timing is fundamentally different: alerts tell you about vulnerabilities after the fact; dependency review tells you about them before the fact.

## The Dependency Review Action

To enforce dependency review as a gate, you need the [`dependency-review-action`](https://github.com/actions/dependency-review-action).

This GitHub Action runs as a CI check on pull requests. It evaluates the dependency changes introduced by the PR and based on the policy set can fail the check if the changes violate your configured policy. When paired with branch protection rules or rulesets that require this check to pass before merging, it becomes a hard gate that prevents vulnerable dependencies from entering your codebase.

### Basic Setup

```yaml
name: Dependency Review

on: pull_request

permissions:
  contents: read

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Dependency Review
        uses: actions/dependency-review-action@v4
```

With no additional configuration, this will fail the check if the PR introduces any dependency with a known vulnerability of **any severity**.

### Configuration Options

The action supports extensive configuration to tune the policy to your organization's risk tolerance.

For a complete reference of all inputs and configuration options, see the [dependency-review-action documentation](https://github.com/actions/dependency-review-action?tab=readme-ov-file#configuration-options).

### Action Output

When the action fails, it produces a detailed summary directly in the pull request check output:

- Which dependencies triggered the failure.
- The advisory identifiers and severity for each.
- The license that violated policy (if the failure was license-related).

This gives the PR author immediate, actionable feedback - they know exactly what to fix without opening a separate tool.

## Enforcing Dependency Review with Rulesets

Configuring the `dependency-review-action` as a workflow in each repository is one approach, but it relies on every repository having the workflow file and appropriate branch protection. At scale, this is fragile - a repository owner could remove the workflow or modify branch protection to bypass the check.

Repository rulesets solve this by providing centralized, tamper-resistant enforcement at the organization and enterprise level. Rulesets can require specific status checks to pass before a pull request is merged, and they can be configured so that repository administrators cannot override them.

### How Rulesets Enforce Dependency Review

The enforcement model works in two layers:

1. The workflow - The `dependency-review-action` runs as a CI check on pull requests, producing a pass/fail status.
2. The ruleset - An organization or enterprise-level ruleset requires that status check to pass before merging, and prevents repository admins from bypassing or removing the requirement.

Together, these ensure that every pull request across your organization is evaluated against your dependency policy, with no opt-out.

### Creating Rulesets for Dependency Review

Rulesets can be created at the oranization or enterprise level to require the `dependency-review-action` status check to pass before merging. Key steps include targeting the default branch, adding the required status check name (matching the job name in your workflow), choosing enforcement mode (Active or Evaluate), and configuring bypass permissions.

For detailed instructions, see:
- [Managing rulesets for repositories in your organization](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-organization-settings/managing-rulesets/managing-rulesets-for-repositories-in-your-organization)
- [Managing rulesets for repositories in your enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/managing-code-security/securing-your-enterprise/managing-rulesets-for-repositories-in-your-enterprise)

### Ruleset Evaluation Mode

Before enforcing a ruleset broadly, use Evaluate mode. In this mode, the ruleset is active, it evaluates every PR, but it does not block merges. Instead, it produces insights showing which PRs would have been blocked. This lets you:

- Assess the impact before turning enforcement on.
- Identify repositories that need to add the `dependency-review-action` workflow.
- Tune the action's configuration (severity threshold, license allowlist, etc.) based on real data.

Once you are confident in the policy, switch the ruleset from Evaluate to Active to begin enforcement.


## Operational Considerations

### Performance and Scale

Dependency review runs on-demand for each pull request. For repositories with large lock files (thousands of dependencies), the diff computation takes seconds, not minutes. The action itself is lightweight, it calls GitHub's dependency review API and evaluates the results against your policy.

### What Dependency Review Does Not Catch

Dependency review evaluates direct changes to dependency manifests and lock files in a pull request. It does not catch:

- Vulnerabilities disclosed after the PR is merged - This is what Dependabot alerts cover.
- Dependencies introduced outside of pull requests - Direct pushes to the default branch bypass PR checks entirely. Use branch protection or rulesets to require PRs for all changes.
- Build-time dependencies not in the graph - If a dependency is resolved at build time and not submitted to the dependency graph, dependency review cannot evaluate it. This is why the dependency submission chapters matter.

### License Compliance at Scale

 Many organizations discover license compliance issues late, during legal review before an acquisition, during open-source audit, or when a downstream consumer raises a concern. By enforcing license policy at the PR gate, you prevent these issues from accumulating.

A recommended approach:

1. Start in audit mode - Run the action with `comment-summary-in-pr: always` but without failing on licenses. Collect data on the licenses in use across your organization.
2. Build a license allowlist - Work with your legal team to define the allowed licenses.
3. Enforce via the action*- Switch to `allow-licenses` with your approved list.
4. Enforce via rulesets -  Create an organization or enterprise ruleset requiring the check to pass.

### Relationship to Security Configurations

Dependency Review enablement is not an option in Security Configurations at Organization or Enterprise level. 

## Further Reading

- [About dependency review](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review)
- [Configuring dependency review](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/configuring-dependency-review)
- [dependency-review-action](https://github.com/actions/dependency-review-action)
- [About rulesets](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [Managing rulesets for an organization](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-organization-settings/managing-rulesets/managing-rulesets-for-repositories-in-your-organization)
- [Managing rulesets for an enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/managing-code-security/securing-your-enterprise/managing-rulesets-for-repositories-in-your-enterprise)
- [Dependency review configuration options](https://github.com/actions/dependency-review-action?tab=readme-ov-file#configuration-options)
