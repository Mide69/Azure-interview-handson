# Project 05 — Azure Entra ID & RBAC

## What You Will Learn
- What Entra ID is and how it differs from on-premises Active Directory
- How to create users and groups via Portal and CLI
- How Azure RBAC (Role-Based Access Control) works
- How to assign roles at different scopes
- How to create Service Principals and Managed Identities
- The principle of least privilege in practice

---

## Background — Read This First

**Azure Entra ID** (formerly Azure Active Directory) is Microsoft's cloud identity platform. It is NOT the same as on-premises AD — though they can synchronise.

| Feature | On-Prem Active Directory | Azure Entra ID |
|---|---|---|
| Authentication | Kerberos, NTLM | OAuth 2.0, SAML, OpenID Connect |
| Objects | Users, Groups, OUs, GPOs, Computer accounts | Users, Groups, Applications, Service Principals |
| Joined devices | Domain-joined | Entra-joined or Hybrid-joined |
| Managed via | ADUC, GPMC, PowerShell | Azure Portal, CLI, PowerShell, Graph API |

**RBAC (Role-Based Access Control)** — who can do what, where. Three components:
- **Role** — a set of permissions (e.g. "Virtual Machine Contributor" = can manage VMs but not networking)
- **Identity** — who gets the role (user, group, service principal, managed identity)
- **Scope** — where the role applies (management group > subscription > resource group > resource)

---

## Prerequisites
- Azure subscription, CLI logged in
- Sufficient permissions (Owner or User Access Administrator on your subscription)

---

## Step 1 — Explore Your Entra ID Tenant

### Portal (GUI)
1. Search **Microsoft Entra ID** (or **Azure Active Directory**) in the top bar → click it
2. The **Overview** page shows: tenant name, tenant ID, user count, group count
3. Click **Users** in the left panel — see all users in your directory
4. Click **Groups** — see all groups
5. Click **App registrations** — see all application identities
6. Click **Roles and administrators** — see all built-in roles and who holds them

### CLI
```bash
az account show
TENANT_ID=$(az account show --query tenantId --output tsv)
echo "Tenant ID: $TENANT_ID"
```

---

## Step 2 — Create a New User

### Portal (GUI)
1. In Entra ID, click **Users** → **+ New user** → **Create new user**
2. Fill in:
   - User name: `jsmith@yourtenant.onmicrosoft.com`
   - Name: `Jane Smith`
   - Password: auto-generate or set manually
   - Under **Properties**: Department = `Infrastructure`, Job title = `Junior Infrastructure Engineer`
3. Click **Create**
4. The user appears in the list — click them to view their profile
5. Under **Assigned roles** — currently empty (no Azure roles assigned yet)

### CLI
```bash
az ad user create \
  --display-name "Jane Smith" \
  --user-principal-name "jsmith@$(az account show --query user.name -o tsv | cut -d@ -f2)" \
  --password "TempP@ssw0rd2026!" \
  --force-change-password-next-sign-in true

az ad user list --output table
```

**What you just learned:** `--force-change-password-next-sign-in true` is a security best practice — new accounts always require a password change at first login. This ensures the user (not the admin) knows their password.

---

## Step 3 — Create Security Groups

### Portal (GUI)
1. In Entra ID, click **Groups** → **+ New group**
2. Group type: **Security**
3. Group name: `Infrastructure-Team`
4. Description: `Infrastructure team members`
5. Members: click **No members selected** → search for `Jane Smith` → select → **Select**
6. Click **Create**
7. Repeat to create `Azure-Admins` group (leave empty for now)

### CLI
```bash
az ad group create \
  --display-name "Infrastructure-Team" \
  --mail-nickname "infrastructure-team"

az ad group create \
  --display-name "Azure-Admins" \
  --mail-nickname "azure-admins"

# Get the group's Object ID (needed for role assignments)
INFRA_GROUP_ID=$(az ad group show \
  --group "Infrastructure-Team" \
  --query id \
  --output tsv)
echo "Infrastructure Team Group ID: $INFRA_GROUP_ID"
```

---

## Step 4 — Add a User to a Group

### Portal (GUI)
1. Navigate to the `Infrastructure-Team` group
2. Click **Members** in the left panel → **+ Add members**
3. Search for `Jane Smith` → select → **Select**
4. Jane appears in the members list

