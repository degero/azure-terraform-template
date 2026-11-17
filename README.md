# Terraform Azure Github Template

[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/OWNER/REPOSITORY/badge)](https://scorecard.dev/viewer/?uri=github.com/OWNER/REPOSITORY)
[![CI](https://github.com/OWNER/REPOSITORY/actions/workflows/main.yml/badge.svg)](https://github.com/OWNER/REPOSITORY/actions/workflows/main.yml)
[![CodeQL](https://github.com/OWNER/REPOSITORY/actions/workflows/codeql.yml/badge.svg)](https://github.com/OWNER/REPOSITORY/actions/workflows/codeql.yml)
[![CSpell check](https://github.com/OWNER/REPOSITORY/actions/workflows/cspell.yml/badge.svg)](https://github.com/OWNER/REPOSITORY/actions/workflows/cspell.yml)
[![Lint](https://github.com/OWNER/REPOSITORY/actions/workflows/lint.yml/badge.svg)](https://github.com/OWNER/REPOSITORY/actions/workflows/lint.yml)
[![Terraform Drift Detection](https://github.com/OWNER/REPOSITORY/actions/workflows/tf-drift.yml/badge.svg)](https://github.com/OWNER/REPOSITORY/actions/workflows/tf-drift.yml)

After creating a repository from this template, replace `OWNER/REPOSITORY` in these badge URLs with its GitHub owner and repository name. GitHub does not automatically rewrite README links when generating a repository from a template.

## Description

An opinionated template for developing Terraform with Github running multi env infra deployments in Azure.

This template is focused on the terraform and github cicd it is not opinionated on the Azure hosted terraform management infrastructure as this can vary greatly by organisation. Scripts are included to get a baseline running with blast radius by "Resource group per environment".

## Key components

- Isolated /environments/env-name root terraform modules to reduce blast radius / increased flexibility
- Isolated terraform federated credentials + 2 UMAI per env (plan,apply) + RBAC to blob storage by environment (tfstate, tfplans containers)
- TFPlan artifacts uploaded to blob storage (retention 7 days)
- Named `/modules` files for easy location in VSCode (instead of lots of main.tf files)
- Github Workflows with: Linting (tflint,actionlint,prettier), Formatting (terraform fmt), Sec checks (checkov, Gitleaks secret scanning with PR comments and SARIF artifact), Github Environments for workflow approval, Dependabot (terraform, github actions), Doco generation (terraform-docs), Terraform Validate, Drift Detection (automated scheduled checks + issue management)
- Pre-push githook to validate/format terraform, lint (prettier,actionlint,powershell,shellscript) and checkov (if installed) locally
- VSCode settings for format on save: markdown / yaml / json(c), powershell
- Prettier rules for markdown / yaml / json(c)
- Scripts folder for dev convenience (tfinit) and admin setup Azure terraform infra / github settings.

## Folder structure

azure-terraform-template
│
├── environments/ - each target azure environment has its own subfolder to reduce blast radius
├── modules/ - Terraform modules used by environments
└── scripts/ - Az CLI scripts to setup terraform remote state store and github actions access to azure

## Terraform AzureRM authentication

use_azuread_auth = true
use_oidc = true

- OpenID Connect / Workload identity federation (Recommended by Hashicorp)
- User Assigned Managed Identity with Federated Credentials (Recommended by Hashicorp)

## Requirements

## Administrators

- Subscription level contributor access
- AZ CLI logged in with target subscription set

### Developers

- Role 'Storage Blob Data Contributor' on the DEV tfstate blob storage
- Role 'Contributor' on the lower-environment resource group (for example, `dev`) when testing Terraform changes from a local machine
- TerraformLinters.tflint
- checkov

## Environment setup

### Administrators

1. Copy [scripts/administrators/.env.example](scripts/administrators/.env.example) to `scripts/administrators/.env` and update it with your Azure environment and target GitHub repositories.
1. Set up Terraform state storage and user-assigned managed identities with [scripts/administrators/setup-azure-tfstate-store.sh](scripts/administrators/setup-azure-tfstate-store.sh).
1. Set up GitHub environments, OIDC federated credentials, and environment variables with [scripts/administrators/setup-github-environments.sh](scripts/administrators/setup-github-environments.sh).
1. Optionally customize the OIDC subject claim with [scripts/administrators/setup-github-oidc-subject.sh](scripts/administrators/setup-github-oidc-subject.sh); see the security trade-off described in [scripts/administrators/README.md](scripts/administrators/README.md).
1. Update each GitHub environment variable `TF_PLAN_STORAGE_ACCOUNT` with the appropriate storage account name.
1. If enabling the docs workflow, add repository-level GitHub Actions secrets `DOCO_DEPLOY_KEY` (the private SSH deploy key used to push generated docs) and `INFRACOST_API_KEY`. The docs workflow does not target a GitHub Environment, so these are repository secrets rather than environment secrets.
1. If you want PRs to apply Terraform to the lowest (first) environment, set a repo variable `PR_TF_APPLY` to `true`.

### Developers

**Prerequisites:**

- Terraform
- Azure CLI (logged in to tenant with remote storage)

(Optional for githook prepush)

- tflint
- Checkov
- PowerShell (pwsh) + PSScriptAnalyzer module
- ShellCheck
- actionlint
- Node.js (for npx → cspell, Prettier)

**Azure Permissions:**

If you have appropriate access, assign yourself access to the dev tfstate storage:

```bash
az ad signed-in-user show --query id -o tsv

az role assignment create \
  --assignee your-id \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>/blobServices/default/containers/tfstate"
```

NOTE: If state storage is in a different subscription or tenant you will need access to these.

**Development workflow:**

```bash
(optional: the prepush check also done in CI) git config core.hooksPath .githooks

If you wish to run local plan runs on higher envs, set an env var "TF_PLUGIN_CACHE_DIR" for a common cache location

cd environments/dev (and other envs needing changes)
../../scripts/tf-init.sh/ps1 (convenience item as first init needs backend.hcl file)

terraform workspace new <issuenumber> (so workspace is unique)

# Make changes

terraform plan -out="tfplan"
terraform apply tfplan

# before committing changes run in repo root:

# after PR complete
terraform workspace select default
terraform workspace delete <issuenumber>

```

It is recommended to use a workspace for local dev to not interfere with the tfstate used by CI or other devs.

It is recommended to setup a TF_PLUGIN_CACHE_DIR env var.

## Admin Setup

### Github Environments, Branch Protection and TF Service Principal guidelines

#### GitHub environments and variables

Set the repository variable `TF_ENVIRONMENTS` to a JSON array of the environment directory names, for example `["dev"]`. Each name must match a directory under `environments/`; start with `["dev"]` and add environments as you create them.

For every configured environment, the workflows use two GitHub Environments:

- `<environment>-plan` is used by Terraform plan and drift-detection jobs. It normally does not require reviewers so plans can run automatically.
- `<environment>-apply` is used by Terraform apply jobs. Configure required reviewers and any other deployment protection rules for environments where an approval is required.

The GitHub setup script creates both environments and configures these environment variables on each:

- `AZURE_CLIENT_ID`: client ID of that environment's user-assigned managed identity.
- `AZURE_TENANT_ID`: Microsoft Entra tenant containing the deployment identity.
- `AZURE_SUBSCRIPTION_ID`: subscription targeted by Terraform deployments.
- `TF_PLAN_STORAGE_ACCOUNT`: storage account used to store Terraform plans.

The plan and apply environments use their corresponding identities. The workflows authenticate to Azure using GitHub OIDC; do not create or store a client secret for these identities. If you use the docs workflow in a generated repository, configure the repository-level `DOCO_DEPLOY_KEY` and `INFRACOST_API_KEY` secrets described above.

#### Branch protection

Protect `main` separately from GitHub Environment approvals. Require pull requests and the CI checks appropriate for your repository, and restrict direct pushes and force pushes according to your team's policy. Environment reviewers gate deployments; branch protection controls changes to the branch and neither setting replaces the other.

#### Terraform identities and Azure permissions

The bootstrap scripts create separate plan and apply user-assigned managed identities for each environment and federate them to the repository's GitHub Actions workflows. These managed identities have service principals in Microsoft Entra ID and authenticate through OIDC federation; no long-lived Azure credentials are stored in GitHub.

The bootstrap script grants `Contributor` at the deployment subscription and `Storage Blob Data Contributor` on the environment's state storage account to both identities as a simple starting point. This subscription-wide Contributor assignment is intentionally broad and is not recommended as a production permission model. Before production use, scope deployment permissions to the resources or resource group the environment manages, and grant plan and apply identities only the permissions each job needs. Keep state storage access limited to the required identities and developers.

For production environments, review the [AzureRM backend security guidance](https://learn.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage). Consider private networking for storage, separate state and plan storage, and separate subscriptions for deployment environments and Terraform management infrastructure. See [Azure subscription democratization](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-principles#subscription-democratization) for guidance on subscription boundaries.

The bootstrap steps and security trade-offs are documented in [scripts/administrators/README.md](scripts/administrators/README.md).

#### Release Please

In GitHub, set **Settings → General → Pull Requests → Default commit message** to **Pull request title** so Release Please can compose release notes from PR titles.

modules/ ← atomic modules only (one resource type each + unit test file)
├── compute/
│ ├── functionapp/functionapp.tf
│ └── vm/vm.tf
├── storage/
│ └── account/storageaccount.tf
├── networking/
│ ├── vnet/vnet.tf
│ └── subnet/subnet.tf
├── secrets/
│ └── keyvault/keyvault.tf

modulegroups/ ← composite modules (patterns of primitives)
├── function-app-order-process/
└─── function-app-order-process.tf # calls modules/compute/functionapp + modules/storage/account

environments/ ←
├── dev(test,stage,prod,etc)/
│ ├── terraform.tfvars
│ ├── backend.hcl
│ └── main.tf # root module for ENV calls modules/, and compositions/ etc.

tests/ (Add cross module / integration tests here)

### Azure tooling

aztfexport
https://learn.microsoft.com/en-us/azure/developer/terraform/azure-export-for-terraform/export-terraform-overview

### Linting and Formatters

If changing .sh/.ps1 scripts run:

```
shellcheck -S warning $(find . -type f -name "*.sh" -not -path "*/.terraform/*")
Invoke-ScriptAnalyzer -Path . -Recurse -Severity Warning
```

## Troubleshooting

### Local dev

If there are any issues with githooks, set the permissions:

```bash
chmod +x .githooks/pre-push
chmod +x ./scripts/prepush.sh
```

## Links

https://github.com/terraform-linters/tflint

https://github.com/bridgecrewio/checkov

https://github.com/terraform-docs/gh-actions

https://github.com/Azure-Samples/terraform-github-actions

https://github.com/azure-samples/github-terraform-oidc-ci-cd

https://www.terraform-best-practices.com/examples/terraform/medium-size-infrastructure

https://developer.hashicorp.com/terraform/language/backend/azurerm

https://github.com/mermaid-js/mermaid

https://github.com/Pwd9000-ML/Azure-Terraform-Deployments
