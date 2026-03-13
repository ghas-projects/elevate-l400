# Python Poetry Monorepo Example

This is a sample Python project using Poetry, structured as a monorepo. The root and each submodule have their own dependencies. Some dependencies are only declared in submodules and will be missed by GitHub's default dependency graph.

## Structure
- `pyproject.toml` (root)
- `module_a/pyproject.toml`
- `module_b/pyproject.toml`
- ...

## How to Use
1. Install Poetry
2. Install dependencies in each module: `poetry install`
3. Use the provided scripts to extract and submit dependencies

See .github/copilot-instructions.md for more details.
