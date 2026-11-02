# Azure Lab 7: Mini Company Network: Active Directory, File Share & Group-Based Drive Mapping

Part of my Azure lab series: [Lab 1: RBAC](../) · [Lab 2: Conditional Access + PIM](../Project2/) · [Lab 3: SAML SSO](../Project3/) · [Lab 4: Static Website](../Project4/) · [Lab 5: CI/CD + OIDC](../Project5/) · [Lab 6: Terraform IaC](../Project6/)

A small on-premises-style Windows network built in Azure. A domain controller hosts a **Finance** file share, and **Group Policy maps drive F: to `\\DC01\Finance` only for members of the `GRP-Finance` security group**. Everyone else gets no drive and is denied access.

## Architecture

```
Azure subscription ── rg-lab-ad (West US 3)
└── vnet-lab-ad  172.16.0.0/24      DNS servers: Custom → 172.16.0.4
    └── snet-servers
        ├── DC01      172.16.0.4 (static)   Windows Server 2022
        │   ├── AD DS + DNS — domain: lab.local
        │   ├── OU: LabUsers → Frank Finance, Riley Sales, GRP-Finance
        │   ├── Share: \\DC01\Finance  (C:\Shares\Finance)
        │   └── GPO "Drive Map - Finance" linked to LabUsers
        └── CLIENT01  (domain-joined)        Windows Server 2022
NSG: RDP 3389 allowed from my public IP only
```

## What I built

### 1. Azure infrastructure (portal)
| Setting | Value | Why |
|---|---|---|
| VM image | Windows Server 2022 Datacenter (Microsoft, x64 Gen2) | Official publisher image. Windows Server license is included in the VM price. |
| VM size | Standard_D2als_v6 (2 vCPU, 4 GiB) | Smallest available size for this subscription (see problems below) |
| DC01 private IP | **Static** 172.16.0.4, set on the Azure NIC | A domain controller's IP must never change. In Azure, set it on the NIC, not inside Windows. |
| VNet DNS | **Custom: 172.16.0.4** | Every VM in the network uses DC01 for DNS so it can find `lab.local` |
| RDP access | NSG rule restricted to **my public IP** | Default rule allowed RDP from anywhere on the internet |
| Auto-shutdown | 11:00 PM Pacific + email notice | Cost control. VMs bill only while running. |
| Delete public IP/NIC with VM | Enabled | No leftover resources when the lab is torn down |

### 2. Active Directory
- Installed **AD DS** and promoted DC01 to a domain controller in a **new forest: `lab.local`** (NetBIOS `LAB`), with integrated DNS
- OU **`LabUsers`** with two test users: **Frank Finance** (`frank.finance`) and **Riley Sales** (`riley.sales`)
- **`GRP-Finance`**: Global Security group, member: Frank only

![AD users and group](Screenshots/01-ad-users-and-group.png)

### 3. File share with two permission layers
Folder `C:\Shares\Finance` shared as **`\\DC01\Finance`** (with a test file `Q4-Budget.txt`).

| Layer | Configuration |
|---|---|
| **Share permissions** (network access) | Removed *Everyone*. `GRP-Finance` = **Change** (read, write, delete, but not change permissions) |
| **NTFS permissions** (file-system access) | Disabled inheritance, removed *Users*. `GRP-Finance` = **Modify**. Administrators/SYSTEM keep Full Control. |

When accessing over the network, Windows applies **the more restrictive** of the two layers.

### 4. Group Policy drive mapping
GPO **`Drive Map - Finance`**, linked to the **LabUsers** OU:
- **User Configuration → Preferences → Windows Settings → Drive Maps**
- Action **Create** · Location **`\\DC01\Finance`** · Label **Finance** · Drive **F:** · Reconnect ✅
- **Common** tab: ✅ Run in logged-on user's security context · ✅ **Item-level targeting** → *the user is a member of the security group `LAB\GRP-Finance`* (resolved to its SID)

![Item-level targeting on the drive map](Screenshots/02-gpo-item-level-targeting.png)

**Design note:** the GPO applies to *everyone* in LabUsers. **Item-level targeting inside the GPO** decides who actually gets the drive. One GPO can hold many drive maps, each targeted to a different group.

