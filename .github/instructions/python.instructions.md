---
description: Python coding standards for this repository
applyTo: "**/*.py"
---

## Python Standards

* Use Python 3.12-compatible syntax and typing.
* Keep imports at file top, grouped and sorted.
* Avoid broad except blocks unless translating errors with clear context.
* Prefer explicit type hints on public functions and class methods.
* Keep functions small and composable.
* Use dependency injection for external services.
* **For LLM interactions, use Microsoft Agent Framework** (`microsoft-agents`). Never use `azure-ai-projects` directly if Agent Framework provides the needed functionality.

## Runtime behavior

* Route all environment reads through the settings model (pydantic-settings).
* Emit logs with stable structured context.
* Favor async interfaces for network-bound operations.
