# Project 06 — Azure Backup & Recovery Services Vault

## What You Will Learn
- What a Recovery Services Vault is and how it works
- How to configure and run VM backup via Portal and CLI
- How to create backup policies with custom schedules
- How to trigger on-demand backups and monitor jobs
- How to restore a VM from a recovery point
- RPO and RTO — what they are and why they matter in the NHS

---

## Background — Read This First

**Recovery Services Vault** — A management container in Azure where backup data and disaster recovery configurations live. Think of it as a secure, geo-redundant safe.

**Backup policy** — Defines how often to back up (daily, weekly), at what time, and how long to retain each recovery point.

**RPO (Recovery Point Objective)** — How much data can you afford to lose? A daily backup = RPO of 24 hours. If the server fails at 11pm, you lose up to 24 hours of data.

**RTO (Recovery Time Objective)** — How quickly must you be back online? Restoring from a VM backup typically takes 15–60 minutes depending on disk size.

In NHS environments, critical systems (patient record systems, clinical apps) have very tight RPO/RTO requirements — sometimes minutes. For those, Azure Site Recovery (continuous replication) is used instead of or alongside backup.

---

## Prerequisites
- Azure subscription, CLI logged in
- Completed Projects 01 and 02

---

## Step 1 — Create a Resource Group

### Portal (GUI)
Search **Resource groups** → **+ Create** → Name: `rg-backup-lab` | Region: `UK South` → **Create**

### CLI
```bash
az group create --name rg-backup-lab --location uksouth
```

---

## Step 2 — Create a Recovery Services Vault

### Portal (GUI)
1. Search **Recovery Services vaults** → **+ Create**
2. Fill in:
   - Resource group: `rg-backup-lab`
   - Vault name: `rsv-lab-vault`
   - Region: `UK South`
3. Click **Review + create** → **Create** → **Go to resource**
4. Look at the left panel — notice **Backup**, **Site Recovery**, **Backup policies**, **Backup jobs**

**Set storage redundancy (for lab, change to LRS to reduce cost):**
5. Click **Properties** in the left panel
6. Under **Backup Configuration**, click **Update**
7. Change to **Locally-redundant** → **Save**
8. In production, keep **Geo-redundant** (GRS)

### CLI
```bash
az backup vault create \
  --name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --location uksouth

# Set to LRS for lab (cheaper)
az backup vault backup-properties set \
  --name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --backup-storage-redundancy LocallyRedundant
```

**What you just learned:** The vault uses GRS by default — your backup data is automatically replicated to the UK West paired region. This means backup data survives even a full UK South datacentre failure. In production, always use GRS for NHS backups.

---

## Step 3 — Deploy a VM to Back Up

### Portal (GUI)
Create a VM named `vm-to-backup` in `rg-backup-lab` (Ubuntu, Standard_B1s) — refer to Project 02 for steps.

### CLI
```bash
az vm create \
  --resource-group rg-backup-lab \
  --name vm-to-backup \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

---

## Step 4 — View Built-In Backup Policies

### Portal (GUI)
1. In `rsv-lab-vault`, click **Backup policies** in the left panel
2. Click **DefaultPolicy** — examine the schedule:
   - Daily backup at 2:30 AM UTC
   - Retain daily backups for 30 days
3. This is suitable for general workloads — for critical systems you would create a custom policy

### CLI
```bash
az backup policy list \
  --vault-name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --output table

az backup policy show \
  --name DefaultPolicy \
  --vault-name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --output json
```

**What you just learned:** The DefaultPolicy backs up once per day and keeps 30 recovery points. For critical NHS systems, you would create a GrandFather-Father-Son policy: daily backups retained for 4 weeks, weekly backups for 12 months, monthly backups for 5 years. Long-term retention is a regulatory requirement.

---

## Step 5 — Enable Backup for the VM

### Portal (GUI)
**Method A — From the vault:**
1. In `rsv-lab-vault`, click **Backup** in the left panel
2. Select: Where is your workload running? **Azure** | What do you want to back up? **Virtual machine**
3. Click **Backup**
4. Select policy: `DefaultPolicy`
5. Click **Add** under Virtual machines → find `vm-to-backup` → click **OK**
6. Click **Enable Backup**

**Method B — From the VM directly:**
1. Navigate to `vm-to-backup`
2. In the left panel, click **Backup** (under Operations)
3. Select the vault `rsv-lab-vault` and policy `DefaultPolicy`
4. Click **Enable Backup**

### CLI
```bash
az backup protection enable-for-vm \
  --resource-group rg-backup-lab \
  --vault-name rsv-lab-vault \
  --vm vm-to-backup \
  --policy-name DefaultPolicy

# Verify backup is enabled
az backup item list \
  --vault-name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --workload-type VM \
  --output table
```

**What you just learned:** Enabling protection registers the VM with the vault and associates it with the policy. Azure automatically runs backups on the schedule. The VM does not restart during a backup — a consistent snapshot is taken while it runs.

---

## Step 6 — Trigger an On-Demand (Immediate) Backup

Before a major change (like a big OS update or configuration change), always take a manual backup first.

### Portal (GUI)
1. Navigate to `vm-to-backup` → click **Backup** in the left panel
2. Click **Backup now** at the top
3. Confirm the retain-until date → click **OK**
4. A backup job starts — click **View all jobs** to watch it progress

### CLI
```bash
az backup protection backup-now \
  --resource-group rg-backup-lab \
  --vault-name rsv-lab-vault \
  --item-name vm-to-backup \
  --backup-management-type AzureIaasVM \
  --retain-until "2026-12-31"
