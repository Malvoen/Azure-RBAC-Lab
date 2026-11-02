# Azure Lab 6: Infrastructure as Code with Terraform + GitHub Actions

Part of my Azure lab series: [Lab 1: RBAC](../) · [Lab 2: Conditional Access + PIM](../Project2/) · [Lab 3: SAML SSO](../Project3/) · [Lab 4: Static Website](../Project4/) · [Lab 5: CI/CD + OIDC](../Project5/)

**Code repo:** https://github.com/Malvoen/nate-site (`infra/main.tf`, `.github/workflows/terraform.yml`)

In Lab 4 I built a static website by hand. In this lab, **the same kind of infrastructure is defined as code in Terraform** and deployed by a GitHub Actions pipeline:

- **Pull request** → `terraform plan` runs so I can review exactly what will change
- **Merge to `main`** → `terraform apply` makes the change in Azure
- **No secrets stored:** the pipeline signs in with OIDC, and Terraform state is accessed with Entra ID instead of storage keys

## Architecture

```
GitHub repo (nate-site)
├── site/                  ← website files (index.html, 404.html, test.html)
├── infra/main.tf          ← Terraform: what Azure resources should exist
└── .github/workflows/
    ├── deploy.yml         ← Lab 5 pipeline (uploads site to the Lab 4 account)
    └── terraform.yml      ← Lab 6 pipeline (plan on PR, apply on merge)

          │ OIDC token (no password)
          ▼
Microsoft Entra ID ── app registration: gh-actions-terraform
          │            federated credentials: main branch + pull requests
          ▼
Azure
├── rg-tfstate / sttfstatenate01 / tfstate   ← Terraform state (Storage Blob Data Contributor)
└── rg-lab-tf                                 ← everything Terraform builds (Contributor, this RG only)
    └── sttfsite<random>  (StorageV2, LRS, static website, tagged managed_by=terraform)
        └── $web/ index.html, 404.html, test.html
```

## Design decisions

| Decision | Why |
|---|---|
| **Separate pipeline identity** (`gh-actions-terraform`) from the Lab 5 deploy identity | Terraform needs to *create resources*; the website pipeline only uploads files. Each gets only the access it needs. |
| **Contributor on one resource group**, not the subscription | Limits the blast radius: the pipeline can't touch anything outside `rg-lab-tf`. |
| **Resource group created by an admin**, referenced in code with a `data` block | Admins control *where* things go; the pipeline controls *what's inside*. |
| **Remote state in Azure Storage** with `use_oidc` + `use_azuread_auth` | State is shared, locked during runs (no two runs collide), and accessed with Entra ID instead of storage account keys. |
| **Blob versioning + soft delete** on the state account | The state file is Terraform's memory of what it built. Losing or corrupting it is painful, so previous versions are kept. |
| **`resource_provider_registrations = "none"`** | The pipeline has no subscription-level rights, so it can't register providers. They were registered by an admin beforehand. |
| **Random suffix** on the storage account name | Storage account names must be globally unique. |
| **`content_md5` on each blob** | Terraform detects when a file's contents change and re-uploads it. |
| **Federated credentials for `main` *and* pull requests** | PR runs can plan (read-only in practice); only `main` applies. |

## The pipeline (`terraform.yml`)

```yaml
on:
  pull_request:  { branches: [main], paths: ['infra/**', 'site/**'] }
  push:          { branches: [main], paths: ['infra/**', 'site/**'] }
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

env:
  ARM_CLIENT_ID:       ${{ secrets.TF_CLIENT_ID }}
  ARM_TENANT_ID:       ${{ secrets.TF_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.TF_SUBSCRIPTION_ID }}
  ARM_USE_OIDC: "true"

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: infra } }
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform validate
      - run: terraform plan -input=false -out=tfplan
      - name: Terraform apply
        if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
        run: terraform apply -input=false -auto-approve tfplan
```

## What I tested

