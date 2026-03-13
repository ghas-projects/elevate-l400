# Introduction
Welcome to the GitHub Advanced Security (GHAS) L400 course - an advanced, instructor-led training program designed for security engineers, DevSecOps practitioners, and engineering leaders who are ready to move beyond foundational knowledge and master the full capabilities of GitHub's native security platform.

## Who This Course Is For

This course is aimed at practitioners who already have working familiarity with GitHub Advanced Security (GHAS) and want to deepen their expertise in securing software at scale.

The following topics are considered assumed knowledge and will not be covered from scratch:

- GitHub repositories, Actions workflows, and CI/CD fundamentals
- Dependabot alerts, security updates, and version updates
- Code scanning basics - enabling default setup, reviewing alerts, and understanding severity levels
- Secret scanning fundamentals - how GitHub detects secrets and the partner program

If any of these areas are unfamiliar, review the [GHAS documentation](https://docs.github.com/en/enterprise-cloud@latest/code-security) before the session. There are also some chapters on these topics in this course for background reading, but may not be covered in depth during the session.

## What This Course Covers

The course is organized into three major modules, each targeting a core pillar of GitHub Advanced Security:

### Module 1: Supply Chain Security

Your application is only as secure as its weakest dependency. Building on your existing knowledge of Dependabot, this module covers advanced supply chain security topics:

- **Dependency Graph** - How GitHub maps your dependency tree and why visibility is the foundation of supply chain security.
- **Dependency Submission API** - Extending the dependency graph with build-time and runtime dependencies that aren't captured from manifest files alone.
- **SBOMs, Attestation, and SLSA** - Generating software bills of materials, signing build artifacts with attestations, and achieving SLSA supply chain integrity levels.
- **Dependabot Alerts** - Advanced configuration, grouping, and triaging alerts at scale.
- **Dependabot Security and Version Updates** - Fine-tuning update strategies and managing updates across an organization.
- **Dependency Review** - Catching vulnerable dependency changes in pull requests before they reach the default branch.

### Module 2: Secret Scanning

Leaked credentials are among the most exploited attack vectors in real-world breaches. This module covers:

- **What Is Secret Scanning** - Deep dive into how GitHub detects over 200 secret types, the partner notification program, alert lifecycle, and validity checks.
- **Push Protection** - Blocking secrets at the push boundary before they ever reach the remote repository, and managing bypass workflows.
- **Custom Patterns** - Defining organization-specific secret patterns using regular expressions to detect internal tokens and proprietary credential formats.

### Module 3: Code Scanning

Building on your existing knowledge of code scanning and CodeQL, this module goes deep on advanced customization and extensibility:

- **What Is Code Scanning** - Architecture deep-dive into CodeQL's semantic analysis, advanced setup, alert management at scale, and Copilot Autofix.
- **CodeQL CLI** - Overview of CodeQL CLI commands.
- **CodeQL Fundamentals** - Foundational concepts in the CodeQL ecosystem.
- **CodeQL Models as Data** - Teaching CodeQL about internal frameworks and unmodeled libraries using declarative YAML source/sink/summary models, without writing any QL.
- **Custom CodeQL Query Packs** - Writing, testing, publishing, and running your own CodeQL queries to detect organization-specific vulnerability patterns.

## How This Course Works

This is an instructor-led course. Your instructor will guide you through the material, provide demonstrations, and facilitate hands-on labs. Each module builds in complexity, with chapters designed to be followed in order.

Chapters combine conceptual depth with practical configuration details. Labs provide hands-on exercises that reinforce the material using your dedicated lab organization. The goal is not just to understand *what* GHAS does, but to know *how to configure, tune, and operate it* at enterprise scale.

## Let's Get Started

Proceed to [Before You Begin](ch01-00-before-you-begin.md) for prerequisites and setup instructions.