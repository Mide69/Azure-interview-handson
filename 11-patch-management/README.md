# Project 11 — Patch Management

## What You Will Learn
- Why patch management is critical for NHS infrastructure
- How to use Azure Update Manager via Portal and CLI
- How to assess patch compliance, schedule maintenance windows, and deploy patches
- How to patch Linux servers from the command line
- How WSUS works for on-premises Windows environments

---

## Background — Read This First

Patch management is the process of identifying, testing, and applying software updates to systems. In the NHS, unpatched systems are a direct patient safety risk — the 2017 WannaCry attack exploited an unpatched Windows vulnerability and shut down large parts of the NHS.

NHS Digital/England mandates via the **Data Security and Protection Toolkit (DSPT)**: Critical and Security patches must be applied within 14 days of release.

**Patching approaches:**
| Tool | Where | Use For |
|---|---|---|
| Azure Update Manager | Azure Portal | All Azure VMs — no agent required |
| WSUS | On-premises | Windows servers not in Azure |
| apt/unattended-upgrades | Linux CLI | Linux servers |
| Intune | Microsoft 365 | Windows workstations |

---

## Prerequisites
- Azure subscription, CLI logged in
- Two VMs deployed (Windows and Linux) — see Step 1

---

## Step 1 — Deploy VMs for the Lab

### Portal (GUI)
Create two VMs in resource group `rg-patch-lab` (UK South):
- `vm-windows-patch`: Windows Server 2022 Datacenter, Standard_B2s
- `vm-linux-patch`: Ubuntu 22.04, Standard_B1s

### CLI
```bash
az group create --name rg-patch-lab --location uksouth

az vm create \
  --resource-group rg-patch-lab \
  --name vm-windows-patch \
  --image Win2022Datacenter \
  --size Standard_B2s \
  --admin-username patchadmin \
  --admin-password "PatchL@b2026!!" \
  --public-ip-sku Standard

az vm create \
  --resource-group rg-patch-lab \
  --name vm-linux-patch \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

---

## Step 2 — Azure Update Manager: Enable Assessment

Assessment scans for available patches **without making any changes**.

### Portal (GUI)
1. Search **Azure Update Manager** in the top bar → click it
2. You land on the Update Manager dashboard — it shows all your VMs and their patch status
3. Click **Machines** in the left panel — find `vm-windows-patch`
4. Select both VMs → click **Check for updates** at the top
5. Wait 2–3 minutes → the Status column shows **Assessed** with a count of pending updates

### CLI
```bash
az vm assess-patches \
  --resource-group rg-patch-lab \
  --name vm-windows-patch

az vm assess-patches \
  --resource-group rg-patch-lab \
  --name vm-linux-patch
```

**What you just learned:** Assessment is always the first step. Never deploy patches without knowing what will change. The Portal dashboard gives you a fleet-wide compliance view at a glance.

---

## Step 3 — View Assessment Results

### Portal (GUI)
1. In Azure Update Manager → **Machines** → click `vm-windows-patch`
2. Click the **Pending updates** tab — you see each available patch with:
   - Patch name and KB number
   - Classification: Critical / Security / Update / Feature Pack
   - Reboot required: yes/no
3. Click **Classification** to sort by Critical first — these are your priority

### CLI
```bash
az vm list-patches \
  --resource-group rg-patch-lab \
  --name vm-windows-patch \
  --output table

az vm list-patches \
  --resource-group rg-patch-lab \
  --name vm-linux-patch \
  --output table
```

**What you just learned:** Focus on Critical and Security patches first — these are exploitable vulnerabilities. Other categories (Update Rollups, Feature Packs) can be scheduled in the next maintenance window.

---

## Step 4 — Deploy Patches Now (On-Demand)

Useful when you need to respond to an emergency vulnerability.

### Portal (GUI)
1. In Update Manager → **Machines** → select `vm-linux-patch`
2. Click **One-time update** at the top
3. Select the VM → **Next**
4. Select patch classifications: tick **Critical** and **Security**
5. Set **Reboot option**: `IfRequired`
6. Set **Maintenance window**: 1 hour
7. Click **Review + install** → **Install**
8. A deployment job starts — click **View deployment** to monitor progress

### CLI
```bash
az vm install-patches \
  --resource-group rg-patch-lab \
  --name vm-linux-patch \
  --maximum-duration PT1H \
  --reboot-setting IfRequired \
  --linux-parameters '{"classificationsToInclude": ["Critical","Security"]}'

