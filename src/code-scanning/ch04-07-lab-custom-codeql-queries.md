# Lab: Running Custom CodeQL Queries

In this lab, you will use published [CodeQL Coding Standards](https://github.com/github/codeql-coding-standards) packs to analyze a C/C++ codebase. You will start with raw packs, layer in query suites for finer control, and finish with a config file that uses query-filters to include and exclude rules. You will work with the [**coding-standards-examples**](https://github.com/ghas-samples/coding-standards-examples) repository, which contains intentional rule violations.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Lab repository | `coding-standards-examples` in your lab organization |
| GitHub Actions | Enabled on the repository |

---

## Part 1: Run Coding Standards Packs

**Goal:** Run MISRA, CERT, and AUTOSAR packs for C and C++ against source code in a single workflow and observe the alerts they produce.

### 1.1 Create a workflow with MISRA, CERT, and AUTOSAR packs

CodeQL treats C and C++ as a single language (`c-cpp`), so one workflow can analyze both. Create `.github/workflows/codeql-coding-standards.yml` that runs all five coding-standards packs and builds the entire project.

<details>
<summary>Hint</summary>

Use `github/codeql-action/init@v4` with `languages: c-cpp`. Reference all packs via an inline `config` block:

```yaml
config: |
  packs:
    - codeql/misra-c-coding-standards
    - codeql/cert-c-coding-standards
    - codeql/autosar-cpp-coding-standards
    - codeql/cert-cpp-coding-standards
    - codeql/misra-cpp-coding-standards

```

Build with `make all`, then run `github/codeql-action/analyze@v4`.

Make sure you switch from default to advanced workflows by disabling default CodeQL in the repository settings.

</details>

<details>
<summary>Solution</summary>

```yaml
name: "CodeQL Coding Standards"

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      packages: read
      actions: read
      contents: read

    steps:
      - uses: actions/checkout@v4

      - uses: github/codeql-action/init@v4
        with:
          languages: c-cpp
          config: |
            packs:
              - codeql/misra-c-coding-standards
              - codeql/cert-c-coding-standards
              - codeql/autosar-cpp-coding-standards
              - codeql/cert-cpp-coding-standards
              - codeql/misra-cpp-coding-standards

      - run: make all

      - uses: github/codeql-action/analyze@v4
        with:
          category: coding-standards
```

</details>

1. Commit, push, and run the workflow.
2. Go to Security > Code scanning alerts and note the total alert count across all three standards.

> **Checkpoint:** Because `c-cpp` is a single CodeQL language, one workflow can run all five MISRA, CERT, and AUTOSAR packs together. There's no need for separate jobs or workflows.

---

## Part 2 - Narrow with Query Suites

**Goal:** Reduce alert volume by using query suites that ship inside each coding-standards pack, so only required-level rules run.

### 2.1 Add query suites to the workflow

Each coding-standards pack ships with query suites under `codeql-suites/` (e.g. `misra-c-required.qls`, `cert-c-required.qls`, `autosar-required.qls`). Update `.github/workflows/codeql-coding-standards.yml` to run only required-level queries by appending `:codeql-suites/<suite>.qls` to each pack specifier.

<details>
<summary>Hint</summary>

MISRA and AUTOSAR suite names follow the pattern `<standard>-<language>-required.qls`, while CERT suites use `<standard>-<language>-l1.qls`. Append the suite path to the pack name with a colon separator, e.g. `codeql/misra-c-coding-standards:codeql-suites/misra-c-required.qls`.

</details>

<details>
<summary>Solution - updated init step</summary>

```yaml
      - uses: github/codeql-action/init@v4
        with:
          languages: c-cpp
          config: |
            packs:
                - codeql/misra-c-coding-standards:codeql-suites/misra-c-required.qls
                - codeql/cert-c-coding-standards:codeql-suites/cert-c-l1.qls
                - codeql/autosar-cpp-coding-standards:codeql-suites/autosar-required.qls
                - codeql/cert-cpp-coding-standards:codeql-suites/cert-cpp-l1.qls
                - codeql/misra-cpp-coding-standards:codeql-suites/misra-cpp-required.qls
```

</details>

1. Commit, push, and run again.
2. Compare the alert count to Part 1 - it should be noticeably lower because advisory-level rules are excluded.

> **Checkpoint:** Query suites let you control exactly which rules run from a pack. This is useful for phased rollouts where you start with required rules and expand later.

---

## Part 3 - Config File with Query Filters

**Goal:** Move the analysis configuration into a reusable config file and use `query-filters` to surgically include and exclude rules.

### 3.1 Create a config file

Create `.github/codeql/coding-standards.yml` that combines all the packs, disables the default queries, and uses `query-filters` to exclude recommendation-severity results and advisory-obligation AUTOSAR rules:

<details>
<summary>Hint</summary>

Use `disable-default-queries: true` so only your chosen packs run. Then use `query-filters` with two exclude entries — one for severity and one for tags:

```yaml
query-filters:
  - exclude:
      problem.severity: recommendation
  - exclude:
      tags contain: external/autosar/obligation/advisory
```

Filters are evaluated in order. A query is excluded if it matches any exclude filter.

</details>

<details>
<summary>Solution</summary>

```yaml
# .github/codeql/coding-standards.yml

disable-default-queries: true

packs:
   - codeql/misra-c-coding-standards
   - codeql/cert-c-coding-standards
   - codeql/autosar-cpp-coding-standards
   - codeql/cert-cpp-coding-standards
   - codeql/misra-cpp-coding-standards

query-filters:
  - exclude:
      problem.severity: recommendation
  - exclude:
      tags contain: external/autosar/obligation/advisory
```

</details>

### 3.2 Update the workflow to use the config file

Update `.github/workflows/codeql-coding-standards.yml` to point to the config file:

<details>
<summary>Hint</summary>

Use the `config-file` input:

```yaml
config-file: ./.github/codeql/coding-standards.yml
```

</details>

   

<details>
<summary>Solution</summary>

```yaml
name: "CodeQL Coding Standards"

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      packages: read
      actions: read
      contents: read

    steps:
      - uses: actions/checkout@v4

      - uses: github/codeql-action/init@v4
        with:
          languages: c-cpp
          config-file: ./.github/codeql/coding-standards.yml

      - run: make all

      - uses: github/codeql-action/analyze@v4
        with:
          category: coding-standards
```

</details>

1. Commit, push, and run the workflow.
2. Confirm that no recommendation-severity alerts appear (filtered by the config).

### 3.3 Switch to includes, filter by ID, and layer security-extended

Replace the exclude filters with an include approach and layer on additional queries from the workflow. Update `.github/codeql/coding-standards.yml` so that only error-severity queries whose ID matches a CERT rule pattern are kept, then update the workflow to add `security-extended` on top:

<details>
<summary>Hint - config file</summary>

Replace the `query-filters` block with include filters. The `id` property supports regex - CERT C query IDs follow the pattern `cpp/cert/.*`. When the first filter is `include`, only matching queries are kept (opt-in model). You can chain includes - a query must match all include filters to be kept:

```yaml
query-filters:
  - include:
      id: /cpp/cert/.*/
  - include:
      problem.severity: error
```

</details>

<details>
<summary>Hint - workflow</summary>

Use the `queries` input with a `+` prefix so it adds to (rather than replaces) the config:

```yaml
queries: +security-extended
```

</details>

<details>
<summary>Solution - config file</summary>

```yaml
# .github/codeql/coding-standards.yml

disable-default-queries: true

packs:
   - codeql/misra-c-coding-standards
   - codeql/cert-c-coding-standards
   - codeql/autosar-cpp-coding-standards
   - codeql/cert-cpp-coding-standards
   - codeql/misra-cpp-coding-standards

query-filters:
  - include:
      id: /cpp/cert/.*/
  - include:
      problem.severity: error
```

</details>

<details>
<summary>Solution - updated init step</summary>

```yaml
      - uses: github/codeql-action/init@v4
        with:
          languages: c-cpp
          config-file: ./.github/codeql/coding-standards.yml
          queries: +security-extended
```

</details>

> If you also use a configuration file for custom settings, any additional packs or queries specified in your workflow are used instead of those specified in the configuration file. If you want to run the combined set of additional packs or queries, prefix the value of packs or queries in the workflow with the `+` symbol.

1. Commit, push, and run the workflow.
2. Verify the alert count dropped significantly. Only error-severity CERT results should remain from the config (AUTOSAR/MISRA alerts are gone because their IDs don't match the pattern), plus alerts from the `security-extended` suite layered on top.

> **Checkpoint:** Config files let you centralize and share analysis settings across repos, while `query-filters` give you fine-grained control over what gets reported. The `+` prefix lets individual repos layer additional queries without modifying the shared config.

---

## Discussion

1. How would you introduce coding standard checks to a large codebase with hundreds of pre-existing violations without overwhelming developers?

2. How would you distribute a baseline CodeQL config file across 200 repositories without each team manually adding it?

---

## Further Reading

- [CodeQL Coding Standards](https://github.com/github/codeql-coding-standards)
- [Customizing your advanced setup](https://docs.github.com/en/code-security/code-scanning/creating-an-advanced-setup-for-code-scanning/customizing-your-advanced-setup-for-code-scanning)
- [Creating CodeQL query suites](https://docs.github.com/en/code-security/codeql-cli/using-the-advanced-functionality-of-the-codeql-cli/creating-codeql-query-suites)
