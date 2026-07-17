# 03 — Infrastructure as Code with Terraform

## Why this chapter matters

The resume's broader skill list includes "Terraform Scripts" alongside Azure DevOps — the pipeline
in chapter 02 automates *code* deployment, but something has to define and provision the Azure
resources that code runs on (the App Service, the Key Vault, the database) in a way that's
repeatable across dev/staging/prod and across client environments. That's what Terraform is for.
This chapter covers Terraform's core model and walks through a worked example provisioning an Azure
App Service + Key Vault — the infrastructure shape underneath both the chatbot (course 01) and
document-uploader (course 03) services.

## Why Infrastructure as Code, Specifically

Before Terraform's mechanics, it's worth being able to answer "why not just click around the Azure
Portal?" clearly, because that's the real interview question underneath "why IaC":

- **Reproducibility.** A consulting engagement needs the same App Service + Key Vault + networking
  shape recreated for dev, staging, and prod — and again for the next client. Clicking through a
  portal three times introduces drift by definition; a `.tf` file applied three times with different
  variables produces identical infrastructure. This is literally how the same module stood up
  separate, isolated App Service environments for HSBC and Bank of America (see `00-README.md`) —
  one module, two sets of variables, zero drift between them.
- **Review and audit trail.** Infrastructure changes go through the same pull-request review as code
  changes when they're expressed as text in Git. "Who opened port 22 to the internet and when" is a
  `git log` and PR-diff question instead of an Azure Activity Log archaeology exercise.
- **Disaster recovery.** If an environment is deleted or corrupted, `terraform apply` against the
  existing state and config rebuilds it, instead of a human trying to remember the twelve settings
  that made the original App Service work.
- **Consistency between environments.** Because dev/staging/prod are the same Terraform module
  invoked with different variables, "it worked in staging but not prod" is far less likely to be
  caused by an infrastructure difference nobody noticed.

## Core Concepts: Providers, Resources, State

**Providers** are plugins that let Terraform talk to a specific platform's API — `azurerm` for Azure,
`aws` for AWS, and so on. A provider block configures which platform (and often which
subscription/region) subsequent resources target.

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.90"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.client_subscription_id
}
```

**Resources** are the individual infrastructure objects you want to exist — an App Service Plan, a
Web App, a Key Vault, a storage account. Each resource block declares its *desired* configuration;
Terraform's job is to figure out what API calls are needed to make reality match that declaration.

```hcl
resource "azurerm_service_plan" "uploader_plan" {
  name                = "asp-document-uploader-${var.environment}"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  os_type             = "Linux"
  sku_name            = "B1"
}
```

**State** is Terraform's record of what it believes actually exists — a JSON file (`terraform.tfstate`)
mapping each resource block in your config to the real object it created, along with all the
attributes that object currently has. State is what makes Terraform *declarative* rather than
*imperative*: you never write "create a web app," you write "a web app with these properties should
exist," and Terraform diffs that desired state against its recorded state to compute the minimal set
of changes.

Critically, state should never be a local file in a consulting context — it must live in **remote,
shared, locked storage** (an Azure Storage Account container is the standard choice) so that:

1. Every teammate and every pipeline run operates against the same source of truth for "what exists."
2. **State locking** prevents two concurrent `terraform apply` runs from racing each other and
   corrupting the state file or double-provisioning resources — Terraform acquires a lock on the
   remote backend before any write, and a second concurrent run simply fails fast with a "state is
   locked" error instead of proceeding.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "tfstateclientdevops"
    container_name       = "tfstate"
    key                  = "document-uploader.dev.tfstate"
  }
}
```

## Plan and Apply: The Two-Phase Workflow

Terraform's core workflow is deliberately two-phase, and understanding why is a common interview
probe:

- **`terraform plan`** computes and displays the diff between desired config and current state
  *without changing anything* — "this apply would add 2 resources, change 1, destroy 0." This is the
  step that belongs in a pipeline *before* any approval gate, so a human (or an automated policy
  check) can review exactly what's about to change to real infrastructure before it happens.
