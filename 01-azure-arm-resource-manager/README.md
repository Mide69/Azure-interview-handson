# Project 01 — Azure ARM & Resource Manager

## What You Will Learn
- What Azure Resource Manager (ARM) is and why it matters
- How to create and manage Resource Groups via the Portal and CLI
- How to write and deploy an ARM template (Infrastructure as Code)
- How to use `what-if` (dry run) before deploying
- How to tag resources for cost tracking

---

## Background — Read This First

Every single thing you create in Azure (a VM, a storage account, a network) is a **resource**. ARM is the control plane that sits between you and those resources. Every click in the portal, every CLI command — they all call the same ARM API underneath.

A **Resource Group** is a logical container. Think of it as a folder. Delete the folder, everything inside goes with it.

An **ARM template** is a JSON file describing what you want Azure to build. Instead of clicking the portal, you declare infrastructure in code — this is **Infrastructure as Code (IaC)**.

---

## Prerequisites

- An active Azure subscription (free tier works)
- Azure CLI installed: `https://learn.microsoft.com/en-us/cli/azure/install-azure-cli`
- A browser for the Portal steps

---

## Step 1 — Log In to Azure

### Portal (GUI)
1. Open your browser and go to **portal.azure.com**
2. Sign in with your Microsoft account
3. You land on the Azure Home page — the dashboard showing your recent resources
4. In the top bar, notice the **search box** — this is the fastest way to find any service
5. Click the **Cloud Shell** icon (`>_`) in the top-right toolbar — this opens a browser-based terminal inside the portal (useful for quick CLI work without installing anything)

### CLI
```bash
az login
```
A browser opens. Sign in. Return to the terminal.

```bash
# Confirm you're signed in to the right account
az account show

# List all your subscriptions
az account list --output table

# Set the subscription to use
az account set --subscription "YOUR-SUBSCRIPTION-NAME-OR-ID"
```

**What you just learned:** The Portal and CLI are two interfaces to the same underlying ARM API. The Portal is visual and good for exploration; the CLI is scriptable and good for automation and repeatable work.

---

## Step 2 — Create a Resource Group

### Portal (GUI)
1. In the top search bar, type **Resource groups** and click it
2. Click **+ Create** (top-left)
3. Fill in:
   - **Subscription:** your subscription
   - **Resource group:** `rg-arm-lab`
   - **Region:** `(Europe) UK South`
4. Click **Review + create**, then **Create**
5. Click **Go to resource group** — notice it is empty
6. Look at the left panel: **Overview**, **Access control (IAM)**, **Tags**, **Deployments** — remember these tabs, you'll use them constantly

### CLI
```bash
az group create \
  --name rg-arm-lab \
  --location uksouth
```

List all resource groups:
```bash
az group list --output table
```

Show details of your new group:
```bash
az group show --name rg-arm-lab
```

**What you just learned:** Resource groups are the foundation of everything. Always create one per project. The **Deployments** tab in the Portal shows every ARM deployment ever run against this group — your audit trail.

---

## Step 3 — Deploy a Storage Account Manually

Before writing a template, deploy something to understand what we are automating.

### Portal (GUI)
1. Inside `rg-arm-lab`, click **+ Create** (or **Create resources**)
2. In the Marketplace search box, type **Storage account** and click it
3. Click **Create**
4. Fill in:
   - **Resource group:** `rg-arm-lab`
   - **Storage account name:** `starmlab001` (must be globally unique, lowercase, 3–24 chars)
   - **Region:** UK South
   - **Performance:** Standard
   - **Redundancy:** Locally-redundant storage (LRS)
5. Click **Review + create**, then **Create**
6. Click **Go to resource** when it completes — explore the left panel (Containers, File shares, Access keys)

### CLI
```bash
az storage account create \
  --name starmlab001 \
  --resource-group rg-arm-lab \
  --location uksouth \
  --sku Standard_LRS \
  --kind StorageV2
```

List resources in your group:
```bash
az resource list --resource-group rg-arm-lab --output table
```

**What you just learned:** `Standard_LRS` = Locally Redundant Storage — 3 copies in one datacentre, cheapest option. Good for labs. For production NHS data use at least GRS (Geo-Redundant Storage).

---

## Step 4 — Export an ARM Template from an Existing Resource

Azure can generate an ARM template from what you have already built — a great way to learn template syntax.

### Portal (GUI)
1. Navigate to your `rg-arm-lab` resource group
2. In the left panel, click **Export template** (under Automation)
3. You see the generated JSON template — Azure reverse-engineered your infrastructure into code
4. Notice the sections: `$schema`, `parameters`, `variables`, `resources`, `outputs`
5. Click **Download** to save the JSON file locally

### CLI
```bash
az group export --name rg-arm-lab > exported-template.json
cat exported-template.json
```

The structure looks like:
```json
{
  "$schema": "...",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "variables": {},
  "resources": [ ... ],
  "outputs": {}
}
```

**What you just learned:** ARM templates have a fixed structure. `resources` is the core section. `parameters` make the template reusable — pass different values each time you deploy.

---

## Step 5 — Write Your Own ARM Template From Scratch

### Portal (GUI)
1. In the top search bar, type **Deploy a custom template** and click it
2. Click **Build your own template in the editor**
3. Delete the default content
4. Paste in the template below and click **Save**
5. Fill in the parameters and deploy (covered in Step 6)

