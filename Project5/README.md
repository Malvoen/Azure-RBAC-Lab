# Azure Lab 5: CI/CD with GitHub Actions + Passwordless (OIDC) Deployment

Part of my Azure lab series: [Lab 1: RBAC](../) · [Lab 2: Conditional Access + PIM](../Project2/) · [Lab 3: SAML SSO](../Project3/) · [Lab 4: Static Website](../Project4/)

**Pipeline repo:** https://github.com/Malvoen/nate-site
**Live site:** https://stnatelab15658.z5.web.core.windows.net/

The static website from Lab 4 now deploys **automatically**: when I commit a change to the site files on GitHub, a GitHub Actions workflow publishes it to Azure Storage. **No passwords or keys are stored in GitHub.** The pipeline signs in to Azure with short-lived OpenID Connect (OIDC) tokens.

## How it works

```
Commit to main (site/**)
        │
        ▼
GitHub Actions runner
  1. Checks out the repo
  2. Requests a signed OIDC token from GitHub
        │   "I am repo Malvoen/nate-site, branch main"
        ▼
Microsoft Entra ID
  3. Checks the token against a federated credential on the app registration
  4. Issues a short-lived Azure access token
        │
        ▼
Azure Storage (stnatelab15658)
  5. Pipeline uploads site/ to the $web container
     (allowed only by its Storage Blob Data Contributor role)
```

## What I built

| Component | Value | Purpose |
|---|---|---|
| GitHub repo | `Malvoen/nate-site` | Website source (`site/index.html`, `site/404.html`) + workflow |
| Workflow | `.github/workflows/deploy.yml` | Runs on push to `main` when `site/**` changes, plus a manual "Run workflow" button |
| Entra app registration | `gh-actions-nate-site` | The pipeline's own identity, separate from any human account |
| Federated credential | Trusts GitHub's OIDC issuer, **only** for this repo's `main` branch | No client secret to store, leak, or rotate |
| Role assignment | **Storage Blob Data Contributor**, scoped to **one storage account** | Least privilege: the pipeline can upload website files and nothing else |
| GitHub secrets | `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` | Identifiers only, not passwords |

## The workflow

```yaml
name: Deploy website to Azure Storage

on:
  push:
    branches: [main]
    paths: ['site/**']
  workflow_dispatch:

permissions:
  id-token: write   # lets the job request an OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Azure (OIDC, no stored password)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          allow-no-subscriptions: true   # pipeline has only a data role, no subscription role

      - name: Upload site to $web container
        run: |
          az storage blob upload-batch \
            --account-name stnatelab15658 \
            --source site \
            --destination '$web' \
            --auth-mode login \
            --overwrite
```

## Testing

| Test | Result |
|---|---|
| Manual run (Actions → Run workflow) | ✅ Succeeded |
| Edit `site/index.html` and commit to `main` | ✅ Workflow ran automatically, change was live on the site |

## Problems I hit and how I fixed them

**1. Login failed: `AADSTS700213: No matching federated identity record found`.**
My federated credential expected this subject:
```
repo:Malvoen/nate-site:ref:refs/heads/main
```
The workflow log showed GitHub actually sent:
```
repo:Malvoen@<owner-id>/nate-site@<repo-id>:ref:refs/heads/main
```
GitHub's token includes **immutable numeric IDs** for the owner and repo. **Fix:** I created a federated credential matching the exact subject from the log. Microsoft now recommends this format: the IDs never change, so someone can't take over the trust by recreating a repo with the same *name*.
**Lesson:** for token-based auth errors, compare the claim the token actually contains with what the trust expects, and don't guess.

**2. Accidentally deleted the working credential while cleaning up.**
While removing the old, unused credential, I also deleted the working one, so the pipeline could no longer log in. **Fix:** I rebuilt it in the Entra portal (App registrations → Certificates & secrets → Federated credentials → *GitHub Actions deploying Azure resources*) with the organization ID, repository ID, and branch. I confirmed the generated subject matched the log, then re-ran the workflow successfully.
**Lesson:** double-check before deleting, and always verify with a test run after any change to authentication.

## Key concepts

- **CI/CD:** changes deploy automatically and consistently from source control, without manual uploads.
- **Workload identity federation (OIDC):** the pipeline gets short-lived tokens from Entra ID instead of storing a long-lived client secret. Nothing to leak, nothing to rotate.
- **A pipeline is a security principal too.** It gets its own identity and a narrowly scoped role (one storage account, data access only), the same least-privilege approach as for human users in Labs 1–2.
- **Scoped trust:** the federated credential only accepts tokens from one repo and one branch. A fork or another branch can't deploy.

## References
- [Use GitHub Actions to deploy a static website in Azure Storage](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-static-site-github-actions)
- [Workload identity federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)
- [Azure Login action](https://github.com/marketplace/actions/azure-login)
