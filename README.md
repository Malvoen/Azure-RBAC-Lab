# Azure RBAC Lab: Group-Based Access Control with Microsoft Entra ID

A hands-on lab simulating a small company where an **Ops team** can build resources in one resource group and an **Audit team** can only view them. Built in a personal Azure tenant after passing AZ-900.

## Labs in this repo
- **Lab 1: Group-based RBAC** (this page)
- **[Lab 2: Conditional Access MFA + Privileged Identity Management](project-2/)**

## What I built

| Component | Name | Purpose |
|---|---|---|
| Resource group | `rg-lab-identity` | The only place the Ops team is allowed to build |
| Users | Alex Ops, Sam Ops, Casey Ops, Jordan Audit | Test identities in Microsoft Entra ID |
| Security group | `GRP-Ops-Contributors` | Ops team members |
| Security group | `GRP-Auditors-Readers` | Audit team members |
| Role assignment | Contributor → `GRP-Ops-Contributors` | Scoped to `rg-lab-identity` only |
| Role assignment | Reader → `GRP-Auditors-Readers` | Scoped to `rg-lab-identity` only |

## Why I assigned roles to groups, not users

Permissions are attached to the **group**, and people get access by joining it. When someone joins or leaves a team, I change their group membership instead of editing role assignments on every resource. This scales, is easier to audit, and reduces the chance of leftover access when someone changes roles.

**Proof:** I added a new user (Casey) to `GRP-Ops-Contributors` using the Azure CLI. Casey could create resources in `rg-lab-identity` immediately, with no role assignment made for Casey directly.

```bash
GROUP_ID=$(az ad group show -g GRP-Ops-Contributors --query id -o tsv)
USER_ID=$(az ad user show --id casey.ops@<tenant>.onmicrosoft.com --query id -o tsv)
az ad group member add --group $GROUP_ID --member-id $USER_ID
```

## Contributor vs Owner

| | Contributor | Owner |
|---|---|---|
| Create, change, delete resources | ✅ | ✅ |
| Assign roles to other people | ❌ | ✅ |

I gave the Ops team **Contributor**, not Owner, following **least privilege**: they need to build resources, but they shouldn't be able to grant access to anyone else. In the Azure portal, Contributor and Owner are listed under the **Privileged administrator roles** tab, not the default Job function roles tab.

## Testing: did the permissions actually work?

| Test | User | Expected | Result |
|---|---|---|---|
| Create an NSG in `rg-lab-identity` | Alex (Contributor) | Allowed | ✅ Allowed |
| Create a new resource group | Alex (Contributor) | Denied | ✅ Denied |
| View `rg-lab-identity` | Jordan (Reader) | Allowed | ✅ Allowed |
| Create or delete a resource | Jordan (Reader) | Denied | ✅ Denied |

## Problems I hit and how I fixed them

**1. Test users were forced to set up MFA at first sign-in.**
New tenants have **Security Defaults** enabled, which requires every user to register for MFA, and the 14-day "skip" grace period no longer exists. I disabled Security Defaults for this lab only; in a later project I replace it with a Conditional Access MFA policy, which is more flexible.

**2. Alex got a "resource provider not registered" error when creating an NSG.**
The subscription had never registered `Microsoft.Network`. Registering a provider is a **subscription-level** action, and Alex's Contributor role only applies to the resource group, so the portal's automatic registration failed under his account. I registered `Microsoft.Network`, `Microsoft.Storage`, and `Microsoft.Compute` as the subscription Owner, waited for the status to show *Registered*, and Alex's deployment then succeeded. This confirmed that his access really did stop at the resource group boundary.

**3. CLI user creation failed with an invalid domain.**
I'd left the wrong domain in the command. A user principal name must use one of the tenant's verified domains (here, `<tenant>.onmicrosoft.com`).

## What I learned

- **Entra roles** (e.g., User Administrator) manage the *directory*; **Azure RBAC roles** (e.g., Contributor) manage *resources*.
- Scope matters: management group → subscription → resource group → resource. Access granted at one level flows down, never up.
- Always test permissions by signing in as the user (I used a separate private browser window), not just by reading the role assignment.

## Tools

Azure portal · Microsoft Entra admin center · Azure CLI

## References

- [Assign Azure roles using the Azure portal](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-portal)
- [Manage Microsoft Entra groups](https://learn.microsoft.com/entra/fundamentals/how-to-manage-groups)
- [Security defaults in Microsoft Entra ID](https://learn.microsoft.com/entra/fundamentals/security-defaults)
- [Azure resource providers and types](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-providers-and-types)
