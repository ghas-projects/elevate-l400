# Lab: CodeQL Models as Data - Community Packs

This hands-on lab walks you through adding the [CodeQL Community Packs](https://github.com/GitHubSecurityLab/CodeQL-Community-Packs) at the **organization level** so that every repository using CodeQL default setup automatically benefits from additional queries and models maintained by the GitHub Security Lab community.

The CodeQL Community Packs are a collection of community-driven CodeQL query, library, and extension packs that supplement the built-in query suites. They include additional security queries, model extensions for popular frameworks, and audit-oriented query suites - all published to the GitHub Container Registry under the `githubsecuritylab` scope.

---

## Part 1 - Explore the Community Packs

**Goal:** Understand what the CodeQL Community Packs provide before adding them to your organization.

### 1.1 Browse the repository

1. Navigate to [GitHubSecurityLab/CodeQL-Community-Packs](https://github.com/GitHubSecurityLab/CodeQL-Community-Packs).
2. Review the repository structure. The packs are organized by language:

```
CodeQL-Community-Packs/
├── configs/           # Ready-to-use configuration files
│   ├── default.yml    # Default community pack configuration
│   ├── audit.yml      # Security audit configuration
│   ├── quality.yml    # Code quality configuration
│   └── synthetics.yml # Synthetic / vulnerable app configuration
├── cpp/               # C/C++ packs
├── csharp/            # C# packs
├── go/                # Go packs
├── java/              # Java packs
├── javascript/        # JavaScript/TypeScript packs
├── python/            # Python packs
└── ruby/              # Ruby packs
```

### 1.2 Understand the pack types

Each language directory contains up to three types of packs:

| Pack Type | Naming Convention | Purpose |
|---|---|---|
| **Query pack** | `githubsecuritylab/codeql-<lang>-queries` | Additional security queries beyond the built-in suites |
| **Extension pack** | `githubsecuritylab/codeql-<lang>-extensions` | Model extensions (sources, sinks, summaries) for popular libraries |
| **Library sources pack** | `githubsecuritylab/codeql-<lang>-library-sources` | Models for library code generated from query analysis |

The query packs add new detection rules. The extension packs and library sources packs are model packs - they extend CodeQL's existing queries with additional framework knowledge, exactly as described in the [Models as Data](ch04-04-codeql-models-as-data.md) chapter.

### 1.3 Review the provided configurations

1. Navigate to the [`configs/`](https://github.com/GitHubSecurityLab/CodeQL-Community-Packs/tree/main/configs) directory.
2. Open `default.yml` and review its contents:

```yaml
name: "GitHub Community Pack Default CodeQL Configuration"

packs:
  # C/C++
  - githubsecuritylab/codeql-cpp-queries
  # C#
  - githubsecuritylab/codeql-csharp-queries
  # Go
  - githubsecuritylab/codeql-go-queries
  # Java
  - githubsecuritylab/codeql-java-queries
  # JavaScript / Typescript
  - githubsecuritylab/codeql-javascript-queries
  # Python
  - githubsecuritylab/codeql-python-queries
  # Ruby
  - githubsecuritylab/codeql-ruby-queries

  # Data extensions via Community Packs for libraries
  - githubsecuritylab/codeql-csharp-library-sources
  - githubsecuritylab/codeql-java-library-sources

  # Data extensions via Community Packs
  - githubsecuritylab/codeql-csharp-extensions
  - githubsecuritylab/codeql-java-extensions
```

Notice that it includes both query packs and model extension packs. The query packs bring new detection rules, while the extension packs improve taint tracking through popular frameworks. Only Java and C# currently have extension and library source packs.

---

## Part 2 - Add Community Packs at the Organization Level

**Goal:** Configure your organization's code security settings so that all repositories using CodeQL default setup automatically include the community packs.

### 2.1 Review existing alerts

1. Navigate to the `mad-example-dotnet` repository and open Security > Code scanning alerts. Note the current alert count (expected: 8).
2. Navigate to the `mad-example-java` repository and open Security > Code scanning alerts. Note the current alert count (expected: 4).
3. Consider whether these counts seem accurate. Are there vulnerabilities that CodeQL might be missing? Why might that be the case?

> **Hint:** The default CodeQL query suites only model well-known frameworks. If the application uses a less common or internal library that CodeQL doesn't have models for, taint flow through that library will be invisible and real vulnerabilities will go unreported. This is exactly the gap that extension packs and library source packs are designed to close.

### 2.2 Navigate to code security settings

1. Navigate to your lab organization settings: `https://github.com/orgs/ghas-labs-2026-03-16-<handle>/settings`.
2. In the left sidebar, under **Advanced Security**, click **Global Settings**.
3. Scroll to the **Expand CodeQL analysis** section, click **Configure**
4. Under **Model packs**, add the community extension packs and library source packs. Add one pack per line, following the <scope>/<pack>@x.x.x format. 
5. Click **Save configuration**.

<details>
<summary>Solution: Model packs to add</summary>

```
githubsecuritylab/codeql-java-extensions@0.2.1
githubsecuritylab/codeql-csharp-extensions@0.2.1
githubsecuritylab/codeql-java-library-sources@0.2.1
githubsecuritylab/codeql-csharp-library-sources@0.2.1
```

</details>


> **Note:** You do not need to add every language pack. Only add packs for languages that are actually used in your organization's repositories. CodeQL will match the appropriate packs to each repository based on the detected language.


---

## Part 3 - Verify the Configuration

**Goal:** Confirm that the community packs are being applied to repositories and review any new findings.

### 3.1 Trigger a new analysis

1. Navigate to the `mad-example-dotnet` repository.
2. Go to **Actions** and find the **CodeQL** workflow.
3. Click **Run workflow** to trigger a fresh analysis, or push a commit to trigger it automatically.
4. Repeat for the `mad-example-java` repository.

### 3.2 Inspect the workflow run

1. Open the most recent CodeQL workflow run.
2. Expand the **Initialize CodeQL** step.
3. In the logs, look for lines showing the community packs being used.

This confirms that the organization-level configuration is being picked up correctly. Alternatively, the extensions used in the latest CodeQL run can be viewed from the CodeQL status page 

### 3.3 Review new alerts

1. Navigate to Security > Code scanning alerts both repositories.
2. Look for any new alerts that were not present before.
3. Compare the alert count before and after. The community packs may surface vulnerabilities that the default suite missed especially in areas where the extension packs improve model coverage.

> **Checkpoint:** You have verified that the community packs are being downloaded and executed as part of CodeQL analysis, and you can see any additional findings they produce.

---


## Summary

| What You Did | Why It Matters |
|---|---|
| Explored the CodeQL Community Packs repository | Understanding what additional detection capability is available |
| Added query packs at the organization level | Every repo using default setup now runs additional security queries |
| Added model extension packs at the organization level | CodeQL's taint tracking now covers more frameworks out of the box |
| Verified the configuration in a workflow run | Confirmed packs are downloaded and producing results |

By adding the community packs at the organization level, you have extended CodeQL's coverage across your entire organization without requiring any changes to individual repositories. This is the recommended approach for organizations that want broad, consistent security coverage with minimal per-repository maintenance.

---

## Discussion

1. Before adding the community packs, the `mad-example-dotnet` and `mad-example-java` repositories reported 8 and 4 alerts respectively. After adding extension and library source packs, did new alerts appear? What does this tell you about the default model coverage for the frameworks these applications use?

2. What are the risks of adding community-maintained packs at the organization level? How would you evaluate whether to use community pack before rolling it out?

3. Extension packs improve taint tracking by modeling additional sources, sinks, and summaries. How would you decide whether a missing detection is better addressed by a community extension pack or by writing your own Models as Data YAML?

---

## Further Reading

- [CodeQL Community Packs repository](https://github.com/GitHubSecurityLab/CodeQL-Community-Packs)
- [Announcing CodeQL Community Packs (GitHub Blog)](https://github.blog/security/vulnerability-research/announcing-codeql-community-packs/)
- [Configuring default setup for code scanning](https://docs.github.com/en/enterprise-cloud@latest/code-security/code-scanning/enabling-code-scanning/configuring-default-setup-for-code-scanning)
- [Managing code security configurations for your organization](https://docs.github.com/en/enterprise-cloud@latest/code-security/securing-your-organization/managing-the-security-of-your-organization/managing-your-code-security-configurations)
