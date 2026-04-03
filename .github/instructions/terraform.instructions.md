---
description: Terraform infrastructure-as-code standards for this repository
applyTo: "infra/**/*.tf"
---

## Tooling and orchestration

- Use **Terraform** (not Bicep or ARM) for all Azure infrastructure.
- Use **Azure Developer CLI (azd)** to orchestrate provisioning (`azd provision`) and deployment (`azd deploy`).
- Store remote state in **Azure Blob Storage** via an `azurerm` backend. Backend configuration is supplied through `infra/provider.conf.json` with `${RS_*}` and `${AZURE_ENV_NAME}` placeholders that azd substitutes at runtime.
- Variable values are supplied through `infra/main.tfvars.json` with `${VAR_NAME}` placeholders that azd substitutes from its environment. This is the primary mechanism for passing configuration from azd to Terraform.
- Before provisioning, the operator sets remote state coordinates: `azd env set RS_RESOURCE_GROUP <rg>`, `azd env set RS_STORAGE_ACCOUNT <sa>`, `azd env set RS_CONTAINER_NAME <container>`.
- Always look up the latest stable versions of Terraform providers and Azure Verified Modules during development before pinning.

## Azure Developer CLI (azd) variable flow

- azd owns the lifecycle of environment variables. Terraform receives them via `main.tfvars.json` placeholder substitution.
- Core Azure context variables are supplied by azd automatically or set by the operator:
  - `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`, `AZURE_LOCATION` — set during `azd init` or `azd env set`.
  - `AZURE_ENV_NAME` — the azd environment name (e.g., `dev`, `prod`).
  - `AZURE_RESOURCE_GROUP` — the target resource group.
  - `AZURE_PRINCIPAL_ID` — the deployer's Entra object ID (used for RBAC bootstrapping).
- Additional variables (networking, service-specific config) are set via `azd env set` and referenced in `main.tfvars.json`:
  ```json
  {
    "azure": {
      "subscription_id": "${AZURE_SUBSCRIPTION_ID}",
      "tenant_id": "${AZURE_TENANT_ID}",
      "location": "${AZURE_LOCATION}",
      "environment_name": "${AZURE_ENV_NAME}",
      "resource_group_name": "${AZURE_RESOURCE_GROUP}",
      "principal_id": "${AZURE_PRINCIPAL_ID}"
    }
  }
  ```
- Terraform outputs (e.g., `APP_INSIGHTS_CONNECTION_STRING`, `BACKEND_APP_SERVICE_NAME`) are automatically captured by azd and made available as environment variables for subsequent `azd deploy` steps and service configuration.
- When adding new infrastructure that produces values needed by the application or by azd service mappings, add corresponding Terraform outputs with SCREAMING_SNAKE_CASE names.

## Provider configuration

- Require `terraform >= 1.9`.
- Pin providers with pessimistic constraints (`~>`) in both root and child modules.
- Commonly used providers: `azurerm`, `azurecaf`, `azuread`, `azapi`, `random`, `time`.
- Use `azapi` when a resource or property is not yet supported by `azurerm`.
- Configure `azurerm` with `storage_use_azuread = true` for Entra-based storage auth.
- Set `features` block: `purge_soft_delete_on_destroy = false`, `recover_soft_deleted_key_vaults = true`, `prevent_deletion_if_contains_resources = true`.
- Declare `backend "azurerm" {}` with no inline config; all values come from `provider.conf.json`.

## Resource group convention

- **Never create the resource group in Terraform.** Assume the resource group is pre-created and passed in via `var.azure.resource_group_name`.
- Reference the resource group with a data source:
  ```hcl
  data "azurerm_resource_group" "rg" {
    name = var.azure.resource_group_name
  }
  ```
- Use `data.azurerm_resource_group.rg.name` and `data.azurerm_resource_group.rg.location` throughout.

## Azure Verified Modules (AVM)

- **Prefer Azure Verified Modules** (`Azure/avm-res-*`) over raw `azurerm_*` resources whenever an AVM module exists for the resource type.
- Always set `enable_telemetry = false` on AVM modules.
- Use AVM-managed `role_assignments`, `diagnostic_settings`, and `private_endpoints` blocks rather than creating separate resources when the AVM module supports them.
- Set `private_endpoints_manage_dns_zone_group = false` when DNS is managed externally (e.g., by a platform team or Azure Policy).
- Pin AVM module versions with `~>` constraints. Look up the latest version on the Terraform registry before using.

## Variable design

- Group related configuration into **typed objects** with `optional()` defaults rather than using many flat variables. Examples:
  ```hcl
  variable "azure" {
    type = object({
      subscription_id     = string
      tenant_id           = string
      location            = string
      environment_name    = string
      resource_group_name = string
      principal_id        = string
    })
  }

  variable "sql" {
    type = object({
      sku_name    = optional(string, "S2")
      max_size_gb = optional(number, 250)
    })
    default = {}
  }
  ```
