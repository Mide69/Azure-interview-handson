# Project 10 — Active Directory & Group Policy

## What You Will Learn
- How to promote a Windows Server to a Domain Controller
- How to use Active Directory Users and Computers (ADUC) via GUI and PowerShell
- How to create OUs, users, and groups
- How to create, configure, and link Group Policy Objects (GPOs)
- How to manage DNS and DHCP in an AD environment
- FSMO roles — what they are and which DC holds them

---

## Background — Read This First

**Active Directory Domain Services (AD DS)** is Microsoft's directory service. It provides authentication, authorisation, and a database of all objects (users, computers, groups, policies) in the organisation.

The job spec mentions this multiple times: *"Experience of designing and administering group policy objects for the forests"*, *"administering a complex multi domain active directory environment of around 50k objects"*, and lists FSMO roles as essential knowledge.

**The AD hierarchy:**
```
Forest: nhs-slam.local
└── Domain: nhs-slam.local
    └── Organisational Units (OUs)
        ├── Users
        │   ├── Clinical-Staff
        │   ├── IT-Staff
        ├── Computers
        │   ├── Clinical-PCs
        └── Service-Accounts
```

**Group Policy** enforces settings across hundreds of computers without touching each one — password policies, screen lock timers, drive mappings, software installation.

---

## Prerequisites
- Azure subscription, CLI logged in
- Completed Projects 02 and 07

---

## Step 1 — Deploy a Windows Server VM (the Domain Controller)

### Portal (GUI)
Create a VM in resource group `rg-ad-lab` — Windows Server 2022 Datacenter, Standard_B2s. Allow RDP. See Project 02 for full steps.

### CLI
```bash
az group create --name rg-ad-lab --location uksouth

az vm create \
  --resource-group rg-ad-lab \
  --name vm-dc01 \
  --image Win2022Datacenter \
  --size Standard_B2s \
  --admin-username domainadmin \
  --admin-password "Dom@inP@ss2026!!" \
  --public-ip-sku Standard

az vm open-port --resource-group rg-ad-lab --name vm-dc01 --port 3389 --priority 1001

DC_IP=$(az vm show \
  --resource-group rg-ad-lab --name vm-dc01 \
  --show-details --query publicIps --output tsv)
echo "DC IP: $DC_IP"
```

---

## Step 2 — Install AD DS and Promote to Domain Controller

### Portal (GUI)
1. RDP into the VM (Project 02 connection steps)
2. In Server Manager (opens automatically), click **Add roles and features**
3. Role-based installation → select your server → select **Active Directory Domain Services**
4. Click through to install — a notification appears when done
5. Click the notification flag → **Promote this server to a domain controller**
6. Choose **Add a new forest** | Root domain name: `lab.local`
7. Set the **DSRM password**: `SafeMode@2026!!` (store this — needed for AD recovery)
8. Click through defaults → **Install** — the server reboots automatically

### Azure Run Command (CLI approach)
```bash
az vm run-command invoke \
  --resource-group rg-ad-lab \
  --name vm-dc01 \
  --command-id RunPowerShellScript \
  --scripts "Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools"
```

```bash
az vm run-command invoke \
  --resource-group rg-ad-lab \
  --name vm-dc01 \
  --command-id RunPowerShellScript \
  --scripts "
    \$SafeModePassword = ConvertTo-SecureString 'SafeMode@2026!!' -AsPlainText -Force
    Install-ADDSForest \`
      -DomainName 'lab.local' \`
      -DomainNetbiosName 'LAB' \`
      -SafeModeAdministratorPassword \$SafeModePassword \`
      -InstallDNS \`
      -Force \`
      -NoRebootOnCompletion:\$true
  "
```

Wait 2–3 minutes for the promotion to complete, then verify:
```bash
az vm run-command invoke \
  --resource-group rg-ad-lab --name vm-dc01 \
  --command-id RunPowerShellScript \
  --scripts "Get-ADDomain | Select-Object DNSRoot, NetBIOSName, PDCEmulator"
```

**What you just learned:** The DSRM (Directory Services Restore Mode) password is used to recover a failed AD. Store it in your password manager — losing it means you cannot recover AD if it breaks.

---

