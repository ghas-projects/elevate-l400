# Custom Patterns

> [!Important]
> **📖 Background Reading - Not Part of the Course**
>
> This page covers assumed knowledge and is provided as a reference for self-study. It is up to the instructor's discretion whether this is covered during session. If you are already familiar with secret scanning concepts, feel free to skip ahead to the next section.

GitHub's built-in partner and non-provider patterns cover a broad range of well-known secret formats, but every organization has credentials that are unique to its own infrastructure. Internal API tokens with a proprietary prefix, database connection strings for internal services, legacy authentication tokens with a custom encoding scheme - none of these will match a built-in pattern. Custom patterns let you extend secret scanning to detect secrets that only your organization knows about.

## Why Custom Patterns Matter

Built-in patterns are designed for universally recognized secret formats. They cannot cover:

- Internal service tokens - Tokens issued by your own identity platform or API gateway that follow a proprietary format (e.g., `mycompany_live_[a-zA-Z0-9]{32}`).
- Database connection strings - Connection URIs for internal databases that embed credentials in a predictable format.
- Legacy credentials - Older authentication mechanisms that predate modern token standards but are still in use across your estate.
- Regulated data - In some cases, organizations use custom patterns to detect non-credential sensitive data that should not appear in source code, such as internal identifiers or encryption keys with a known structure.

Without custom patterns, these secrets flow through secret scanning undetected. Custom patterns close this gap by letting you define exactly what constitutes a secret in your environment.

## Defining a Custom Pattern

A custom pattern consists of the following components:

| Component | Required | Description |
|---|---|---|
| **Pattern name** | Yes | A human-readable label displayed in alerts (e.g., "Acme Internal API Token"). |
| **Secret format** | Yes | A regular expression that matches the secret itself. This is the core detection logic. |
| **Before secret** | No | A regular expression that must appear immediately before the secret match. Used to reduce false positives by requiring contextual prefixes (e.g., `api_key=` or `Authorization: Bearer `). |
| **After secret** | No | A regular expression that must appear immediately after the secret match. Used to anchor the match and reduce false positives (e.g., requiring a line ending or delimiter). |
| **Additional match requirements** | No | Up to 10 additional regular expressions that must (or must not) match somewhere in the content surrounding the secret. These act as further context filters. |

### Writing Effective Patterns

The quality of your custom pattern directly determines its usefulness. A pattern that is too broad will flood teams with false positives; a pattern that is too narrow will miss real leaks.

Start with real examples - Collect 5–10 actual instances of the credential you want to detect. Identify the common structure — fixed prefixes, character sets, lengths, delimiters, and surrounding context.

Use the secret format field for the core match - The regular expression in the secret format field should match the credential itself — the value that would be dangerous if exposed. For example, if your internal tokens look like `acme_live_AbCdEf1234567890AbCdEf1234567890`, the secret format might be:

```
acme_(live|test)_[A-Za-z0-9]{32}
```

Use before/after fields to add context - If the token always appears after a specific key name or header, use the before-secret field to require that context. This dramatically reduces false positives:

- Before secret: `ACME_API_KEY\s*=\s*`
- After secret: `\s*$` (end of line)

Use additional match requirements to filter further - If you know the secret should only appear in certain file types or always alongside a specific configuration key, add these as additional requirements. You can also use negative requirements ("must NOT match") to exclude known safe patterns like documentation examples.

### Regex Syntax and Limits

Custom patterns use [Hyperscan](https://www.intel.com/content/www/us/en/developer/articles/technical/introduction-to-hyperscan.html)-compatible regular expressions. Most standard regex features are supported, including:

- Character classes (`[A-Za-z0-9]`)
- Quantifiers (`{32}`, `+`, `*`, `?`)
- Alternation (`live|test`)
- Anchors (`^`, `$`)
- Lookaheads and lookbehinds (limited support)

## Dry Runs

Before publishing a custom pattern, you can perform a dry run to evaluate its effectiveness without generating real alerts. A dry run scans the repository (or, for organization patterns, a selected set of repositories) and shows you:

- How many matches the pattern produces.
- Sample matches with file names, line numbers, and the matched string.
- Whether the matches look like true positives or false positives.

Dry runs are essential for tuning patterns before deployment. The workflow:

1. Create the pattern in draft mode - Enter the regex components and save as a draft.
2. Run a dry run - Select one or more repositories to scan. GitHub will execute the pattern against the full history of those repositories.
3. Review results - Examine the matches. If the false positive rate is too high, refine the pattern (tighten the regex, add before/after context, add additional match requirements).
4. Iterate - Repeat the dry run until you are satisfied with the precision.
5. Publish - Promote the pattern from draft to active. It will immediately begin scanning all pushes and (if historical scanning has not already been performed) the full Git history.

> **Key detail:** Dry runs scan the repository's current content and history. They do not generate alerts — they produce a preview report only. Once you publish the pattern, alerts will be created for any matches found during the first full scan.

## Further Reading

- [Defining custom patterns for secret scanning](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/using-advanced-secret-scanning-and-push-protection-features/custom-patterns/defining-custom-patterns-for-secret-scanning)
- [Managing custom patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/using-advanced-secret-scanning-and-push-protection-features/custom-patterns/managing-custom-patterns)
- [Supported secret scanning patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/supported-secret-scanning-patterns)
- [Push protection for custom patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/introduction/about-push-protection)