```

**What you just learned:** On-demand backups are your safety net before risky changes. In ITIL change management, a pre-change backup is mandatory for high-risk changes. The retain-until date overrides the policy for this specific recovery point.

---

## Step 7 — Monitor Backup Jobs

### Portal (GUI)
1. In `rsv-lab-vault`, click **Backup jobs** in the left panel
2. You see all recent jobs with status (In progress / Completed / Failed) and duration
3. Click a job to see its details: which VM, what type (initial backup or incremental), warnings
4. In a real environment, you would check this dashboard every morning

### CLI
```bash
az backup job list \
  --vault-name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --output table

# Get the latest job ID
JOB_ID=$(az backup job list \
  --vault-name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --query "[0].name" \
  --output tsv)

az backup job show \
  --vault-name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --name $JOB_ID \
  --output json
```

**What you just learned:** Monitoring backup jobs is daily ops. A failing backup is a P2 or P3 incident — not urgent, but it must be resolved before the next scheduled backup. Consistent backup failures mean you have no recovery capability.

---

## Step 8 — List Recovery Points

Once a backup completes, a recovery point is created. These are your restore targets.

### Portal (GUI)
1. In `rsv-lab-vault`, click **Backup items** in the left panel → **Azure Virtual Machine**
2. Click `vm-to-backup`
3. Click **Recovery points** — each backup run creates one entry with a timestamp and type (crash-consistent or application-consistent)

### CLI
```bash
az backup recoverypoint list \
  --vault-name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --container-name "IaasVMContainer;iaasvmcontainerv2;rg-backup-lab;vm-to-backup" \
  --item-name "VM;iaasvmcontainerv2;rg-backup-lab;vm-to-backup" \
  --backup-management-type AzureIaasVM \
  --output table
```

---

## Step 9 — Restore a VM (Conceptual Walkthrough)

Restoration creates a **new** VM from the recovery point — it does NOT overwrite the original.

### Portal (GUI)
1. In the VM's **Backup** blade, click **Restore VM**
2. Select a recovery point (choose the most recent)
3. **Restore type:** Choose between:
   - **Create new VM** — creates a completely new VM from the backup
   - **Replace existing disk** — replaces the OS disk in place (VM must be stopped)
4. Fill in the new VM name, resource group, VNet details
5. Click **Restore** — a restore job starts (takes 15–60 minutes)

### CLI (reference — not run to avoid cost)
```bash
# Get the recovery point name
RECOVERY_POINT=$(az backup recoverypoint list \
  --vault-name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --container-name "IaasVMContainer;iaasvmcontainerv2;rg-backup-lab;vm-to-backup" \
  --item-name "VM;iaasvmcontainerv2;rg-backup-lab;vm-to-backup" \
  --backup-management-type AzureIaasVM \
  --query "[0].name" --output tsv)

echo "Recovery Point: $RECOVERY_POINT"

# Restore command (structure shown for reference)
# az backup restore restore-azurevm \
#   --vault-name rsv-lab-vault \
#   --resource-group rg-backup-lab \
#   --container-name "IaasVMContainer;..." \
#   --item-name "VM;..." \
#   --rp-name $RECOVERY_POINT \
#   --restore-mode CreateNewVM \
#   --target-resource-group rg-backup-lab \
#   --target-vm-name vm-restored
```

**What you just learned:** Always restore to a NEW VM first. Validate the restored VM works before decommissioning the broken original. This gives you a safe rollback option if the restore has unexpected issues.

---

## Step 10 — Enable Soft Delete on the Vault

### Portal (GUI)
1. In `rsv-lab-vault`, click **Properties** → **Security Settings** → **Update**
2. Enable **Soft delete** → **Save**
3. With soft delete on, backup data is retained for 14 days after you stop protection — it cannot be permanently deleted during this period

### CLI
```bash
az backup vault backup-properties set \
  --name rsv-lab-vault \
  --resource-group rg-backup-lab \
  --soft-delete-feature-state Enable
```

**What you just learned:** Soft delete protects against accidental or malicious deletion of backup data. A threat actor who gains access to Azure cannot destroy your recovery capability during the 14-day window. Always enable this on production vaults.

---

## Step 11 — Clean Up

### Portal (GUI)
Navigate to `rg-backup-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-backup-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Recovery Services Vault | Container for backup data and configurations |
| Backup policy | Schedule + retention rules for automatic backups |
| Recovery point | A single point-in-time snapshot to restore from |
| RPO | Max acceptable data loss — drives backup frequency |
| RTO | Max time to restore — drives backup architecture choice |
| On-demand backup | Immediate backup triggered manually — use before risky changes |
| Soft delete | 14-day protection against deletion of backup data |
| GRS vault | Backup data replicated to UK West — survives regional outage |

---

## Common Interview Questions

1. **"What is the difference between RPO and RTO?"** — RPO is how much data you can afford to lose — it drives backup frequency (daily backup = 24-hour RPO). RTO is how quickly you must be back online — it drives the recovery architecture (VM restore from backup takes 30+ mins; Azure Site Recovery can be minutes).

2. **"A database server was corrupted at 3pm. Your last backup was 2am. What is the data loss?"** — 13 hours of data. That is why critical databases also use transaction log backups every 15 minutes in addition to nightly full backups — bringing the RPO down to 15 minutes.

3. **"What is the difference between Azure Backup and Azure Site Recovery?"** — Azure Backup is point-in-time recovery — you restore a VM or file to a previous state. Azure Site Recovery is continuous replication — it mirrors a VM to another region in near-real-time. Use Backup for data recovery; use Site Recovery for disaster recovery with near-zero RPO.

---

## Next Steps
Move on to **Project 07 — Windows Server Administration**.
