# Running Custom CodeQL Queries

The out of the box query suites that ship with CodeQL cover the most common vulnerability classes, but every organization has unique security requirements. Internal coding standards, domain-specific vulnerability patterns, compliance mandates, and proprietary frameworks often demand analysis rules that no general-purpose suite can provide. The advanced setup gives you full control over which queries run, how they are selected, and how they are distributed across your organization.

## Built-in Query Suites

Before customizing anything, it is worth understanding what already ships out of the box. GitHub provides three built-in query suites per language:

| Suite Name | What It Includes | False Positive Rate | Available In |
|---|---|---|---|
| `default` / `code-scanning` | Highly precise, vetted queries | Very low | Default + Advanced setup |
| `security-extended` | Everything in `default` + extra security queries | Low to Medium | Default + Advanced setup |
| `security-and-quality` | Everything in `security-extended` + code quality queries | Medium | Advanced setup only |

Each suite cascades: `security-and-quality` includes everything in `security-extended`, which includes everything in `default`. For many teams, switching from `default` to `security-extended` is all that is needed. The rest of this chapter covers what to do when the built-in suites are not enough.

## Customization Mechanisms

There are three complementary mechanisms for customizing which queries run in code scanning:

| Mechanism | Purpose |
|---|---|
| **Query Packs** | Package, version, distribute, and run collections of queries |
| **Query Suites** | Select, filter, and compose subsets of queries from one or more packs |
| **Config Files** | Orchestrate packs and suites together with additional controls (query-filters, per-language packs, registries) |

