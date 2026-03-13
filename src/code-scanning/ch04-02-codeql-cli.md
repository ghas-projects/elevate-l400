# The CodeQL CLI

Understanding how CodeQL works under the hood gives you the knowledge to debug analysis failures, optimize performance, and integrate CodeQL into any CI/CD system. This chapter walks through the CodeQL CLI, the tool that powers every code scanning workflow on GitHub, whether you are using default setup, advanced setup, or a third-party pipeline.

## Three Ways to Run CodeQL

There are three primary ways to run CodeQL analysis:

| Method | Description |
|---|---|
| **Default setup** | Automatically detects languages, selects query suites, and configures triggers. Zero configuration required. |
| **Advanced setup** | Uses a custom GitHub Actions workflow (`.github/workflows/codeql.yml`), giving you fine-grained control over languages, queries, and build steps. |
| **CodeQL CLI in an external CI/CD pipeline** | Runs CodeQL outside of GitHub Actions, with results uploaded back to GitHub for code scanning. |

All three methods rely on the same underlying tool: the CodeQL CLI.

## Tooling Setup

You should always download the CodeQL bundle from the Releases page of the [`codeql-action` repository](https://github.com/github/codeql-action/releases). The bundle includes the CodeQL CLI along with a set of queries that are guaranteed to be compatible with that release.

> **Important:** Downloading the CodeQL CLI and query packs separately is not recommended. Mismatched versions can lead to incompatibilities and unexpected analysis behavior.

Key repositories:

- [CodeQL Action source code](https://github.com/github/codeql-action/) - the GitHub Action that wraps the CLI
- [CodeQL queries](https://github.com/github/codeql) - the open-source query repository

## How CodeQL Analysis Works

A full CodeQL analysis follows four steps: database creation, build tracing, analysis, and cleanup. GitHub Actions exposes these as separate low-level commands to support the wide variety of repository structures and build systems in the ecosystem. For most local or custom CI/CD workflows, higher-level commands handle these steps automatically (see [The Common Case](#the-common-case) at the end of this chapter).

## Step 1: Creating a Database

The first step in any CodeQL analysis is creating a CodeQL database. The database is a structured representation of the source code, which is later queried during the analysis phase.

### Database Initialization

In GitHub Actions, the database initialization step is performed by:

```yaml
github/codeql-action/init@v4
```

All supported input options are declared in the [`init` action.yml](https://github.com/github/codeql-action/blob/main/init/action.yml).

This action invokes the equivalent of the [`codeql database init`](https://docs.github.com/en/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-init) CLI command.

**What this step does:**

- Creates the directory structure for a CodeQL database
- Configures language-specific extraction settings
- Prepares the database to receive extracted code data
- Does **not** yet populate the database with a raw QL dataset

### Example CLI Command

```bash
codeql database init \
  --force-overwrite \
  --db-cluster ${DIR} \
  --source-root=${SOURCE_DIR} \
  --calculate-language-specific-baseline \
  --sublanguage-file-coverage \
  --extractor-include-aliases \
  --language={LANGUAGE} \
  --codescanning-config=/home/runner/work/_temp/user-config.yaml \
  --build-mode=none
```

## Step 2: Tracing the Build

After the database is initialized, CodeQL needs to extract code into it. How this happens depends on the build mode.

### Build Modes

| Mode | Description | Supported Languages |
|---|---|---|
| **`none`** | No build is performed. CodeQL parses source files directly. | All interpreted languages, plus C/C++, C#, Java, and Rust |
| **`autobuild`** | CodeQL automatically detects and executes the build. | C/C++, C#, Go, Java, Kotlin, Swift |
| **`manual`** | Build steps are explicitly defined in the workflow. Recommended for complex or non-standard builds. | C/C++, C#, Go, Java, Kotlin, Swift |

### Autobuild in GitHub Actions

In GitHub Actions, [autobuild](https://github.com/github/codeql-action/blob/main/autobuild/action.yml) is triggered using:

```yaml
github/codeql-action/autobuild@v4
```

This invokes [`codeql database trace-command`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-trace-command), which runs build commands under a tracer and seeds the database with extracted data. Note that this does not finalize the database.

```bash
codeql database trace-command \
  --use-build-mode \
  --working-dir {DIR} \
  {DATABASE}
```

## Step 3: Performing the Analysis

In GitHub Actions, analysis and database finalization are performed using:

```yaml
github/codeql-action/analyze@v4
```

The supported input options are defined in the [`analyze` action.yml](https://github.com/github/codeql-action/blob/main/analyze/action.yml).

### Database Finalization

Before queries can run, the database must be finalized. This step invokes the [`codeql database finalize`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-finalize) command. Finalization takes the raw data from the trace step and produces a queryable QL dataset.

```bash
codeql database finalize \
  --finalize-dataset \
  --threads=2 \
  --ram=6915 \
  {DATABASE}
```

### Running Queries

The [`codeql database run-queries`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-run-queries) command runs one or more queries against a CodeQL database, saving the results to the results subdirectory of the database directory.

```bash
codeql database run-queries \
  --ram=6915 \
  --threads=2 \
  --expect-discarded-cache \
  --min-disk-free=1024 \
  -v \
  {DATABASE}
```

Built-in query suites include:

| Suite | Description |
|---|---|
| **`code-scanning`** (default) | High-precision security queries |
| **`security-extended`** | Broader security coverage with slightly higher false-positive rates |
| **`security-and-quality`** | Security queries plus code quality checks |

For details on built-in queries, see the [official documentation](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-queries/about-built-in-queries).

## Step 4: Cleanup and Bundling

### Cleanup

The [`codeql database cleanup`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-cleanup) command compacts a CodeQL database on disk. It deletes temporary data and makes the database as small as possible without degrading its future usefulness.

```bash
codeql database cleanup {DATABASE} --cache-cleanup=clear
```

### Bundle Database

The [`codeql database bundle`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-bundle) command creates a relocatable archive of a CodeQL database. This zips up the useful parts of the database, including only the mandatory components unless you specifically request results, logs, or TRAP data.

```bash
codeql database bundle {DATABASE} --output={DATABASE}.zip --name={NAME}
```

## The Common Case

When running CodeQL locally or configuring a custom CI/CD workflow, you typically do not need to use the low-level commands described above. GitHub Actions exposes these lower-level commands to support the wide variety of repository structures and build systems in the ecosystem. For most standard workflows, CodeQL provides higher-level commands that handle everything.

In the majority of cases, only two commands are required:

| Command | What It Does |
|---|---|
| [`codeql database create`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-create) | Creates, traces the build, and finalizes the database in a single step |
| [`codeql database analyze`](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-analyze) | Runs the queries and produces SARIF results |

These commands handle database initialization, build tracing, extraction, finalization, and query execution under the hood.

The analysis step produces a SARIF file, which can then be uploaded to GitHub using:

```yaml
github/codeql-action/upload-sarif@v4
```

## Further Reading

- [CodeQL CLI manual](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-cli-manual)
- [Setting up the CodeQL CLI](https://docs.github.com/en/enterprise-cloud@latest/code-security/code-scanning/using-codeql-code-scanning-with-your-existing-ci-system/installing-codeql-cli-in-your-ci-system)
- [CodeQL Action source code](https://github.com/github/codeql-action/)
- [About built-in queries](https://docs.github.com/en/enterprise-cloud@latest/code-security/reference/code-scanning/codeql/codeql-queries/about-built-in-queries)