### CLI
Create the file locally:
```bash
cat > storage-template.json << 'EOF'
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string",
      "metadata": {
        "description": "The name of the storage account"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for the storage account"
      }
    }
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "StorageV2",
      "properties": {
        "accessTier": "Hot"
      }
    }
  ],
  "outputs": {
    "storageAccountId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
    }
  }
}
EOF
```

**What you just learned:** `[parameters('...')]` is ARM's expression language. `[resourceGroup().location]` reads the resource group's location automatically. `outputs` surface values after deployment — useful for chaining templates.

---

## Step 6 — Deploy the ARM Template

### Portal (GUI)
1. From the **Deploy a custom template** screen (search it in the top bar)
2. Click **Build your own template in the editor**, paste your template, click **Save**
3. Fill in:
   - **Resource group:** `rg-arm-lab`
   - **Storage account name:** `starmtemplate002`
4. Click **Review + create**, then **Create**
5. After deployment: go to `rg-arm-lab` → **Deployments** tab — you see the deployment listed with its status and parameters

### CLI
```bash
az deployment group create \
  --resource-group rg-arm-lab \
  --template-file storage-template.json \
  --parameters storageAccountName=starmtemplate002
```

Verify:
```bash
az resource list --resource-group rg-arm-lab --output table
```

**What you just learned:** `az deployment group create` applies the template. The Deployments tab in the Portal shows every deployment run against this resource group — a built-in audit trail. In NHS change management, this satisfies the "document your changes" requirement.

---

## Step 7 — View Deployment History

### Portal (GUI)
1. Navigate to `rg-arm-lab`
2. Click **Deployments** in the left panel
3. Click any deployment to see: **Input parameters**, **Output values**, **Template** used, and the deployment **Operations** (each resource created)
4. This is the full audit record of what was deployed, when, and by whom

### CLI
```bash
az deployment group list --resource-group rg-arm-lab --output table

az deployment group show \
  --resource-group rg-arm-lab \
  --name storage-template \
  --output json
```

**What you just learned:** ARM deployment history is immutable — you cannot edit it. This makes it a reliable audit trail. If an auditor asks "what changed and when?", the Deployments tab is your answer.

---

## Step 8 — Use `what-if` Before Deploying (Dry Run)

Always preview changes before applying them in a production environment.

### Portal (GUI)
1. Go to **Deploy a custom template**, load your template
2. Fill in the parameters but instead of clicking Create, click **Preview changes** (under Review + create)
3. The Portal shows you exactly what would be created, modified, or deleted — colour coded

### CLI
```bash
az deployment group what-if \
  --resource-group rg-arm-lab \
  --template-file storage-template.json \
  --parameters storageAccountName=starmtemplate002
```

Output colours: green = create, yellow = modify, red = delete.

**What you just learned:** `what-if` is ARM's equivalent of `terraform plan`. Always run it before applying changes in a production environment. In NHS change management, the what-if output can be included in your change request as evidence of what will change.

---

## Step 9 — Tag Your Resources

Tags are key-value pairs for cost tracking and organisation.

### Portal (GUI)
1. Navigate to `rg-arm-lab`
2. Click **Tags** in the left panel
3. Add:
   - Name: `environment` / Value: `lab`
   - Name: `project` / Value: `interview-prep`
   - Name: `team` / Value: `infrastructure`
4. Click **Apply**
5. To find resources by tag: in the top search bar, type **Tags**, click the Tags service, select `environment : lab` to see all tagged resources

### CLI
```bash
az group update \
  --name rg-arm-lab \
  --tags environment=lab project=interview-prep team=infrastructure

# Find resources by tag
az resource list --tag environment=lab --output table
```

**What you just learned:** Without tags, it is impossible to track costs per team or project. NHS trusts use tags to allocate costs to cost centres (e.g. which ward or service owns each resource). Always ask your new team "what tags should I apply?" before creating resources.

---

## Step 10 — Clean Up (Always Do This After Labs)

### Portal (GUI)
1. Navigate to `rg-arm-lab`
2. Click **Delete resource group** at the top
3. Type the resource group name to confirm
4. Click **Delete** — everything inside is deleted

### CLI
```bash
az group delete --name rg-arm-lab --yes --no-wait
```

`--yes` skips the confirmation prompt. `--no-wait` returns you to the terminal immediately while deletion runs in the background.

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| ARM | Control plane for all Azure resources — every portal click calls this API |
| Resource Group | Logical container — delete the group, delete everything in it |
| ARM Template | JSON describing infrastructure — version controlled, repeatable |
| `what-if` | Dry-run — shows what would change without making changes |
| Tags | Key-value metadata for cost tracking and resource filtering |
| Deployment History | Immutable audit log of every ARM deployment |

---

## Common Interview Questions

1. **"What is ARM and why does it matter?"** — ARM is the control plane for all Azure resources. Every action — portal, CLI, SDK, REST API — goes through ARM. It enforces RBAC, policies, locks, and keeps deployment history. It matters because it is the single authoritative layer for everything in Azure.

2. **"Why would you use an ARM template instead of the CLI?"** — Templates are declarative and idempotent — run them twice, get the same result. CLI commands are imperative — each runs once. For repeatable, version-controlled infrastructure (e.g. deploying a consistent dev/test/prod environment), templates win.

3. **"How do you roll back a bad ARM deployment?"** — ARM is not transactional. You re-deploy the previous template version. Use `what-if` before every deployment to understand the impact. In a complete-mode deployment, resources not in the template are deleted — be careful with that mode.

---

## Next Steps
Move on to **Project 02 — Azure Virtual Machines**.
