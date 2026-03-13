# Code Scanning

This module builds on your existing knowledge of code scanning and CodeQL to explore advanced configuration, extensibility, and customization. You are expected to be familiar with enabling code scanning, reviewing alerts, and understanding severity levels.

Your instructor will guide you through the following topics:

- [**What Is Code Scanning**](ch04-01-what-is-code-scanning.md) - Architecture deep-dive into CodeQL's semantic analysis engine, default vs. advanced setup, alert management at scale, and Copilot Autofix.
- [**The CodeQL CLI**](ch04-02-codeql-cli.md) - Using the CodeQL CLI to create databases, run queries, and integrate CodeQL into custom workflows outside of GitHub Actions.
- [**CodeQL Foundations**](ch04-03-codeql-fundamentals.md) - Core CodeQL concepts including the query language, dataflow analysis, taint tracking, and how queries detect vulnerabilities.
- [**CodeQL Models as Data**](ch04-04-codeql-models-as-data.md) - Teaching CodeQL about internal frameworks and unmodeled libraries using declarative YAML source/sink/summary models, without writing any QL.
- [**Lab: CodeQL Models as Data**](ch04-05-lab-codeql-models-as-data.md) - Hands-on: adding the CodeQL Community Packs at the organization level.
- [**Custom CodeQL Query Packs**](ch04-06-custom-codeql-query-packs.md) - Writing, packaging, publishing, and running your own CodeQL queries to detect organization-specific vulnerability patterns.
- [**Lab: Running Custom CodeQL Queries**](ch04-07-lab-custom-codeql-queries.md) - Hands-on: running coding standard packs, query suites, and config files against a C/C++ codebase.