| Run | Trigger | Result |
|---|---|---|
| #1 Create main.tf | Push to `main` | Plan + apply: **4 added** (random suffix, storage account, index.html, 404.html) |
| #2 Update index.html | Pull request | **Plan only.** Apply step skipped. Plan: `1 to add, 0 to change, 1 to destroy` |
| #3 Merge PR #2 | Push to `main` | Apply: index.html replaced, change live on the site |
| PR #3 Add test page | Pull request | First run **failed validate** (see below), then plan: `1 to add` after the fix |
| Merge PR #3 | Push to `main` | Apply: `test.html` live at `/test.html` |

## Problems I hit and how I fixed them

**1. `main.tf` saved in the wrong folder.**
I created the file while browsing `.github/workflows`, so it was saved as `.github/workflows/infra/main.tf`, and the workflow (which watches `infra/**`) never triggered. **Fix:** closed the PR without merging and recreated the file from the repo root as `infra/main.tf`.
**Lesson:** GitHub's web editor creates new files in whatever folder you're currently viewing. Check the path before committing.

**2. "ResourceNotFound" when opening the site.**
I opened the **blob endpoint** (`…blob.core.windows.net/$web/index.html`) instead of the **static website endpoint** (`…web.core.windows.net/`). Anonymous blob access is disabled (correctly), so the blob URL refuses public requests. **Lesson:** a storage account has separate endpoints. The `$web` container is public *only* through the web endpoint.

**3. Plan said "replace," not "change."**
Editing `index.html` produced `-/+ must be replaced` because `content_md5` changed. Terraform can't edit a blob's contents in place, so it deletes and re-uploads it. Harmless for a web page, but **this is exactly why plans get reviewed**: a "replace" on a database or storage account would mean data loss.

**4. Pipeline failed: `filemd5 … no such file or directory`.**
I'd named the new page `test,html` (comma instead of dot), so `main.tf` pointed at a file that didn't exist. `terraform validate` failed the run in 16 seconds, **before plan or apply, with no impact on Azure**. **Fix:** renamed the file on the same branch; the PR re-ran automatically and passed.
**Lesson:** validate → plan → apply ordering catches mistakes early and cheaply.

## Known limitations / next steps

- **Deprecation warning:** the `static_website` block inside `azurerm_storage_account` is deprecated in favor of the separate `azurerm_storage_account_static_website` resource and will be removed in AzureRM provider v5.0. The provider version is pinned to `~> 4.0`, so it works today. Migrating is a planned improvement.
- **Files listed one by one:** each page needs its own `azurerm_storage_blob` block. Next step: loop over the folder with `for_each = fileset(...)` so new pages deploy automatically.
- **Action versions:** GitHub flagged `actions/checkout@v4` and `hashicorp/setup-terraform@v3` for Node.js 20 deprecation. Bump to newer releases.
- **Approval gate:** add a GitHub *environment* with required reviewers so apply waits for a manual approval, even on `main`.

## Key concepts

- **Infrastructure as code:** infrastructure is reviewed, versioned, and repeatable like application code. If it's not in the code, it doesn't exist.
- **Plan before apply:** you always see exactly what will be created, changed, replaced, or destroyed before anything happens.
- **State:** Terraform's record of what it manages. Stored remotely, locked, versioned, and protected.
- **Least privilege for automation:** the pipeline is a security principal with narrowly scoped roles, the same principles as for human users in Labs 1–2.

## References
- [Terraform azurerm backend](https://developer.hashicorp.com/terraform/language/backend/azurerm)
- [Modernizing Terraform Pipelines on Azure: OIDC Federation for GitHub Actions](https://techcommunity.microsoft.com/t5/azure-infrastructure-blog/modernizing-terraform-pipelines-on-azure-oidc-federation-for/ba-p/4516620)
- [Use GitHub Actions to deploy a static website in Azure Storage](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-static-site-github-actions)
