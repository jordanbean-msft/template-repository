# template-repository

[![CI](https://github.com/jordanbean-msft/template-repository/actions/workflows/ci.yml/badge.svg)](https://github.com/jordanbean-msft/template-repository/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/jordanbean-msft/template-repository)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.12%2B-blue)](https://www.python.org/)

## Architecture

![architecture](./.img/architecture.drawio.png)

## Disclaimer

**THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.**

## Setup from Template

When creating a new repository from this template, find-and-replace the following:

| Find | Replace with | Files affected |
| --- | --- | --- |
| `template-repository` | `your-repo-name` | `README.md`, `azure.yaml`, `pyproject.toml`, `.code-workspace` filename, `.github/copilot-instructions.md`, `infra/provider.conf.json` |
| `template_repository` | `your_repo_name` | `src/` directory name, `pyproject.toml`, `.code-workspace` (launch/task configs), `.github/copilot-instructions.md`, Python imports |
| `tmplrepo` | your short project name (1-8 chars) | `infra/variables.tf`, `infra/main.tfvars.json` |
| `jordanbean-msft/template-repository` | `your-org/your-repo-name` | `README.md` badge URLs |

Then:

1. Rename `src/template_repository/` to `src/your_repo_name/`
2. Rename `template-repository.code-workspace` to `your-repo-name.code-workspace`
3. Update the architecture diagram in `.img/architecture.drawio`
4. Update `.env.example` with your project's environment variables
5. Set remote state: `azd env set RS_PROJECT_NAME your-repo-name`

## Prerequisites

- [Python 3.12+](https://www.python.org/)
- [uv](https://docs.astral.sh/uv/)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd)
- [Terraform](https://www.terraform.io/downloads)

## Setup

1. Install all dependencies:

```bash
uv sync --extra dev
```

2. Install pre-commit hooks:

```bash
uv run pre-commit install
uv run pre-commit install --hook-type pre-push
```

3. Copy the environment file and fill in your values:

```bash
cp .env.example .env
```

4. Open the VS Code workspace:

```bash
code template-repository.code-workspace
```

## Run

```bash
uv run template-repository
```

## Validate

```bash
uv run ruff check .
uv run pytest
```

## Test with coverage

```bash
uv run pytest --cov --cov-report=xml --cov-report=term-missing
```

## Deploy

```bash
az login
azd up
```

## Links