- Add `validation` blocks for variables that have constrained value sets or formats.
- Use a `tags` variable (`map(string)`, default `{}`) for user-provided tag overrides.
- Use a `naming` object variable with a short `project_name` (1–8 lowercase alphanumeric) for resource naming.
- Provide a `public_network_access_enabled` boolean (default `false`) at the root and propagate it to every module. This enables easy toggling between public-accessible (dev) and private-only (prod) deployments.
- When a new variable is needed, add it to both `variables.tf` (with type and default) and `main.tfvars.json` (with an azd `${PLACEHOLDER}` if operator-supplied, or a static default if fixed per project).

## Naming conventions

- Use the `azurecaf` provider (`azurecaf_name` resources) to generate Azure-compliant resource names.
- Derive a deterministic **resource token** from location, subscription, and resource group:
  ```hcl
  locals {
    resource_token_sha = base64encode(sha256("${var.azure.location}${data.azurerm_client_config.current.subscription_id}${var.azure.resource_group_name}"))
    resource_token     = substr(join("", regexall("[a-z0-9]", lower(local.resource_token_sha))), 0, 13)
  }
  ```
- Pass `resource_token` to all child modules and use it as the `name` argument in `azurecaf_name`.
- Always set `random_length = 0` and `clean_input = true` for deterministic, idempotent naming.
- Use `suffixes` to distinguish resources (e.g., `["identity"]`, `["law"]`, `["stor"]`, `["plan"]`).

## Tagging

- Build `default_tags` in a `locals` block by merging a base set with user-provided `var.tags`:
  ```hcl
  locals {
    default_tags = merge(
      {
        project     = "<repo-name>"
        environment = var.azure.environment_name
        managed_by  = "terraform"
      },
      var.tags
    )
  }
  ```
- Pass `local.default_tags` (or `tags = var.tags` inside modules) to every resource and module.
- For azd-deployed services, add `azd-env-name` and `azd-service-name` tags.

## Module structure

- Organize infrastructure into **child modules** under `infra/modules/<service>/`.
- Every module contains exactly four files:
  - `providers.tf` — required providers (version-pinned, matching root constraints).
  - `variables.tf` — input variables with types, descriptions, defaults, and validations.
  - `main.tf` — resource definitions.
  - `outputs.tf` — exported values for the root to consume.
- Common module input variables: `resource_token`, `resource_group_name`, `location`, `tags`, `public_network_access_enabled`.
- Use `count` on modules to make entire service blocks optional (e.g., `count = var.app_service.enabled ? 1 : 0`).
- Use `for_each` for sets of similar resources (e.g., multiple web apps).

## Networking and private endpoints

- Pass a `private_endpoint_subnet_id` variable into modules that need private endpoints.
- For AVM modules, use the built-in `private_endpoints` block.
- For direct `azurerm_private_endpoint` resources, always add `lifecycle { ignore_changes = [private_dns_zone_group] }` when DNS is managed externally.
- Name private endpoints as `{resource_name}-pe` and service connections as `{resource_name}-psc`.
- Default to private-only access (`public_network_access_enabled = false`). The root variable allows toggling for development.

## RBAC and identity

- Create a **user-assigned managed identity** for the application and pass its principal ID and client ID to modules that need data-plane access.
- Use AVM `role_assignments` blocks when available; otherwise use `azurerm_role_assignment` resources.
- Apply least-privilege roles (e.g., `Storage Blob Data Contributor` not `Owner`).
- For role assignments on resources with eventual-consistency identity propagation, use `time_sleep` resources as explicit dependencies.
- Use `uuidv5("dns", ...)` for deterministic role assignment `name` fields to ensure idempotency.
- Grant the deployer principal (`var.azure.principal_id`) admin-level access for initial setup (e.g., `Key Vault Secrets Officer`).
- Grant the app identity only data-level access (e.g., `Key Vault Secrets User`, `Storage Blob Data Contributor`).

## Outputs

- Export values that downstream azd deployment or application configuration needs (connection strings, URIs, resource names).
- Mark sensitive outputs with `sensitive = true` (connection strings, instrumentation keys, FQDNs).
- Use SCREAMING_SNAKE_CASE for output names that map to azd environment variables (e.g., `APP_INSIGHTS_CONNECTION_STRING`).
- Use ternary expressions for outputs from conditional modules: `var.x.enabled ? module.x[0].output : null`.
- Terraform outputs are automatically captured by azd after `azd provision` and become available as environment variables for `azd deploy` and application runtime.

## Monitoring and diagnostics

- Always deploy a **Log Analytics Workspace** and **Application Insights** resource (monitoring module).
- Pass the Log Analytics workspace ID to all modules so they can configure `diagnostic_settings`.
- In AVM modules, configure `diagnostic_settings` with appropriate `log_categories` and `metric_categories`.

## General best practices

- Use `data` sources to reference pre-existing resources (VNets, subnets, resource groups) — never recreate shared infrastructure.
- Prefer explicit `depends_on` over implicit dependency when ordering matters for identity propagation or RBAC.
- Keep modules focused on a single Azure service or logical grouping.
- Add descriptions to all variables, outputs, and complex locals.
- Use `lifecycle { ignore_changes = [...] }` sparingly and only for properties managed outside Terraform (e.g., DNS zone groups managed by Azure Policy).