The mental model is straightforward: a query pack *contains* queries, a query suite *selects* queries, and a config file *orchestrates* packs and suites together while adding capabilities that neither can provide alone.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Config File (.yml)                           │
│  Orchestration layer                                                │
│                                                                     │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐ │
│  │   packs:                     │  │   queries:                   │ │
│  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │ │
│  │  │  Query Pack            │  │  │  │  Query Suite (.qls)    │  │ │
│  │  │  qlpack.yml            │  │  │  │  - include:            │  │ │
│  │  │  src/*.ql              │  │  │  │  - exclude:            │  │ │
│  │  │  suites/*.qls          │  │  │  │  - import:             │  │ │
│  │  └────────────────────────┘  │  │  └────────────────────────┘  │ │
│  └──────────────────────────────┘  └──────────────────────────────┘ │
│                                                                     │
│  + query-filters          (surgical include/exclude)                │
│  + disable-default-queries (full control)                           │
│  + per-language packs      (multi-language repos)                   │
│  + registries              (GHES + ghcr.io)                         │
└─────────────────────────────────────────────────────────────────────┘
```

## Query Packs

### What Is a Query Pack?

A query pack is a versioned, distributable package of CodeQL queries (`.ql` files) and optionally library files (`.qll`). It is the primary unit of distribution for CodeQL content.

There are two types of CodeQL packs:

| Type | Contains | Executable? | `library` field |
|---|---|---|---|
| **Query pack** | `.ql` + optional `.qll` files | Yes | `false` or omitted |
| **Library pack** | `.qll` files only | No (used as dependency) | `true` |

> **Note:** Model packs (covered in [CodeQL Models as Data](ch04-03-codeql-models-as-data.md)) are a specialized form of library pack that use `extensionTargets` and `dataExtensions` to extend CodeQL coverage for custom dependencies. They are not a separate pack type.

### Pack Structure and `qlpack.yml`

Every query pack has a `qlpack.yml` manifest at its root. This file defines the pack's identity, version, dependencies, and default behavior.

```yaml
# qlpack.yml
name: my-org/security-queries
version: 1.0.0
description: Custom security queries for our organization
license: MIT
dependencies:
  codeql/javascript-all: ^1.0.0
defaultSuiteFile: suites/default.qls
```

For the full property reference, see [qlpack.yml properties](https://docs.github.com/en/code-security/codeql-cli/reference/qlpack-overview#qlpackyml-properties).

### Creating a Query Pack

Use the CodeQL CLI to scaffold a new pack:

```bash
# Initialize a new query pack
codeql pack init my-org/security-queries

# This creates:
#   my-org/security-queries/
#   └── qlpack.yml
```

Add your `.ql` query files and optionally define a default suite.

### Adding Dependencies

Your queries likely depend on the standard CodeQL libraries for a given language:

```bash
# Add a dependency on the JavaScript standard library
codeql pack add codeql/javascript-all@^1.0.0

# Install all dependencies and create the lock file
codeql pack install
```

This updates your `qlpack.yml` and generates a `codeql-pack.lock.yml` lock file that pins transitive dependencies to exact versions. Commit the lock file for reproducible builds.

### Publishing to the GitHub Container Registry

Once your pack is ready, publish it so it can be consumed by workflows across your organization:

```bash
# Authenticate (option 1: environment variable)
export GITHUB_TOKEN=ghp_xxxxxxxxxxxx

# Authenticate (option 2: stdin)
echo "ghp_xxxxxxxxxxxx" | codeql pack publish --github-auth-stdin

# Publish the pack (run from the pack root)
codeql pack publish
```

The pack appears in the Packages section of your GitHub organization.
To download a published pack (for example, for local testing):

```bash
# Download the latest version
codeql pack download my-org/security-queries

# Download a specific version
codeql pack download my-org/security-queries@1.2.3
```

### Using Query Packs in Workflows

Reference published query packs using the **`packs`** key in `github/codeql-action/init`:

```yaml
- uses: github/codeql-action/init@v4
  with:
    languages: javascript
    packs: my-org/security-queries@^1.0.0
```

**Pack specifier format:** `<scope>/<name>[@<version>][:<path>]`

| Specifier | Behavior |
|---|---|
| `my-org/pack1` | Latest version, all default queries |
| `my-org/pack1@1.2.3` | Exact version `1.2.3` |
| `my-org/pack1@~1.2.0` | Latest patch (>=1.2.0, <1.3.0) |
| `my-org/pack1@^1.0.0` | Latest minor (>=1.0.0, <2.0.0) |
| `my-org/pack1@1.0.0:src/security` | Only queries under `src/security` |
| `my-org/pack1:path/to/query.ql` | A single specific query |
| `my-org/pack1:suites/security.qls` | A specific query suite in the pack |

Multiple packs use comma separation:

```yaml
packs: my-org/security-queries@^1.0.0,my-org/quality-queries@^2.0.0
```

## Query Suites

### What Is a Query Suite?

A query suite is a YAML file (`.qls` extension) that defines which queries to run by selecting and filtering from available queries. It does not *contain* queries; it *points to* them.

Think of a query suite as a playlist: it references tracks (queries) from albums (packs) and can filter by genre (tags), rating (precision), or mood (severity).

### Suite File Structure

A `.qls` file is a YAML sequence of instructions executed in order. The result is a set of selected queries.

```yaml
# suites/security-high-precision.qls
- description: High-precision security queries

# Step 1: Locate queries
- queries: .
  from: codeql/javascript-queries

# Step 2: Filter to only security queries with high precision
- include:
    tags contain: security
    precision:
      - high
      - very-high
```

### Locating Queries

Every suite must start with at least one source instruction that tells CodeQL where to find queries:

**Single file:**

```yaml
- query: src/security/SqlInjection.ql
```

Path is relative to the pack containing the suite.

**Directory (recursive):**

```yaml
# All queries in a subdirectory of the current pack
- queries: src/security

# All queries from a different pack
- queries: .
  from: codeql/javascript-queries
  version: ^1.0.0
```

**Default suite of a named pack:**

```yaml
- qlpack: codeql/javascript-queries
  version: ^1.0.0
```

This resolves to the pack's `defaultSuiteFile` (or all queries in the pack if no default is defined).

### Filtering with `include` and `exclude`

After locating queries, apply filters to narrow down the selection.

#### Filter Properties

| Property | Description | Example Values |
|---|---|---|
| `id` | Unique query identifier | `js/sql-injection` |
| `kind` | Query kind | `problem`, `path-problem` |
| `name` | Query name | `SQL injection` |
| `tags` | Exact tag match | `security` |
| `tags contain` | Tag contains one of the values | `security`, `correctness` |
| `tags contain all` | Tag contains ALL values | `["security", "external/cwe"]` |
| `precision` | Query precision | `high`, `very-high`, `medium`, `low` |
| `problem.severity` | Problem severity | `error`, `warning`, `recommendation` |
| `query filename` | File name (last path component) | `SqlInjection.ql` |
| `query path` | Path relative to enclosing pack | `src/security/SqlInjection.ql` |

Values can be a string, a `/regex/`, or a list of either.

#### How Filter Order Works

> **Key insight:** The first filter after source instructions determines the default behavior.

- If the first filter is `include`, all queries are excluded by default (opt-in model)
- If the first filter is `exclude`, all queries are included by default (opt-out model)

```yaml
# OPT-IN approach: Start with include, only matching queries kept
- qlpack: codeql/javascript-queries
- include:
    tags contain: security     # Only security queries are selected
```

```yaml
# OPT-OUT approach: Start with exclude, all queries except matching
- qlpack: codeql/javascript-queries
- exclude:
    precision: low             # ALL queries EXCEPT low-precision
```

#### Filter Examples

For detailed filter examples (AND/OR logic, exclude by ID, regex patterns, etc.), see [Creating CodeQL query suites](https://docs.github.com/en/code-security/codeql-cli/using-the-advanced-functionality-of-the-codeql-cli/creating-codeql-query-suites#filtering-the-queries-in-a-query-suite).

### Reusing Suite Definitions

Query suites support `import` (to include queries from another suite) and `apply` (to inline reusable filter conditions from another file). For details and examples, see [Reusing existing query suite definitions](https://docs.github.com/en/code-security/codeql-cli/using-the-advanced-functionality-of-the-codeql-cli/creating-codeql-query-suites#reusing-existing-query-suite-definitions).

### Using Query Suites in Workflows

Reference query suites using the **`queries`** key:

```yaml
- uses: github/codeql-action/init@v4
  with:
    languages: javascript
    queries: security-extended
```

You can also reference custom `.qls` files:

```yaml
- uses: github/codeql-action/init@v4
  with:
    languages: javascript
    queries: ./.github/codeql/suites/my-custom-suite.qls
```

Or combine multiple references with commas:

```yaml
queries: security-extended,./.github/codeql/suites/extra-checks.qls
```

## Config Files

### What Is a Config File?

A config file is a YAML file that acts as the *orchestration layer for your code scanning setup. It combines query packs and query suites together, while adding capabilities that neither can provide on its own.

Config files live in your repository (typically at `.github/codeql/codeql-config.yml`) and are referenced via the `config-file:` or `config:` workflow keys.

Capabilities unique to config files:

| Capability | Description |
|---|---|
| `disable-default-queries` | Replace the default query set entirely for full control |
| Per-language `packs` | Specify different query packs for each language in a multi-language repo |
| `registries` | Pull packs from multiple container registries (GHES + ghcr.io) |
| Combining `packs` + `queries` | Reference both packs and suites in a single unified configuration |
| `query-filters` | Apply include/exclude filtering after all packs and suites have been resolved |

### Config File Structure

```yaml
# .github/codeql/codeql-config.yml

# Optional: disable the default code-scanning queries
disable-default-queries: true

# Query packs to run (flat list or per-language)
packs:
  javascript-typescript:
    - my-org/js-security@^1.0.0
    - my-org/js-quality@^2.0.0
  python:
    - my-org/python-security@^1.0.0

# Query suites and individual queries
queries:
  - uses: security-extended
  - uses: ./.github/codeql/suites/org-required-checks.qls

# Surgical post-resolution filters
query-filters:
  - exclude:
      id: js/redundant-assignment
  - exclude:
      problem.severity: recommendation

# Multi-registry support
registries:
  - url: https://containers.ghes.mycompany.com/v2/
    packages:
      - my-company/*
    token: ${{ secrets.GHES_TOKEN }}
  - url: https://ghcr.io/v2/
    packages: "*/*"
    token: ${{ secrets.GHCR_TOKEN }}