az vm install-patches \
  --resource-group rg-patch-lab \
  --name vm-windows-patch \
  --maximum-duration PT1H \
  --reboot-setting IfRequired \
  --windows-parameters '{"classificationsToInclude": ["Critical","Security"]}'
```

**What you just learned:** `IfRequired` reboots the VM only if a patch needs it — minimises downtime. `PT1H` is ISO 8601 for "1 hour" — the maximum time the patching operation runs before stopping.

---

## Step 5 — Create a Maintenance Configuration (Scheduled Patching)

Schedule patches to deploy automatically in a maintenance window — without manual intervention.

### Portal (GUI)
1. In Azure Update Manager → **Maintenance Configurations** in the left panel → **+ Create**
2. Fill in:
   - Resource group: `rg-patch-lab`
   - Name: `mc-sunday-patching`
   - Region: `UK South`
   - Maintenance scope: **Guest (Azure VM)**
3. **Schedule** tab:
   - Start: `2026-07-06 02:00`
   - Repeat: **Week** | Day: **Sunday**
   - Duration: `3 hours`
   - Time zone: `(UTC+00:00) Greenwich Mean Time`
4. **Updates** tab:
   - Include classifications: tick **Critical** and **Security**
   - Reboot: `IfRequired`
5. Click **Review + create** → **Create**

**Assign VMs to the maintenance configuration:**
6. Click the new maintenance config → **Add machines** → select both VMs → **Add**

### CLI
```bash
az maintenance configuration create \
  --resource-group rg-patch-lab \
  --name mc-sunday-patching \
  --maintenance-scope InGuestPatch \
  --location uksouth \
  --recur-every "Week Sunday" \
  --start-date-time "2026-07-06 02:00" \
  --duration "03:00" \
  --time-zone "GMT Standard Time" \
  --extension-properties '{"InGuestPatchMode":"User"}' \
  --install-patches-linux-parameters '{"classificationsToInclude": ["Critical","Security"]}' \
  --install-patches-windows-parameters '{"classificationsToInclude": ["Critical","Security"]}' \
  --reboot-setting "IfRequired"
```

**What you just learned:** Maintenance windows are a core ITIL concept. In an NHS trust, all patching must be done in agreed maintenance windows (usually Sunday nights) unless it is an emergency change. Scheduled maintenance configurations enforce this automatically.

---

## Step 6 — Patch Compliance Dashboard

### Portal (GUI)
1. In Azure Update Manager → **Overview** dashboard
2. The top section shows: **Machines needing attention**, **Pending updates by classification**, **Patch compliance percentage**
3. Click **Update compliance** in the left panel for a detailed compliance report across all subscriptions
4. You can filter by subscription, resource group, OS type, or classification

This is the dashboard you check every Monday morning to confirm Sunday's patching succeeded.

---

## Step 7 — Linux Patching from Inside the VM

SSH into the Linux VM:
```bash
LINUX_IP=$(az vm show \
  --resource-group rg-patch-lab --name vm-linux-patch \
  --show-details --query publicIps --output tsv)
ssh -i ~/.ssh/azure_lab_key azureuser@$LINUX_IP
```

Inside the VM:
```bash
# Check what needs updating
sudo apt update
apt list --upgradeable 2>/dev/null | wc -l && echo "packages available"

# Show security updates only
apt list --upgradeable 2>/dev/null | grep -i security

# Install all updates
sudo apt upgrade -y

# Check if a reboot is required
if [ -f /var/run/reboot-required ]; then
  echo "REBOOT REQUIRED"
  cat /var/run/reboot-required.pkgs
else
  echo "No reboot required"
fi

