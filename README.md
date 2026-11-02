# Azure Cloud Engineering Labs

Hands-on Azure and Microsoft Entra ID projects I built after earning the **Microsoft Certified: Azure Fundamentals (AZ-900)** certification. Each lab has its own write-up covering **what I built, why I made each design decision, how I tested it, and the real problems I hit and fixed.**

**Live demo sites:** [Lab 4/5 site](https://stnatelab15658.z5.web.core.windows.net/) · [Lab 6 Terraform-built site](https://sttfsitebr6zq9.z5.web.core.windows.net/)
**Pipeline + IaC code:** [Malvoen/nate-site](https://github.com/Malvoen/nate-site)

---

## Labs

| # | Lab | What it demonstrates | Key tools |
|---|---|---|---|
| 1 | [**Group-based RBAC**](#lab-1-group-based-rbac-with-microsoft-entra-id) *(this page)* | Least-privilege access with security groups and resource-group scope | Azure RBAC, Entra ID, Azure CLI |
| 2 | [**Conditional Access + PIM**](Project2/) | MFA by policy, break-glass accounts, just-in-time admin access | Conditional Access, PIM, What If |
| 3 | [**SAML Single Sign-On**](Project3/) | Enterprise app SSO, app assignment, SAML token troubleshooting | Entra enterprise apps, SAML |
| 4 | [**Static Website on Azure Storage**](Project4/) | Serverless hosting from Blob Storage, built from the CLI | Azure Storage, Cloud Shell |
| 5 | [**CI/CD with GitHub Actions + OIDC**](Project5/) | Automatic deployments with **no stored secrets** | GitHub Actions, workload identity federation |
| 6 | [**Infrastructure as Code with Terraform**](Project6/) | Plan on pull request, apply on merge, remote state with Entra auth | Terraform, GitHub Actions |
| 7 | [**AD, File Share & Group-Based Drive Mapping**](Project7/) | Windows domain in Azure: AD DS, share/NTFS permissions, GPO drive maps | Azure VMs, AD DS, Group Policy |
| 8 | **Azure Monitor → PagerDuty** *(in progress)* | Alerting and on-call incident response | Azure Monitor, Action Groups, PagerDuty |

## Skills covered

| Area | Where |
|---|---|
| **Identity & access** (RBAC, Entra roles, groups, least privilege) | Labs 1, 2, 3, 7 |
| **Security** (MFA, Conditional Access, PIM, break-glass, NSG lockdown) | Labs 2, 7 |
| **Automation & DevOps** (CI/CD, OIDC, infrastructure as code) | Labs 5, 6 |
| **Networking** (VNets, subnets, static IPs, custom DNS, NSGs) | Lab 7 |
| **Windows Server** (AD DS, DNS, Group Policy, file shares) | Lab 7 |
| **Storage & hosting** | Labs 4, 5, 6 |
| **Cost management** (budget alerts, auto-shutdown, deallocation) | Labs 4, 7 |
| **Troubleshooting** (decoded SAML/OIDC tokens, AADSTS errors, pipeline failures) | Every lab, in a "Problems I hit" section |

---

## Lab 1: Group-Based RBAC with Microsoft Entra ID

A small-company scenario: an **Ops team** can build resources in one resource group, and an **Audit team** can only view them.

### What I built

| Component | Name | Purpose |
|---|---|---|
| Resource group | `rg-lab-identity` | The only place the Ops team is allowed to build |
| Users | Alex Ops, Sam Ops, Casey Ops, Jordan Audit | Test identities in Microsoft Entra ID |
| Security group | `GRP-Ops-Contributors` | Ops team members |
| Security group | `GRP-Auditors-Readers` | Audit team members |
| Role assignment | Contributor → `GRP-Ops-Contributors` | Scoped to `rg-lab-identity` only |
| Role assignment | Reader → `GRP-Auditors-Readers` | Scoped to `rg-lab-identity` only |

### Why I assigned roles to groups, not users

Permissions are attached to the **group**, and people get access by joining it. When someone joins or leaves a team, I change their group membership instead of editing role assignments on every resource. This scales, is easier to audit, and reduces the chance of leftover access when someone changes roles.

**Proof:** I added a new user (Casey) to `GRP-Ops-Contributors` using the Azure CLI. Casey could create resources in `rg-lab-identity` immediately, with no role assignment made for Casey directly.

```bash
GROUP_ID=$(az ad group show -g GRP-Ops-Contributors --query id -o tsv)
USER_ID=$(az ad user show --id casey.ops@<tenant>.onmicrosoft.com --query id -o tsv)
az ad group member add --group $GROUP_ID --member-id $USER_ID
```

### Contributor vs Owner

| | Contributor | Owner |
|---|---|---|
| Create, change, delete resources | ✅ | ✅ |
| Assign roles to other people | ❌ | ✅ |

I gave the Ops team **Contributor**, not Owner, following **least privilege**: they need to build resources, but they shouldn't be able to grant access to anyone else.

### Testing: did the permissions actually work?

| Test | User | Expected | Result |
|---|---|---|---|
| Create an NSG in `rg-lab-identity` | Alex (Contributor) | Allowed | ✅ Allowed |
| Create a new resource group | Alex (Contributor) | Denied | ✅ Denied |
| View `rg-lab-identity` | Jordan (Reader) | Allowed | ✅ Allowed |
| Create or delete a resource | Jordan (Reader) | Denied | ✅ Denied |

### Problems I hit and how I fixed them

**1. Test users were forced to set up MFA at first sign-in.**
New tenants have **Security Defaults** enabled, which requires every user to register for MFA. I disabled Security Defaults for this lab and replaced it in Lab 2 with a targeted Conditional Access MFA policy.

**2. "Resource provider not registered" when Alex created an NSG.**
The subscription had never registered `Microsoft.Network`. Registering a provider is a **subscription-level** action, and Alex's Contributor role only applies to the resource group, so the portal's automatic registration failed under his account. I registered the providers as the subscription Owner, and Alex's deployment then succeeded. This confirmed that his access really did stop at the resource group boundary.

**3. CLI user creation failed with an invalid domain.**
I'd left a placeholder domain in the command. A user principal name must use one of the tenant's verified domains.

### What I learned

- **Entra roles** (e.g., User Administrator) manage the *directory*; **Azure RBAC roles** (e.g., Contributor) manage *resources*.
- **Scope matters:** management group → subscription → resource group → resource. Access granted at one level flows down, never up.
- **Test as the user:** I signed in as each test account in a private browser window instead of only reading the role assignments.

### References

- [Assign Azure roles using the Azure portal](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-portal)
- [Manage Microsoft Entra groups](https://learn.microsoft.com/entra/fundamentals/how-to-manage-groups)
- [Security defaults in Microsoft Entra ID](https://learn.microsoft.com/entra/fundamentals/security-defaults)
- [Azure resource providers and types](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-providers-and-types)