- **`terraform apply`** executes the plan, calling the provider's API to create/update/destroy
  resources and updating state to match.

In a CI/CD pipeline, `plan` typically runs on every PR (so reviewers see the infrastructure diff
alongside the code diff) and `apply` runs after merge and after an approval gate — mirroring the
build-then-gated-deploy pattern from chapter 02, just for infrastructure instead of application code.

## Modules: Reusing Infrastructure Patterns

A **module** is a reusable, parameterized bundle of resources — the Terraform equivalent of the
pipeline templates from chapter 02. Instead of writing the App Service + Key Vault + networking
pattern from scratch for every client and every environment, you write it once as a module and invoke
it with different variables:

```hcl
module "uploader_service_infra" {
  source          = "../modules/azure-webapp-with-keyvault"
  environment     = "dev"
  app_name        = "document-uploader"
  resource_group  = azurerm_resource_group.rg.name
  location        = "eastus"
  sku_name        = "B1"
}

module "chatbot_service_infra" {
  source          = "../modules/azure-webapp-with-keyvault"
  environment     = "dev"
  app_name        = "chatbot-assistant"
  resource_group  = azurerm_resource_group.rg.name
  location        = "eastus"
  sku_name        = "P1v3"   # chatbot needs more headroom for concurrent LLM calls
}
```

This is the realistic way a shared "App Service + Key Vault" pattern would be reused across the
chatbot and document-uploader services in this curriculum — one module, invoked twice with different
sizing.

## Worked Example: Azure App Service + Key Vault

A minimal but complete example provisioning the infrastructure shape both services need — a Linux App
Service running a container, and a Key Vault holding its secrets, with the App Service granted
access via a managed identity (no credentials in app settings):

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "rg-${var.app_name}-${var.environment}"
  location = var.location
}

resource "azurerm_service_plan" "plan" {
  name                = "asp-${var.app_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  os_type             = "Linux"
  sku_name            = var.sku_name
}

resource "azurerm_linux_web_app" "app" {
  name                = "${var.app_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  service_plan_id     = azurerm_service_plan.plan.id

  site_config {
    application_stack {
      docker_image_name   = "${var.app_name}:latest"
      docker_registry_url = "https://${var.acr_login_server}"
    }
  }

  identity {
    type = "SystemAssigned"   # lets the app authenticate to Key Vault without a stored secret
  }

  app_settings = {
    KEY_VAULT_URI = azurerm_key_vault.kv.vault_uri
  }
}

resource "azurerm_key_vault" "kv" {
  name                = "kv-${var.app_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  tenant_id           = var.tenant_id
  sku_name            = "standard"
}

resource "azurerm_key_vault_access_policy" "app_access" {
  key_vault_id = azurerm_key_vault.kv.id
  tenant_id    = var.tenant_id
  object_id    = azurerm_linux_web_app.app.identity[0].principal_id

  secret_permissions = ["Get", "List"]
}
```

Running `terraform plan` against this config for a brand-new environment would report something like
"5 to add, 0 to change, 0 to destroy" — App Service Plan, Web App, Key Vault, access policy, and the
implicit resource group dependency. That plan-output shape (add/change/destroy counts) is exactly
what notebook `02_terraform_basics_demo.ipynb` parses programmatically, because reading a plan summary
correctly — not just running `apply` and hoping — is the actual skill being tested when someone asks
"walk me through how you'd review a Terraform plan before approving it."

## Interview Framing

If asked "why Terraform and not ARM templates / Bicep," a fair answer for a multi-client consultancy
is: Terraform is cloud-agnostic (useful if any engagement touches AWS/GCP too), has a larger module
ecosystem, and its state-plus-plan model gives an explicit, reviewable diff step that fits naturally
into a PR-gated pipeline — while acknowledging Bicep is a perfectly reasonable Azure-only alternative
with tighter native integration.