```

### Query Filters

`query-filters` use the same include/exclude filtering as query suites (by ID, severity, tags, etc.), but applied at the config file level after all packs and suites have been resolved.

```yaml
# Exclude specific noisy queries by ID
query-filters:
  - exclude:
      id: js/redundant-assignment
  - exclude:
      id: js/useless-assignment-to-local
```

```yaml
# Exclude all low-severity findings
query-filters:
  - exclude:
      problem.severity:
        - warning
        - recommendation
```

```yaml
# Only include security-tagged queries
query-filters:
  - include:
      tags contain: security
```

> **Use case:** "I want `security-extended`, but query X is too noisy for our codebase." Query filters are the right tool for this.

### Per-Language Packs

In multi-language repos, you can specify different packs per language:

```yaml
packs:
  javascript-typescript:
    - my-org/js-security@^1.0.0
    - my-org/js-quality@^2.0.0
  python:
    - my-org/python-security@^1.0.0
  java-kotlin:
    - my-org/java-security@^1.0.0
```

### Multi-Registry Support

Enterprise users can pull packs from both GitHub Enterprise Server and the public GitHub Container Registry:

```yaml
registries:
  - url: https://containers.ghes.mycompany.com/v2/
    packages:
      - my-company/*
    token: ${{ secrets.GHES_TOKEN }}
  - url: https://ghcr.io/v2/
    packages: "*/*"
    token: ${{ secrets.GHCR_TOKEN }}