### CLI
```bash
USER_ID=$(az ad user show \
  --id "jsmith@yourtenant.onmicrosoft.com" \
  --query id --output tsv)

az ad group member add \
  --group "Infrastructure-Team" \
  --member-id $USER_ID

az ad group member list \
  --group "Infrastructure-Team" \
  --output table
```

**What you just learned:** Assign roles to groups, not individual users. When someone joins the team, add them to the group — they inherit all permissions instantly. When they leave, remove them from the group — all permissions are revoked instantly. This is role-based access in practice.

---

## Step 5 — Assign an RBAC Role to a Group

### Portal (GUI)
**Create a resource group to use as scope:**
1. Create resource group `rg-rbac-lab` (UK South)

**Assign the role:**
2. Navigate to `rg-rbac-lab`
3. Click **Access control (IAM)** in the left panel
4. Click **+ Add** → **Add role assignment**
5. **Role tab:** search for `Contributor` → select it → click **Next**
6. **Members tab:**
   - Assign access to: **User, group, or service principal**
   - Click **+ Select members** → search `Infrastructure-Team` → select → **Select**
7. Click **Review + assign** → **Review + assign**
8. Back in **Access control (IAM)**, click **Role assignments** tab — you see the group listed with Contributor role

### CLI
```bash
az group create --name rg-rbac-lab --location uksouth

RG_ID=$(az group show --name rg-rbac-lab --query id --output tsv)

az role assignment create \
  --assignee $INFRA_GROUP_ID \
  --role "Contributor" \
  --scope $RG_ID

az role assignment list --scope $RG_ID --output table
```

**What you just learned:** Scope is critical. A Contributor role on `rg-rbac-lab` means the group can create/modify resources in THAT resource group only — not anywhere else in the subscription. The Portal IAM blade is where you manage access for any Azure resource.

---

## Step 6 — Explore Built-In Roles

### Portal (GUI)
1. In `rg-rbac-lab`, click **Access control (IAM)** → **Roles** tab
2. Browse the list — there are 100+ built-in roles
3. Click `Contributor` → **View** — see the full list of allowed Actions and NotActions
4. Click `Virtual Machine Contributor` — note it allows VM management but not networking or storage

### CLI
```bash
# List all built-in roles
az role definition list --output table | head -30

# Find roles related to storage
az role definition list \
  --query "[?contains(roleName,'Storage')]" \
  --output table

# Show permissions for a specific role
az role definition list \
  --name "Virtual Machine Contributor" \
  --output json
```

---

## Step 7 — Create a Custom Role

Sometimes built-in roles are too broad or too narrow.

### Portal (GUI)
1. In `rg-rbac-lab`, click **Access control (IAM)** → **Roles** tab → **+ Create**
2. **Basics:** Name: `VM Start-Stop Operator`, Description: `Can start and stop VMs only`
3. **Permissions tab:** click **+ Add permissions**
4. Search for `Microsoft.Compute` → select it
5. Tick: `Microsoft.Compute/virtualMachines/start/action`, `/deallocate/action`, `/restart/action`, `/read`
6. Click **Add** → **Review + create** → **Create**

### CLI
```bash
SUB_ID=$(az account show --query id --output tsv)

cat > custom-role.json << EOF
{
  "Name": "VM Start-Stop Operator",
  "IsCustom": true,
  "Description": "Can start and stop VMs but cannot create or delete them",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/$SUB_ID"]
}
EOF

az role definition create --role-definition custom-role.json
```

**What you just learned:** Custom roles enforce the principle of least privilege — grant exactly the permissions needed, nothing more. In an NHS trust, an on-call engineer who only needs to restart VMs should not have full Contributor access.

---

## Step 8 — Create a Service Principal (Identity for Scripts/Apps)

### Portal (GUI)
1. In Entra ID, click **App registrations** → **+ New registration**
2. Name: `sp-infra-lab` | Supported account types: this directory only
3. Click **Register**
4. Note the **Application (client) ID** and **Directory (tenant) ID**
5. Click **Certificates & secrets** → **+ New client secret**
6. Description: `lab-secret` | Expires: 6 months
7. Click **Add** — copy the **Value** immediately (you cannot see it again)
8. Now assign it a role: go to `rg-rbac-lab` → **Access control (IAM)** → **Add role assignment** → Contributor → select `sp-infra-lab`

