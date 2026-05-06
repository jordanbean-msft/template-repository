---
description: Python tooling conventions for this repository
applyTo: "**/*.py"
---

## Package Management — Always use `uv`

* `uv add <package>` — add a dependency (pulls latest version by default)
* `uv add --group dev <package>` — add a dev dependency (pulls latest version)
* `uv sync --extra dev` — install/sync environment with dev extras
* `uv sync --reinstall-package <name>` — force reinstall a specific package
* `uv run <tool>` — run a CLI tool within the managed environment
* `uv run pytest` — run tests
* `uv run ruff check .` — run linter
* Never use raw `pip install` in this project.
* **Always pull latest package versions** when adding new packages with `uv add`. Avoid pinning to older versions unless there's a specific compatibility reason (document in PR).
* **Latest GitHub Actions versions** must be used. Always update action versions to the latest available (e.g., `actions/checkout@v4`, `actions/setup-python@v5`). Check the official action repositories for the latest versions.

## Pre-release Packages

* Never use `--prerelease=allow` globally (it promotes all packages to pre-release).
* Pin specific pre-releases explicitly in `pyproject.toml`:
  ```toml
  [tool.uv]
  prerelease = "if-necessary-or-explicit"
  constraint-dependencies = [
      "some-dep>=1.0.0b5,<1.0.0b6",
  ]
  ```

## Linting and Formatting

* Lint: `uv run ruff check .`
* Auto-fix: `uv run ruff check . --fix`
* Format: `uv run ruff format .`
* Type check: `uv run mypy src/`

## Testing

* Run all tests: `uv run pytest`
* Strict mode (warnings as errors): `uv run pytest -W error`
* With coverage: `uv run pytest --cov --cov-report=xml --cov-report=term-missing`

## Pre-commit Hooks

* Install: `uv run pre-commit install && uv run pre-commit install --hook-type pre-push`
* Run manually: `uv run pre-commit run --all-files`
* **Fast checks on pre-commit**: linting (ruff), formatting (ruff), type checking (mypy), security scanning (bandit, semgrep), and infrastructure validation
* **Slower checks on pre-push**: pytest runs before pushing to remote, ensuring all tests pass before code goes upstream
* **Pre-commit must be fast** — commit should not be blocked for more than a few seconds
* Install both hook types to enable full protection: pre-commit validates quality, pre-push validates correctness

## Feature Completion Checklist — Static Code Analysis

**All local static analysis tools must run and succeed before a new feature is considered complete.**

Run all checks locally before submitting a PR:

```bash
# Python static analysis
uv run ruff check .                  # Lint check
uv run ruff format .                 # Auto-format code
uv run mypy src/                     # Type checking
uv run bandit -r src/                # Security scanning
uv run semgrep --config=p/security-audit src/  # Code analysis

# Tests
uv run pytest -W error               # Run tests (strict mode)

# Infrastructure (if modified)
cd infra && terraform fmt -check -recursive .  # Terraform format
cd infra && terraform validate                  # Terraform validation
cd infra && tflint --recursive                  # Terraform linting
cd infra && tfsec .                             # Terraform security
cd infra && checkov -d . --framework terraform  # Infrastructure compliance
```

Or run all pre-commit hooks at once:

```bash
uv run pre-commit run --all-files    # Runs all enabled pre-commit hooks
```

**Note**: The `pytest -W error` check is automatically run on `pre-push`, so ensure it passes before pushing. If any tool fails, fix the issues and commit/push again — do not skip or bypass these checks.

## Static Code Analysis and Security Scanning

* Run static analysis locally: `uv run ruff check .`, `uv run mypy src/`, `uv run bandit -r src/`
* **Pre-commit hooks** include fast static analysis tools (ruff, mypy, bandit, semgrep)
* **GitHub Actions pipeline** (`code-analysis.yml`) runs comprehensive static analysis and code scanning on every push and PR
* See `.github/workflows/code-analysis.yml` for the full suite of analysis tools run in CI/CD


## LLM and AI Interactions — Always use Microsoft Agent Framework

* **Always use the Microsoft Agent Framework** (`microsoft-agents`) for interactions with LLMs instead of lower-level Azure AI Projects library.
* Always install the **latest version** of `microsoft-agents` from PyPI.
* Only use `azure-ai-projects` (lower-level SDK) when there is no Agent Framework abstraction available for your use case.
* Agent Framework provides a higher-level, more maintainable interface for building agent systems and LLM interactions.
* Example: `uv add microsoft-agents` — this will pull the latest version.
* For agent-specific guidance, refer to Agent Framework documentation and examples in the repository's prompt files (`.github/prompts/`).

## Structured Output from LLMs — Always use Pydantic

* **Always request structured output from LLMs** using Pydantic models instead of parsing free-form text.
* Define clear Pydantic schemas (`BaseModel` subclasses) for all LLM responses.
* Use Microsoft Agent Framework's structured output capabilities when available, passing Pydantic models as response schemas.
* Benefits: type safety, validation, predictable responses, easier testing, and reduced parsing errors.
* Example: Instead of parsing JSON strings, use `response_format=MyPydanticModel` to get validated instances.
* Ensure all public LLM-facing functions have explicit type hints with Pydantic model return types.
