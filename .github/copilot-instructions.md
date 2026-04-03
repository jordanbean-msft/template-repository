---
description: Repository-wide guidance for implementing and validating this Python project
---

## Purpose

This repository is a Python project using FastAPI, uv for package management, and Terraform for Azure infrastructure. Keep implementation simple, explicit, and testable.

## Project layout

```
src/template_repository/   # Application source code
tests/                     # Test suite
infra/                     # Terraform infrastructure-as-code
```

## Required engineering standards

- Always use uv for environment and dependency management.
- Keep Python imports at the top of files, grouped and sorted.
- Do not use lazy imports or wrap imports in try/except blocks.
- Prefer pydantic-settings for centralized configuration.
- Use structured logging.
- Use explicit type hints on public functions and class methods.
- Keep functions small and composable.
- Use dependency injection for external services.
- Favor async interfaces for network-bound operations.

## Build and validation commands

- Sync environment: `uv sync --extra dev`
- Run lint: `uv run ruff check .`
- Run all tests: `uv run pytest`
- Enforce warning-free tests: `uv run pytest -W error`
- Run with coverage: `uv run pytest --cov --cov-report=xml --cov-report=term-missing`
- Run API: `uv run template-repository`

## Testing patterns

- Cover happy paths and key failure paths.
- Mock external dependencies (API calls, Azure credentials, databases).
- Keep tests deterministic and independent.
- Use clear arrange-act-assert structure.
- Use async tests for async behavior.
- Use `httpx.AsyncClient` with FastAPI's `TestClient` for API testing.
- Prefer deterministic mocks over network calls.

## Pull request quality bar

- Include tests for all newly added behavior.
- Do not merge with test warnings. Treat warnings as failures.
- Keep commit scope coherent and focused.

## Infrastructure

- Terraform is used for Azure infrastructure under `infra/`. See `.github/instructions/terraform.instructions.md` for detailed IaC conventions.
- Azure Developer CLI (`azd`) orchestrates provisioning and deployment.
- Remote state is stored in Azure Blob Storage (configured via `provider.conf.json`).
- azd environment variables flow into Terraform via `main.tfvars.json` placeholder substitution (`${VAR_NAME}` syntax). Core variables (`AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`, `AZURE_LOCATION`, `AZURE_ENV_NAME`, `AZURE_RESOURCE_GROUP`, `AZURE_PRINCIPAL_ID`) are set during `azd init` or `azd env set`.
- Terraform outputs (SCREAMING_SNAKE_CASE) are captured by azd and become environment variables for deployment and application runtime.
- Prefer Azure Verified Modules (`Azure/avm-res-*`) over raw `azurerm_*` resources. Look up the latest versions on the Terraform registry.
- The resource group is always pre-created and passed in — never create it in Terraform.