```

### Using Config Files in Workflows

**Option 1: External config file**

```yaml
- uses: github/codeql-action/init@v4
  with:
    languages: ${{ matrix.language }}
    config-file: ./.github/codeql/codeql-config.yml
```

**Option 2: Inline config (no separate file)**

```yaml
- uses: github/codeql-action/init@v4
  with:
    languages: ${{ matrix.language }}
    config: |
      queries:
        - uses: security-extended
      packs:
        - my-org/security-queries@^1.0.0
      query-filters:
        - exclude:
            id: js/redundant-assignment
```

**Option 3: Config file with additive overrides (`+` prefix)**

By default, inline `queries:`/`packs:` values override the config file. Prefix with `+` to combine them:

```yaml
- uses: github/codeql-action/init@v4
  with:
    config-file: ./.github/codeql/codeql-config.yml
    # The + prefix ADDS to the config file instead of overriding
    queries: +security-and-quality
    packs: +my-org/extra-checks@^1.0.0
```

## Three-Way Comparison

| Aspect | Query Pack | Query Suite | Config File |
|---|---|---|---|
| **What it is** | A distributable package of queries | A selection/filter definition over queries | An orchestration layer for packs, suites, and controls |
| **File format** | Directory with `qlpack.yml` + `.ql`/`.qll` | Single `.qls` file (YAML) | Single `.yml` file (YAML) |
| **Contains queries?** | Yes, actual `.ql` files | No, references and filters only | No, references packs and suites |
| **Distribution** | Published to GitHub Container Registry | Stored in a repo or inside a query pack | Stored in a repo (checked in) |
| **Versioning** | Full SemVer + lock files | No independent versioning | No independent versioning |
| **Workflow key** | `packs:` | `queries:` | `config-file:` / `config:` |
| **Pre-compilation** | `.qlx` bundled at publish time (fast) | Resolved at analysis time | Resolved at analysis time |
| **Unique capabilities** | Dependency management, pre-compiled `.qlx` | Metadata filtering, import/apply/compose | Per-language packs, registries, `disable-default-queries`, combine packs+suites |

### When to Use Each

**Use Query Packs when you want to:**
- Package and distribute queries across repositories
- Version your queries with SemVer
- Get pre-compiled queries for faster CI
- Manage dependencies between query and library packs
- Publish to GitHub Container Registry for organization-wide use

**Use Query Suites when you want to:**
- Select a subset of queries from an existing pack
- Filter by metadata (tags, severity, precision)
- Compose queries from multiple sources
- Define what a scanning run should include

**Use Config Files when you want to:**
- Combine packs and suites in one place
- Disable default queries for full control
- Define per-language packs for multi-language repos
- Pull packs from multiple registries (GHES + ghcr.io)
- Apply query-filters on top of resolved packs and suites

> **In practice:** Most organizations use all three together. Packs to distribute, suites to select, and config files to orchestrate.

## Workflow Examples

### Example 1: Built-in suite plus custom pack

```yaml
name: "CodeQL Analysis"

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '30 1 * * 1'

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      packages: read
      actions: read
      contents: read

    strategy:
      fail-fast: false
      matrix:
        language: ['javascript']

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v4
        with:
          languages: ${{ matrix.language }}
          queries: security-extended
          packs: my-org/security-queries@^1.0.0

      - name: Autobuild
        uses: github/codeql-action/autobuild@v4

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v4
```

### Example 2: Multiple packs with path specifiers

```yaml
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v4
        with:
          languages: javascript
          packs: >
            my-org/security-queries@^1.0.0,
            my-org/quality-queries@^2.0.0:src/maintainability