### CLI
```bash
RG_ID=$(az group show --name rg-rbac-lab --query id --output tsv)

az ad sp create-for-rbac \
  --name sp-infra-lab \
  --role Contributor \
  --scopes $RG_ID \
  --output json
# Save the appId, password, and tenant from the output — password shown once only
```

**What you just learned:** Service principals are identity for apps and automation. A CI/CD pipeline, a script, or a scheduled task uses a service principal to authenticate to Azure — not a human's account. Never tie automation to a person's account (if they leave, automation breaks).

---

## Step 9 — Managed Identity (The Better Alternative)

### Portal (GUI)
**Create a VM with a managed identity:**
1. Create a VM (as in Project 02), and on the **Management** tab:
   - Under **Identity**, turn on **System assigned managed identity**
2. After creation, on the VM's **Identity** blade: status = **On**, Object ID is shown
3. Assign the identity a role: go to `rg-rbac-lab` → **Access control (IAM)** → **Add role assignment** → Reader → search for your VM name → select it as a managed identity

### CLI
```bash
az vm create \
  --resource-group rg-rbac-lab \
  --name vm-identity-lab \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --assign-identity

IDENTITY_PRINCIPAL=$(az vm show \
  --resource-group rg-rbac-lab \
  --name vm-identity-lab \
  --query "identity.principalId" \
  --output tsv)

az role assignment create \
  --assignee $IDENTITY_PRINCIPAL \
  --role "Reader" \
  --scope $RG_ID
```

**What you just learned:** Managed Identities have NO credentials to manage — Azure handles authentication automatically. The VM can call Azure APIs (like "list all VMs in this resource group") without any password or secret in the code. This eliminates credential leakage — one of the most common cloud security failures.

---

## Step 10 — Remove a Role Assignment

### Portal (GUI)
1. Go to `rg-rbac-lab` → **Access control (IAM)** → **Role assignments** tab
2. Find the `Infrastructure-Team` Contributor assignment
3. Tick the checkbox next to it → click **Remove** → confirm

### CLI
```bash
az role assignment delete \
  --assignee $INFRA_GROUP_ID \
  --role "Contributor" \
  --scope $RG_ID
```

---

## Step 11 — Clean Up

### Portal (GUI)
Delete resource group `rg-rbac-lab`. Also delete the test user and service principal from Entra ID → Users and App registrations.

### CLI
```bash
az group delete --name rg-rbac-lab --yes --no-wait
az ad sp delete --id "APP_ID_FROM_STEP_8"
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Entra ID | Cloud identity platform — NOT the same as on-prem AD |
| RBAC | Who can do what, where — role + identity + scope |
| Owner vs Contributor | Owner can also manage access (IAM); Contributor cannot |
| Scope | Where the role applies — narrows from subscription → resource group → resource |
| Service Principal | Identity for apps/automation — has a client ID and secret |
| Managed Identity | Azure-managed identity — no credentials to store or rotate |
| Least Privilege | Grant only the minimum permissions actually needed |

---

## Common Interview Questions

1. **"How would you give a new team member access to manage VMs in Azure?"** — Add them to a security group (e.g. `Infrastructure-Team`). Assign the `Virtual Machine Contributor` built-in role to that group at the appropriate scope (resource group or subscription). The new member inherits permissions via group membership.

2. **"What is the difference between a Service Principal and a Managed Identity?"** — Both are app identities. A service principal has credentials (client ID + secret) you must manage, store securely, and rotate. A managed identity has no credentials — Azure manages the authentication entirely. Always prefer Managed Identities for Azure-to-Azure authentication.

3. **"Someone in your team accidentally deleted a production VM. What permissions change would you make?"** — Move the team from Contributor (can delete resources) to a custom role without delete permissions, or to `Virtual Machine Contributor` (cannot delete the resource group). Apply a resource lock (Delete lock) to critical VMs — even Owners cannot delete a locked resource without first removing the lock.

---

## Next Steps
Move on to **Project 06 — Azure Backup**.