## Testing

| Test | User | Expected | Result |
|---|---|---|---|
| CLIENT01 resolves the domain (`nslookup lab.local`) | admin | Answer from 172.16.0.4 | ✅ Pass |
| CLIENT01 joins `lab.local` | admin | "Welcome to the lab.local domain" | ✅ Pass |
| F: drive appears at logon | Frank (GRP-Finance) | ✅ F: Finance (\\DC01) | ✅ Pass |
| Open and create files on F: | Frank | ✅ Allowed | ✅ Pass |
| F: drive appears at logon | Riley (not in group) | ❌ No drive | ✅ Pass (no drive) |
| Browse `\\DC01\Finance` manually | Riley | ❌ Access denied | ✅ Pass (denied) |
| `gpresult /r` | both | GPO listed as applied for both; targeting filters the drive | ✅ Pass |

**Result:** the same GPO reached both users, but only the member of `GRP-Finance` received the F: drive, and the share's permissions independently blocked the non-member even when browsing to the UNC path directly. That's two layers of control: *convenience* (the mapped drive) and *security* (share + NTFS permissions).

<!-- Optional: add test screenshots, e.g.:
![Frank sees F:](Screenshots/03-frank-f-drive.png)
![Riley has no F: drive](Screenshots/04-riley-no-drive.png)
![Riley access denied](Screenshots/05-riley-access-denied.png)
-->

## Problems I hit and how I fixed them

**1. VM size not available (`NotAvailableForSubscription`).**
The default size and the B-series (B2s) were blocked for my subscription. The older B-series is being phased out and restricted on trial subscriptions. **Fix:** used the portal's size picker, which groups sizes by availability. I moved to **West US 3** and picked **Standard_D2als_v6** (2 vCPU, 4 GiB). I kept both VMs at 2 vCPUs to stay within the subscription's regional vCPU quota.

**2. Marketplace image choice.**
Searching for Windows Server returned many third-party repackaged images that add their own hourly software fee. **Fix:** used Microsoft's official image only. I avoided *Core* (no GUI), *Hotpatch*, and *smalldisk* variants.

**3. Windows Server instead of Windows 11 for the client.**
Windows 10/11 images on Azure require confirming an eligible license with multitenant hosting rights, while Windows Server's license is included in the VM price. Drive mapping by Group Policy works the same on both.

**4. Restricting RDP: edit, don't add.**
My first attempt opened *Add inbound security rule*. A new rule wouldn't help, because the existing **priority-300 RDP rule allowed *Any* source**. **Fix:** edited that rule's source to *My IP address*.

**5. Share permission set to Read only.**
The default when adding the group was Read, which would have blocked saving files. **Fix:** set **Change**. I didn't use Full Control, because users shouldn't be able to modify share permissions.

**6. "Stopped" vs "Stopped (deallocated)."**
Shutting down from inside Windows leaves the VM allocated and still billing for compute. **Fix:** always stop from the Azure portal and confirm *Stopped (deallocated)*. Start DC01 before CLIENT01.

## Key concepts

- **AD DS + DNS are inseparable:** clients find domain controllers through DNS, so the VNet DNS must point at the DC.
- **Share vs NTFS permissions:** two independent layers. The most restrictive one wins over the network.
- **GPO scope vs item-level targeting:** linking decides *which users process the GPO*. Targeting decides *which settings inside it apply*.
- **Least privilege:** Change rather than Full Control on the share, and RDP open to one IP rather than the internet.
- **Cost control:** auto-shutdown, deallocating VMs, a budget alert, and resources grouped in one resource group for easy cleanup.

## Cleanup
Stop both VMs between sessions. When finished: delete resource group **`rg-lab-ad`**, which removes the VMs, disks, NICs, public IPs, NSGs, and VNet.

## References
- [Using Group Policy Preferences to map drives based on group membership](https://learn.microsoft.com/en-us/archive/blogs/askds/using-group-policy-preferences-to-map-drives-based-on-group-membership)
- [Group Policy Preferences](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-preferences)
- [Auto-shutdown a VM](https://learn.microsoft.com/azure/virtual-machines/auto-shutdown-vm)