# View what was installed recently
grep "installed" /var/log/dpkg.log | tail -20

exit
```

**What you just learned:** `/var/run/reboot-required` is created automatically when a kernel update needs a reboot. Always check this after patching. Schedule the reboot during the maintenance window.

---

## Step 8 — Configure Automatic Security Updates on Linux

```bash
az vm run-command invoke \
  --resource-group rg-patch-lab --name vm-linux-patch \
  --command-id RunShellScript \
  --scripts "
    apt-get install -y unattended-upgrades

    cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'CONF'
Unattended-Upgrade::Allowed-Origins {
    \"\${distro_id}:\${distro_codename}-security\";
};
Unattended-Upgrade::Automatic-Reboot \"false\";
Unattended-Upgrade::Automatic-Reboot-Time \"02:00\";
CONF

    cat > /etc/apt/apt.conf.d/20auto-upgrades << 'CONF'
APT::Periodic::Update-Package-Lists \"1\";
APT::Periodic::Unattended-Upgrade \"1\";
APT::Periodic::AutocleanInterval \"7\";
CONF

    echo 'Auto security updates configured'
  "
```

**What you just learned:** `Automatic-Reboot "false"` prevents the server from rebooting unexpectedly. Change to `true` with a time only after confirming what services are running and that a reboot at that time is acceptable.

---

## Step 9 — WSUS Overview (On-Premises Windows Patching)

WSUS (Windows Server Update Services) is the on-premises tool for managing Windows patches. It:
1. Downloads patches once from Microsoft, then distributes to all internal servers (saves internet bandwidth)
2. Lets you **approve or deny** specific patches before they deploy
3. Groups computers into collections so you patch in waves

**Install WSUS (run on a dedicated server — not the DC):**
```powershell
# On a Windows Server VM via PowerShell
Install-WindowsFeature -Name UpdateServices, UpdateServices-WidDB -IncludeManagementTools

# Post-install configuration
& 'C:\Program Files\Update Services\Tools\WsusUtil.exe' postinstall CONTENT_DIR=C:\WSUS

# Check the WSUS service
Get-Service WsusService
```

**Staged patching approach (Grandfather principle):**
1. **Pilot group** (5–10 test machines) — patch first, wait 48 hours
2. **Wave 1** (~20% of production) — patch, monitor for 24 hours
3. **Wave 2** (remaining 80% of production) — only if Wave 1 succeeded

---

## Step 10 — Clean Up

### Portal (GUI)
Navigate to `rg-patch-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-patch-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Assessment | Read-only scan — shows what patches are available, makes no changes |
| Maintenance window | Agreed time slot for patching — usually Sunday 02:00–05:00 |
| Critical/Security | Must apply within 14 days (NHS DSPT requirement) |
| `IfRequired` reboot | Reboot only if the patch needs it — minimises downtime |
| WSUS | On-prem Windows update server — approve, download once, distribute |
| `/var/run/reboot-required` | Linux file created when a kernel patch needs a reboot |
| Staged rollout | Pilot → Wave 1 → Wave 2 — prevents a bad patch taking everything down |

---

## Common Interview Questions

1. **"A critical zero-day vulnerability has just been published. How do you respond?"** — Immediately assess which systems are affected (`az vm assess-patches` or check WSUS). Raise an emergency change request. Test the patch on a non-production system if possible (even 30 minutes). Deploy in an emergency maintenance window with rollback plan (snapshots). Document all actions.

2. **"What is the difference between Azure Update Manager and WSUS?"** — Azure Update Manager is cloud-native and agentless for Azure VMs — managed from the Portal. WSUS is on-premises, requires a dedicated server, and works for machines that can't reach the internet directly. In a hybrid NHS environment you would use both.

3. **"How do you ensure all 500 NHS servers are patched within the monthly window?"** — Use Azure Update Manager maintenance configurations to group servers and patch them in waves (non-production Sunday night, production the following Sunday). Track compliance in the Update Manager dashboard. Escalate any VMs that fail or need manual intervention.

---

## Next Steps
Move on to **Project 12 — Azure Monitor & Alerting**.
