# GitHubActions_Learning

A hands-on learning repository demonstrating how to use **GitHub Actions** to automate **Terraform** infrastructure deployments on **Microsoft Azure** using **OpenID Connect (OIDC)** for passwordless authentication.

---

## 📁 Repository Structure

```
GitHubActions_Learning/
├── .github/
│   └── workflows/
│       └── terraform-cd.yml       # CI/CD pipeline for Terraform plan & apply
├── terraform_infra/
│   └── main.tf                    # Terraform configuration (Azure resources)
├── .gitignore                     # Terraform-specific ignore rules
└── README.md
```

---

## ⚙️ GitHub Actions Workflow

**File:** [`.github/workflows/terraform-cd.yml`](.github/workflows/terraform-cd.yml)

**Workflow name:** `Terraform Azure Infra Deploy`

### Triggers

| Event          | Branch |
|----------------|--------|
| `push`         | `main` |
| `pull_request` | `main` |

### Jobs

#### 1. `Terraform-Validate-Plan` — Security Scan & Plan

Runs on every push or pull request to `main`.

| Step | Action / Command | Description |
|------|-----------------|-------------|
| Checkout | `actions/checkout@v7` | Checks out the repository |
| Checkov Security Scan | `bridgecrewio/checkov-action@v12` | Scans `terraform_infra/` for security misconfigurations (`soft_fail: true`) |
| Azure Login | `azure/login@v3` (OIDC) | Authenticates to Azure using federated credentials |
| Setup Terraform | `hashicorp/setup-terraform@v2` | Installs Terraform CLI |
| `terraform init` | CLI | Initialises the working directory |
| `terraform validate` | CLI | Validates configuration syntax |
| `terraform plan` | CLI | Generates and saves `tfplan` execution plan |

#### 2. `Terraform-Deploy` — Infrastructure Apply

Runs **after** `Terraform-Validate-Plan` succeeds. Requires manual approval via a **`production` environment** gate.

| Step | Action / Command | Description |
|------|-----------------|-------------|
| Checkout | `actions/checkout@v7` | Checks out the repository |
| Azure Login | `azure/login@v3` (OIDC) | Authenticates to Azure using federated credentials |
| Setup Terraform | `hashicorp/setup-terraform@v2` | Installs Terraform CLI |
| `terraform init` | CLI | Initialises the working directory |
| `terraform apply` | CLI (`--auto-approve`) | Applies infrastructure changes |

---

## 🏗️ Terraform Infrastructure

**File:** [`terraform_infra/main.tf`](terraform_infra/main.tf)

**Provider:** `hashicorp/azurerm` v4.78.0

### Resources Provisioned

| Resource | Name | Location | Details |
|----------|------|----------|---------|
| `azurerm_resource_group` | `rg-dhondhu` | Central India | Root resource group |
| `azurerm_storage_account` | `messy007devinfra` | Central India | Standard LRS, tagged `environment=dev` |

### Remote Backend (Optional)

The `terraform_infra/main.tf` includes a commented-out Azure remote backend configuration for storing Terraform state in Azure Blob Storage:

```hcl
backend "azurerm" {
  resource_group_name  = "terraformBackend-rg"
  storage_account_name = "terraformstatemgrdev"
  container_name       = "tfstate"
  key                  = "dev.terraform.tfstate"
}
```

To enable remote state, uncomment this block and ensure the storage account exists before running `terraform init`.

---

## 🔐 Required GitHub Secrets

The workflow authenticates to Azure using **OIDC (federated identity)** — no client secrets are stored. Configure the following repository secrets:

| Secret Name | Description |
|-------------|-------------|
| `AZURE_CLIENT_ID` | Application (client) ID of the Azure AD App Registration |
| `AZURE_TENANT_ID` | Azure Active Directory Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Target Azure Subscription ID |

> **How to set up OIDC:** In your Azure AD App Registration, add a **Federated Credential** for GitHub Actions with the subject claim matching your repo and branch (e.g., `repo:<org>/<repo>:ref:refs/heads/main`).

---

## 🌍 GitHub Environment

The `Terraform-Deploy` job uses a GitHub **`production`** environment. Configure this environment in **Settings → Environments → production** to add:

- Required reviewers (manual approval gate before `apply`)
- Environment-specific secrets (optional)

---

## 🛡️ Security Scanning

[Checkov](https://www.checkov.io/) is integrated directly into the pipeline via `bridgecrewio/checkov-action`. It scans all Terraform files for:

- Security misconfigurations
- Compliance violations (CIS, NIST, SOC2, etc.)

`soft_fail: true` is set so the pipeline continues even if Checkov findings are detected, allowing you to review results without blocking deployments during learning.

---

## 🚀 Getting Started

### Prerequisites

- An Azure subscription with Owner or Contributor access
- An Azure AD App Registration with OIDC federated credentials configured for this repo
- GitHub repository secrets (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`) set
- A `production` environment created in GitHub repository settings

### Local Development

```bash
# Navigate to the Terraform directory
cd terraform_infra

# Authenticate with Azure CLI
az login

# Initialise Terraform
terraform init

# Preview changes
terraform plan

# Apply changes
terraform apply
```

---

## 📝 .gitignore

The `.gitignore` is configured for Terraform projects and excludes:

- `.terraform/` — local provider cache
- `*.tfstate`, `*.tfstate.*` — state files (contain sensitive data)
- `*.tfvars`, `*.tfvars.json` — variable files (may contain secrets)
- `crash.log` — Terraform crash logs
- `.terraform.tfstate.lock.info` — lock files
- `.terraformrc`, `terraform.rc` — CLI config files

---

## 📚 Learning Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Terraform AzureRM Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Azure Login via OIDC](https://github.com/Azure/login#login-with-openid-connect-oidc-recommended)
- [Checkov – Infrastructure Security Scanner](https://www.checkov.io/)
- [Configuring GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)