```

### Example 3: Config file with the `+` prefix

```yaml
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v4
        with:
          languages: javascript
          config-file: ./.github/codeql/codeql-config.yml
          queries: +security-and-quality
          packs: +my-org/extra-checks@^1.0.0
```

### Example 4: Inline configuration

```yaml
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v4
        with:
          languages: ${{ matrix.language }}
          config: |
            disable-default-queries: true
            queries:
              - uses: security-extended
              - uses: ./.github/codeql/suites/custom-security.qls
            packs:
              - my-org/security-queries@^1.0.0
            query-filters:
              - exclude:
                  id: js/redundant-assignment
              - exclude:
                  problem.severity: recommendation
```

## CLI Reference

| Command | Purpose |
|---|---|
| `codeql pack init <scope>/<pack>` | Scaffold a new pack |
| `codeql pack add <scope>/<name>@version` | Add a dependency |
| `codeql pack install` | Install dependencies and create lock file |
| `codeql pack publish` | Publish to GitHub Container Registry |
| `codeql pack download <scope>/<pack>[@version]` | Download a published pack |
| `codeql pack create` | Build a pack locally (without publishing) |
| `codeql resolve queries <suite.qls>` | Preview which queries a suite selects |
| `codeql database analyze <db> <pack-or-suite>` | Run analysis with a pack or suite |

## Best Practices

### General

1. **Start with built-in suites.** Use `security-extended` or `security-and-quality` before writing custom queries.
2. **Use query packs for distribution.** If multiple repos need the same queries, publish a pack.
3. **Use query suites for selection.** If you need to filter or compose, write a `.qls` file.
4. **Use config files for orchestration.** Combine packs, suites, and additional controls in one place.
5. **Commit lock files.** Always commit `codeql-pack.lock.yml` for reproducible builds.
6. **Version your packs.** Use SemVer ranges (`^1.0.0`) in workflows to get bug fixes automatically.

### Query Packs

7. **Separate queries from tests.** Keep test files in a separate pack or directory.
8. **Set a `defaultSuiteFile`.** This gives consumers a curated experience by default.
9. **Add query metadata.** Every `.ql` file should have `@id`, `@name`, `@description`, `@kind`, `@tags`, `@precision`, and `@problem.severity`.
10. **Pin major versions.** Use `^1.0.0` (not `*`) to avoid breaking changes.

### Query Suites

11. **Use `codeql resolve queries`.** Always verify your suite selects the expected queries.
12. **Understand filter order.** The first `include`/`exclude` determines the default behavior.
13. **Create reusable filter files.** Use `apply:` to share filter logic across suites.
14. **Start from a built-in suite.** Use `import:` to extend `security-extended` rather than building from scratch.

### Config Files

15. **Use `query-filters`.** Surgically exclude noisy queries without rewriting suites.
16. **Use per-language packs.** In multi-language repos, specify packs per language in the config file.
17. **Use the `+` prefix.** When combining config files with inline `queries:`/`packs:` values.
18. **Prefer `config-file:` over `config:`.** External config files are easier to maintain and review for complex setups.

## Further Reading

- [Customizing your advanced setup](https://docs.github.com/en/code-security/code-scanning/creating-an-advanced-setup-for-code-scanning/customizing-your-advanced-setup-for-code-scanning)
- [Creating CodeQL query suites](https://docs.github.com/en/code-security/codeql-cli/using-the-advanced-functionality-of-the-codeql-cli/creating-codeql-query-suites)
- [Customizing analysis with CodeQL packs](https://docs.github.com/en/code-security/codeql-cli/getting-started-with-the-codeql-cli/customizing-analysis-with-codeql-packs)
- [CodeQL documentation](https://codeql.github.com/docs/)
- [CodeQL standard query packs on GitHub](https://github.com/github/codeql)