## Step 3 — Create Organisational Units (OUs)

### Portal (GUI) — Active Directory Users and Computers
1. RDP into the DC
2. Open **Server Manager** → **Tools** → **Active Directory Users and Computers (ADUC)**
3. Expand `lab.local` in the left panel
4. Right-click `lab.local` → **New** → **Organizational Unit**
5. Name: `NHS-OU` | Tick **Protect container from accidental deletion** → **OK**
6. Right-click `NHS-OU` → **New** → **Organizational Unit** → Name: `Users` → **OK**
7. Inside `Users`, create sub-OUs: `IT-Staff`, `Clinical-Staff`, `Admin-Staff`
8. Back in `NHS-OU`, create: `Computers`, `Service-Accounts`

### PowerShell (on the DC)
```powershell
New-ADOrganizationalUnit -Name 'NHS-OU' -Path 'DC=lab,DC=local' -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name 'Users' -Path 'OU=NHS-OU,DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'IT-Staff' -Path 'OU=Users,OU=NHS-OU,DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'Clinical-Staff' -Path 'OU=Users,OU=NHS-OU,DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'Admin-Staff' -Path 'OU=Users,OU=NHS-OU,DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'Computers' -Path 'OU=NHS-OU,DC=lab,DC=local'
New-ADOrganizationalUnit -Name 'Service-Accounts' -Path 'OU=NHS-OU,DC=lab,DC=local'

# List all OUs
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName | Sort-Object DistinguishedName
```

**What you just learned:** The `DistinguishedName` (DN) is the full path to any object: `OU=IT-Staff,OU=Users,OU=NHS-OU,DC=lab,DC=local`. Read it right to left — domain first, then OUs in order. You use DNs in almost every AD PowerShell command.

---

## Step 4 — Create AD Users

### Portal (GUI) — ADUC
1. In ADUC, expand down to `IT-Staff` OU
2. Right-click it → **New** → **User**
3. Fill in: First name: `Jane` | Last name: `Smith` | User logon name: `jsmith`
4. Click **Next** → Set password: `Welcome@2026!!` | Tick **User must change password at next logon**
5. Click **Finish**
6. Right-click `jsmith` → **Properties** → explore tabs: General, Account, Member Of, Profile

### PowerShell
```powershell
$DefaultPassword = ConvertTo-SecureString 'Welcome@2026!!' -AsPlainText -Force

New-ADUser `
  -Name 'Jane Smith' `
  -GivenName 'Jane' `
  -Surname 'Smith' `
  -SamAccountName 'jsmith' `
  -UserPrincipalName 'jsmith@lab.local' `
  -Path 'OU=IT-Staff,OU=Users,OU=NHS-OU,DC=lab,DC=local' `
  -AccountPassword $DefaultPassword `
  -Enabled $true `
  -ChangePasswordAtLogon $true `
  -Department 'Infrastructure' `
  -Title 'Junior Infrastructure Engineer'

New-ADUser `
  -Name 'Bob Jones' `
  -GivenName 'Bob' `
  -Surname 'Jones' `
  -SamAccountName 'bjones' `
  -UserPrincipalName 'bjones@lab.local' `
  -Path 'OU=IT-Staff,OU=Users,OU=NHS-OU,DC=lab,DC=local' `
  -AccountPassword $DefaultPassword `
  -Enabled $true `
  -ChangePasswordAtLogon $true

# List users in the OU
Get-ADUser -Filter * -SearchBase 'OU=IT-Staff,OU=Users,OU=NHS-OU,DC=lab,DC=local' | Select-Object Name, SamAccountName, Enabled
```

---

## Step 5 — Common User Management Tasks

### Portal (GUI) — ADUC
- **Reset password:** right-click user → **Reset Password**
- **Disable account:** right-click → **Disable Account**
- **Move to different OU:** right-click → **Move** → select target OU
- **Unlock account:** right-click → **Unlock Account** (appears when locked)
- **Member of:** right-click → **Properties** → **Member Of** tab

