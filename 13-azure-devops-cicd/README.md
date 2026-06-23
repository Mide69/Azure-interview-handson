# Project 13 — Azure DevOps & CI/CD Pipelines

## What You Will Learn
- What CI/CD means and why infrastructure teams need to understand it
- How to create an Azure DevOps project and repository
- How to write a YAML pipeline for infrastructure deployments
- How to use service connections to let pipelines deploy to Azure
- How to add approval gates for production deployments
- How to view pipeline runs and troubleshoot failures

---

## Background — Read This First

**CI** (Continuous Integration) — every time code is pushed, automated tests run. Problems found in minutes, not days.

**CD** (Continuous Deployment/Delivery) — after passing tests, code is automatically deployed to an environment.

For infrastructure engineers at the NHS, this means:
- ARM templates / Terraform / Bicep files are tested automatically on every PR
- Deployments to test environments happen automatically
- Deployments to production require a human approval
- Every change is tracked in git — full audit trail for compliance

**Azure DevOps** is Microsoft's DevOps platform. It includes: Repos (git), Pipelines (CI/CD), Boards (work items), Artifacts (package registry), Test Plans.

---

## Prerequisites
- Azure subscription, CLI logged in
- An Azure DevOps organisation — sign up free at dev.azure.com
- Azure CLI DevOps extension: `az extension add --name azure-devops`

---

## Step 1 — Create an Azure DevOps Project

### Portal (GUI)
1. Go to dev.azure.com → sign in with your Microsoft account
2. Click **New organization** (or use an existing one)
3. Click **+ New project**
4. Name: `nhs-infra-lab`
5. Visibility: **Private**
6. Click **Create**

### CLI
```bash
# Set your organisation URL (replace with your org name)
az devops configure --defaults organization=https://dev.azure.com/YOUR-ORG

# Create project
az devops project create \
  --name nhs-infra-lab \
  --visibility private \
  --source-control git
```

---

## Step 2 — Create a Repository and Push Code

### Portal (GUI)
1. In your new project, click **Repos** in the left navigation
2. You see the default empty repository
3. Click **Initialize** (or clone it to your machine and push files manually)
4. Click the **+ (New)** button → **New file**
5. Create: `infrastructure/main.bicep` with a simple resource group template

### CLI
```bash
# Clone the repo (get the URL from the Portal → Repos → Clone)
git clone https://YOUR-ORG@dev.azure.com/YOUR-ORG/nhs-infra-lab/_git/nhs-infra-lab
cd nhs-infra-lab

# Create folder structure
mkdir -p infrastructure pipelines

# Create a simple Bicep template
cat > infrastructure/storage.bicep << 'EOF'
param storageAccountName string = 'stlabcicd${uniqueString(resourceGroup().id)}'
param location string = resourceGroup().location

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}

output storageAccountId string = storageAccount.id
EOF

git add .
git commit -m "Add storage account Bicep template"
git push
```

---

## Step 3 — Create a Service Connection (Azure Credentials for Pipelines)

The pipeline needs permission to deploy to Azure. A service connection stores those credentials.

### Portal (GUI)
1. In Azure DevOps, click **Project settings** (gear icon, bottom-left)
2. Under Pipelines → **Service connections** → **+ New service connection**
3. Choose **Azure Resource Manager** → **Service principal (automatic)** → **Next**
4. Scope: **Subscription** → select your subscription
5. Resource group: leave blank (allows the pipeline to deploy to any resource group)
6. Service connection name: `azure-subscription-connection`
7. Tick **Grant access permission to all pipelines** → **Save**

### CLI
```bash
# Create a service principal for the pipeline
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name sp-azdevops-lab \
  --role Contributor \
  --scopes /subscriptions/$(az account show --query id --output tsv))

echo $SP_OUTPUT
# Note the appId, password, and tenant — needed for the service connection
```

---

## Step 4 — Create a CI Pipeline (YAML)

### Portal (GUI)
1. In Azure DevOps, click **Pipelines** → **+ New pipeline**
2. Select **Azure Repos Git** → select `nhs-infra-lab`
3. Select **Starter pipeline** — edit it with the YAML below
4. Click **Save and run** → **Save and run** (commits the pipeline file to the repo)