### PowerShell
```powershell
# Reset password (common helpdesk task)
$NewPass = ConvertTo-SecureString 'NewP@ss2026!!' -AsPlainText -Force
Set-ADAccountPassword -Identity jsmith -NewPassword $NewPass -Reset
Set-ADUser -Identity jsmith -ChangePasswordAtLogon $true

# Disable account (when someone leaves)
Disable-ADAccount -Identity bjones

# Re-enable
Enable-ADAccount -Identity bjones

# Move user to different OU
Move-ADObject `
  -Identity (Get-ADUser bjones).DistinguishedName `
  -TargetPath 'OU=Admin-Staff,OU=Users,OU=NHS-OU,DC=lab,DC=local'

# Unlock a locked account
Unlock-ADAccount -Identity jsmith

# Find all locked accounts
Search-ADAccount -LockedOut | Select-Object Name, LockedOut, LastLogonDate

# Find inactive users (no login in 90 days)
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
  -Properties LastLogonDate | Select-Object Name, SamAccountName, LastLogonDate
```

---

## Step 6 — Create and Manage Security Groups

### Portal (GUI) — ADUC
1. Right-click `NHS-OU` → **New** → **Group**
2. Group name: `IT-Infrastructure-Team` | Scope: **Global** | Type: **Security**
3. Double-click the group → **Members** tab → **Add** → search for `jsmith` and `bjones` → **OK**

### PowerShell
```powershell
New-ADGroup `
  -Name 'IT-Infrastructure-Team' `
  -GroupScope Global `
  -GroupCategory Security `
  -Path 'OU=NHS-OU,DC=lab,DC=local' `
  -Description 'Infrastructure team members'

# Add members
Add-ADGroupMember -Identity 'IT-Infrastructure-Team' -Members jsmith, bjones

# List members
Get-ADGroupMember -Identity 'IT-Infrastructure-Team' | Select-Object Name, SamAccountName

# Find which groups a user belongs to
Get-ADPrincipalGroupMembership -Identity jsmith | Select-Object Name
```

---

## Step 7 — Create and Link a Group Policy Object (GPO)

### Portal (GUI) — Group Policy Management Console (GPMC)
1. In Server Manager → **Tools** → **Group Policy Management**
2. Expand Forest → Domains → `lab.local` → right-click `IT-Staff` OU → **Create a GPO in this domain, and Link it here**
3. Name: `IT-Password-Policy` → **OK**
4. Right-click `IT-Password-Policy` → **Edit** — the Group Policy Management Editor opens
5. Navigate to: **Computer Configuration** → **Windows Settings** → **Security Settings** → **Account Policies** → **Password Policy**
6. Double-click **Minimum password length** → Enable → set to `14` → **OK**
7. Double-click **Password must meet complexity requirements** → **Enabled** → **OK**
8. Close the editor — the policy is saved and linked automatically

### PowerShell
```powershell
# Create a new GPO
New-GPO -Name 'IT-Password-Policy' -Comment 'Strong password enforcement for IT staff'

# Link to the IT-Staff OU
New-GPLink `
  -Name 'IT-Password-Policy' `
  -Target 'OU=IT-Staff,OU=Users,OU=NHS-OU,DC=lab,DC=local' `
  -LinkEnabled Yes

# List all GPOs
Get-GPO -All | Select-Object DisplayName, GpoStatus | Format-Table

# List GPOs linked to an OU (shows inheritance)
Get-GPInheritance -Target 'OU=IT-Staff,OU=Users,OU=NHS-OU,DC=lab,DC=local'

# Force all computers to apply latest GPOs immediately
# Invoke-GPUpdate -Force
```

**What you just learned:** GPOs are one of AD's most powerful features. They can configure thousands of settings across every computer in the domain without touching each machine. The NHS uses GPOs extensively to enforce security baselines (CIS/DSPT standards), map network drives, deploy software, and set screen lock policies.

---

## Step 8 — DNS Management

### Portal (GUI) — DNS Manager
1. In Server Manager → **Tools** → **DNS**
2. Expand the server → **Forward Lookup Zones** → `lab.local`
3. Right-click the zone → **New Host (A or AAAA)**
4. Name: `webserver` | IP: `10.10.1.50` → **Add Host**
5. Right-click → **New Alias (CNAME)** → Alias: `www` | FQDN: `webserver.lab.local`

### PowerShell
```powershell
# View DNS zones
Get-DnsServerZone | Select-Object ZoneName, ZoneType

# Add an A record
Add-DnsServerResourceRecordA -ZoneName 'lab.local' -Name 'webserver' -IPv4Address '10.10.1.50'

# Add a CNAME
Add-DnsServerResourceRecordCName -ZoneName 'lab.local' -Name 'www' -HostNameAlias 'webserver.lab.local'

# List records
Get-DnsServerResourceRecord -ZoneName 'lab.local' | Select-Object HostName, RecordType, TimeToLive

# Test resolution
Resolve-DnsName -Name 'webserver.lab.local' -Server '127.0.0.1'
```

---

## Step 9 — FSMO Roles

Five special AD roles where only one DC can hold each at a time.

### Portal (GUI)
In ADUC: right-click the domain → **Operations Masters** — shows RID, PDC, Infrastructure roles. For Schema Master and Domain Naming Master: use Active Directory Schema snap-in and Active Directory Domains and Trusts respectively.

### PowerShell
```powershell
# View FSMO role holders
netdom query fsmo

# PowerShell way
Get-ADDomain | Select-Object PDCEmulator, RIDMaster, InfrastructureMaster
Get-ADForest | Select-Object SchemaMaster, DomainNamingMaster
```

**The five FSMO roles:**
| Role | Scope | What breaks if it goes down |
|---|---|---|
| **Schema Master** | Forest | Cannot extend the AD schema |
| **Domain Naming Master** | Forest | Cannot add/remove domains |
| **PDC Emulator** | Domain | Time sync breaks, password changes queue, account lockouts slow |
| **RID Master** | Domain | Cannot create new AD objects |
| **Infrastructure Master** | Domain | Cross-domain group membership issues |

**What you just learned:** The PDC Emulator is the most important for daily operations. It handles time synchronisation (Kerberos requires clocks within 5 minutes), processes password changes, and manages account lockout policies. If it goes offline, users start getting unexpected lockouts and password change delays.

---

## Step 10 — Search AD at Scale

```powershell
# Count all objects
(Get-ADObject -Filter *).Count

# Find disabled users
Get-ADUser -Filter {Enabled -eq $false} | Select-Object Name, SamAccountName

# Find users inactive for 90 days
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
  -Properties LastLogonDate | Select-Object Name, SamAccountName, LastLogonDate

# Find empty security groups
Get-ADGroup -Filter * | Where-Object {
  (Get-ADGroupMember -Identity $_).Count -eq 0
} | Select-Object Name

# Find all computers in the domain
Get-ADComputer -Filter * | Measure-Object | Select-Object Count
```

---

## Step 11 — Clean Up

### Portal (GUI)
Navigate to `rg-ad-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-ad-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Domain Controller | Server running AD DS — holds the directory database |
| OU | Organisational folder in AD — where you link GPOs |
| GPO | Group Policy Object — enforces settings across users and computers |
| Distinguished Name | Full path to an AD object: `OU=IT,DC=lab,DC=local` |
| FSMO Roles | Five special roles — PDC Emulator most important for daily ops |
| DSRM Password | Recovery password for AD — store it securely |
| `Get-ADUser` | Most common AD PowerShell command |

---

## Common Interview Questions

1. **"A user changed their password but can still log in with the old one on some machines. Why?"** — Kerberos caches tickets, and password changes replicate from the PDC Emulator to other DCs. There may be a replication delay. Also, clients may be authenticating to a DC that has not yet received the password change. This usually resolves within the replication interval.

2. **"What are the five FSMO roles and which is most important for daily operations?"** — Schema Master (can modify schema), Domain Naming Master (add/remove domains), PDC Emulator (time sync + password changes + lockouts), RID Master (issue new SIDs for object creation), Infrastructure Master (cross-domain group membership). PDC Emulator affects users most directly on a daily basis.

3. **"What is the difference between a GPO applied to a computer OU vs a user OU?"** — Computer GPOs apply at machine boot (before login) — good for security baselines and drive mappings. User GPOs apply at user login — good for desktop settings and folder redirection. Computer policies are generally more security-critical because they apply regardless of who logs in.

---

## Next Steps
Move on to **Project 11 — Patch Management**.