### YAML File
```bash
cat > pipelines/ci-pipeline.yml << 'EOF'
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/**

pool:
  vmImage: 'ubuntu-latest'

variables:
  resourceGroup: 'rg-cicd-lab'
  location: 'uksouth'
  serviceConnection: 'azure-subscription-connection'

stages:
  - stage: Validate
    displayName: Validate Bicep Templates
    jobs:
      - job: ValidateJob
        displayName: Lint and Validate
        steps:
          - task: AzureCLI@2
            displayName: Install Bicep CLI
            inputs:
              azureSubscription: $(serviceConnection)
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                az bicep install
                az bicep version

          - task: AzureCLI@2
            displayName: Bicep Lint
            inputs:
              azureSubscription: $(serviceConnection)
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                az bicep build --file infrastructure/storage.bicep
                echo "Bicep template compiled successfully"

          - task: AzureCLI@2
            displayName: What-If (Dry Run)
            inputs:
              azureSubscription: $(serviceConnection)
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                az group create --name $(resourceGroup) --location $(location) --output none || true
                az deployment group what-if \
                  --resource-group $(resourceGroup) \
                  --template-file infrastructure/storage.bicep

  - stage: DeployDev
    displayName: Deploy to Dev
    dependsOn: Validate
    condition: succeeded()
    jobs:
      - deployment: DeployDevJob
        displayName: Deploy Storage Account
        environment: dev
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self
                - task: AzureCLI@2
                  displayName: Deploy Bicep
                  inputs:
                    azureSubscription: $(serviceConnection)
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az group create --name $(resourceGroup) --location $(location) --output none || true
                      az deployment group create \
                        --resource-group $(resourceGroup) \
                        --template-file infrastructure/storage.bicep \
                        --mode Incremental
EOF

git add pipelines/ci-pipeline.yml
git commit -m "Add CI/CD pipeline for Bicep deployment"
git push
```

---

## Step 5 — View Pipeline Runs

### Portal (GUI)
1. In Azure DevOps → **Pipelines** → click your pipeline
2. You see each run listed with:
   - Status: In progress / Succeeded / Failed
   - Trigger: what started it (push, manual, schedule)
   - Duration
3. Click a run → click a stage → click a job → see each step's output
4. When a step fails, click it — you see the exact error message and the log

### CLI
```bash
# List recent pipeline runs
az pipelines runs list \
  --project nhs-infra-lab \
  --top 5

# Get details of a specific run (replace with actual run ID)
az pipelines runs show \
  --project nhs-infra-lab \
  --id <run-id>
```

**What you just learned:** When a pipeline fails, the Portal shows you exactly which step failed and the full log output. Read the error, fix the YAML or template, push the fix — the pipeline re-runs automatically.

---

## Step 6 — Add an Approval Gate for Production

### Portal (GUI)
1. In Azure DevOps → **Environments** (under Pipelines)
2. Click **+ New environment** → name: `production` → **Create**
3. On the environment page, click the three-dot menu (top right) → **Approvals and checks**
4. Click **+** → **Approvals**
5. Add yourself as an approver
6. Set timeout: `24 hours`
7. Click **Create**

Now add a production stage to the pipeline that references this environment — the pipeline will pause and email you for approval before deploying.

```yaml
  - stage: DeployProd
    displayName: Deploy to Production
    dependsOn: DeployDev
    condition: succeeded()
    jobs:
      - deployment: DeployProdJob
        displayName: Deploy to Production
        environment: production        # <-- this triggers the approval gate
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self
                - task: AzureCLI@2
                  displayName: Deploy to Production
                  inputs:
                    azureSubscription: $(serviceConnection)
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      echo "Deploying to production after approval"
```

**What you just learned:** The `environment: production` field maps to the environment you created. Because you added an approval check, the pipeline pauses before this stage and sends a notification to the approver. The approver can view what changed and approve or reject. This satisfies ITIL Change Management requirements — changes to production must be authorised before they happen.

---

## Step 7 — Clean Up

### Portal (GUI)
Navigate to `rg-cicd-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-cicd-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| CI pipeline | Automatically validates code on every push |
| CD pipeline | Automatically deploys code after validation passes |
| YAML pipeline | Pipeline defined as code — stored in the repo, versioned |
| Service connection | Stored credentials allowing the pipeline to deploy to Azure |
| Environment | Named deployment target (dev, test, prod) — can have approval gates |
| Approval gate | Human sign-off required before deployment proceeds |
| `what-if` | Dry run — shows what *would* change without making changes |
| Stage → Job → Step | Pipeline hierarchy: stage contains jobs, jobs contain steps |

---

## Common Interview Questions

1. **"What is the difference between CI and CD?"** — CI (Continuous Integration) validates every code push automatically — compiles, lints, tests. CD (Continuous Deployment/Delivery) automatically deploys after CI passes. In the NHS, CD to production typically requires a human approval gate to satisfy change management requirements.

2. **"Why should infrastructure code be in version control and deployed via pipelines, not manually?"** — Version control provides an audit trail of every change (who changed what and when — required for NHS DSPT compliance). Pipelines enforce consistent process — no manual steps skipped. Pipelines can also enforce testing (lint, what-if) before any change reaches production.

3. **"A pipeline failed in the Deploy stage. How do you diagnose it?"** — Click the failed run → click the failed stage → click the failed step → read the log output. Common causes: service connection permissions expired, ARM/Bicep template has an error, resource quota exceeded, resource name conflict. Fix the root cause and re-run or push a fix.

---

## Next Steps
Move on to **Project 14 — Microsoft 365 & Hybrid Exchange**.
