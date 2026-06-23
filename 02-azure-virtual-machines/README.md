# Project 02 — Azure Virtual Machines (IaaS)

## What You Will Learn
- How to create Windows and Linux VMs via the Portal and CLI
- How to connect to VMs (SSH, RDP, Azure Bastion)
- How to stop, start, deallocate, and resize VMs
- How to attach and manage data disks
- The difference between deallocate and stop (critical for billing)

---

## Background — Read This First

A **Virtual Machine (VM)** is a software-based computer running on Azure hardware. This is IaaS — Azure manages the hardware, you manage everything from the OS upward.

Key concepts:
- **VM Size** — defines CPU and RAM (e.g. `Standard_B2s` = 2 vCPUs, 4 GB RAM)
- **OS Disk** — created automatically; holds the operating system
- **Data Disk** — additional storage you attach for data (separate from OS)
- **Deallocate vs Stop** — "Stop" in the portal still charges for the VM. **Deallocate** releases the hardware and stops compute billing. Always deallocate lab VMs.
- **NSG** — virtual firewall; all inbound traffic is blocked by default

---

## Prerequisites
- Azure subscription
- Azure CLI installed and logged in
- SSH key pair (Step 1 below)

---

## Step 1 — Generate an SSH Key Pair

You need this before creating a Linux VM.

### Portal (GUI)
The Portal can generate a key pair for you during VM creation (Step 3). You download the private key at creation time — it cannot be downloaded again later.

### CLI
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/azure_lab_key -N ""
# View your public key (this goes to Azure)
cat ~/.ssh/azure_lab_key.pub
```

---

## Step 2 — Create a Resource Group

### Portal (GUI)
1. Search **Resource groups** → click **+ Create**
2. Name: `rg-vm-lab` | Region: `UK South`
3. Click **Review + create** → **Create**

### CLI
```bash
az group create --name rg-vm-lab --location uksouth
```

---

## Step 3 — Create a Linux VM

### Portal (GUI)
1. Search **Virtual machines** in the top bar → click **+ Create** → **Azure virtual machine**
2. Fill in the **Basics** tab:
   - Resource group: `rg-vm-lab`
   - VM name: `vm-linux-lab`
   - Region: `(Europe) UK South`
   - Image: `Ubuntu Server 22.04 LTS`
   - Size: click **See all sizes** → search `B1s` → select `Standard_B1s`
   - Authentication type: **SSH public key**
   - Username: `azureuser`
   - SSH public key source: **Use existing public key**
   - Paste the contents of `~/.ssh/azure_lab_key.pub`
3. **Disks tab:** leave defaults (Standard SSD for OS disk)
4. **Networking tab:**
   - A new VNet, subnet, public IP, and NSG are created automatically — notice the names
   - Under **Public inbound ports**, select **SSH (22)**
5. Click **Review + create** → **Create**
6. After deployment: click **Go to resource** — note the **Public IP address** on the Overview page

### CLI
```bash
az vm create \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-value ~/.ssh/azure_lab_key.pub \
  --public-ip-sku Standard

# Get the public IP
LINUX_IP=$(az vm show \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --show-details \
  --query publicIps \
  --output tsv)
echo "VM IP: $LINUX_IP"
```

**What you just learned:** `Standard_B1s` is the cheapest burstable size — 1 vCPU, 1 GB RAM. Fine for a lab. The portal creates supporting resources (VNet, NSG, NIC, public IP) automatically. The CLI does the same when you don't specify them.

---

## Step 4 — Connect to the Linux VM via SSH

### Portal (GUI)
**Option A — Azure Cloud Shell (no local SSH needed):**
1. Click the **Cloud Shell** icon (`>_`) in the top toolbar
2. Choose **Bash**
3. Run: `ssh -i ~/.ssh/azure_lab_key azureuser@<YOUR_VM_IP>`

**Option B — Connect via the VM blade:**
1. On the VM's Overview page, click **Connect** at the top
2. Select **SSH** → follow the displayed command
3. Download the private key if you used portal-generated key pair

### CLI
```bash
ssh -i ~/.ssh/azure_lab_key azureuser@$LINUX_IP
```

Once inside, run some checks:
```bash
cat /etc/os-release       # OS version
df -h                     # Disk space
free -h                   # Memory
nproc                     # CPU count
sudo apt update           # Update package list
exit                      # Leave the VM
```

---

## Step 5 — Run Commands Without SSH (Run Command)

### Portal (GUI)
1. On the VM's left panel, scroll to **Operations** → click **Run command**
2. Click **RunShellScript**
3. In the script box, type:
   ```bash
   hostname && uptime && df -h
   ```
4. Click **Run** — output appears at the bottom of the page within ~30 seconds

### CLI
```bash
az vm run-command invoke \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --command-id RunShellScript \
  --scripts "hostname && uptime && df -h"
```

**What you just learned:** Run Command is your emergency access tool. If you lock yourself out of SSH, or if a VM has no public IP, Run Command still works because it goes through the Azure VM Agent (not the network). Critical for troubleshooting.

---

## Step 6 — Check VM Status

### Portal (GUI)
1. Go to `rg-vm-lab` resource group
2. Click the VM `vm-linux-lab`
3. The **Overview** page shows: Status (Running/Stopped/Deallocated), Size, OS, Public IP
4. Click **Activity log** in the left panel to see who started/stopped/modified this VM and when

### CLI
```bash
# Power state
az vm get-instance-view \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --query "instanceView.statuses[1].displayStatus" \
  --output tsv

# Full overview
az vm show \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --output table

# List all VMs in the group
az vm list --resource-group rg-vm-lab --show-details --output table
```

---

## Step 7 — Stop vs Deallocate (Critical Billing Concept)

### Portal (GUI)
**Stop (still billed):**
1. On the VM Overview, click **Stop** in the top toolbar
2. Notice the status changes to **Stopped (deallocated)** — wait, the portal actually deallocates when you click Stop!
3. However: if you use the OS shutdown command (`sudo shutdown now`) from inside the VM, the status becomes **Stopped** (not deallocated) and you ARE still billed

**Deallocate properly:**
- Always use the portal **Stop** button or CLI `deallocate` — not the OS shutdown

**Start:**
1. Click **Start** in the top toolbar

### CLI
```bash
# Deallocate (stops billing for compute)
az vm deallocate --resource-group rg-vm-lab --name vm-linux-lab

# Check it is deallocated
az vm get-instance-view \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --query "instanceView.statuses[1].displayStatus" \
  --output tsv

# Start again
az vm start --resource-group rg-vm-lab --name vm-linux-lab
```

**What you just learned:** This is the #1 billing mistake junior engineers make. If you SSH into a Linux VM and run `sudo shutdown now`, the VM's OS shuts down but the Azure compute resource is still allocated — you are still billed. Always use the Portal Stop button or `az vm deallocate`. OS disk storage is billed regardless of power state.

---

## Step 8 — Resize a VM

### Portal (GUI)
1. First **Stop** (deallocate) the VM
2. In the left panel, click **Size** (under Settings)
3. Browse available sizes — they are grouped by family (B-series for burstable, D-series for general, E-series for memory)
4. Select `Standard_B2s` and click **Resize**
5. The VM restarts automatically
6. Start it again when done

### CLI
```bash
# Check available sizes
az vm list-vm-resize-options \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --output table

# Resize (briefly restarts the VM)
az vm resize \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --size Standard_B2s

# Verify
az vm show \
  --resource-group rg-vm-lab \
  --name vm-linux-lab \
  --query hardwareProfile.vmSize \
  --output tsv
```

**What you just learned:** Resizing requires a brief restart — plan this during a maintenance window in production. Not all sizes are available in every region. The Portal shows you only sizes available in your region; the CLI does the same with `list-vm-resize-options`.

---

## Step 9 — Attach a Data Disk

Keep OS and data separate. Put application data on a dedicated data disk.

### Portal (GUI)
1. On the VM's left panel, click **Disks** (under Settings)
2. Click **+ Create and attach a new disk**
3. Fill in:
   - Disk name: `disk-data-lab`
   - Size: `32 GiB`
   - Disk type: `Standard HDD` (cheapest for a lab)
4. Click **Apply**
5. The disk is attached but not yet formatted — you must initialise it inside the VM (see below)

### CLI
```bash
az vm disk attach \
  --resource-group rg-vm-lab \
  --vm-name vm-linux-lab \
  --name disk-data-lab \
  --size-gb 32 \
  --sku Standard_LRS \
  --new
```

Now initialise the disk inside the VM:
```bash
ssh -i ~/.ssh/azure_lab_key azureuser@$LINUX_IP

lsblk                                          # Find the new disk (e.g. /dev/sdc)
sudo parted /dev/sdc --script mklabel gpt mkpart xfspart xfs 0% 100%
sudo mkfs.xfs /dev/sdc1
sudo mkdir /datadrive
sudo mount /dev/sdc1 /datadrive
df -h /datadrive                               # Confirm it is mounted
# Make permanent (survives reboots)
sudo bash -c 'echo "/dev/sdc1 /datadrive xfs defaults,nofail 1 2" >> /etc/fstab'
exit
```

**What you just learned:** Managed disks are Azure-managed storage — you don't deal with the underlying storage account. The `nofail` option in `/etc/fstab` is critical: without it, the VM will hang at boot if the disk has a problem.

---

## Step 10 — Create a Windows Server VM

### Portal (GUI)
1. Search **Virtual machines** → **+ Create** → **Azure virtual machine**
2. Basics:
   - Resource group: `rg-vm-lab`
   - VM name: `vm-windows-lab`
   - Image: `Windows Server 2022 Datacenter`
   - Size: `Standard_B2s`
   - Authentication: **Password**
   - Username: `winadmin`
   - Password: `WinL@b2026!!`
3. **Networking tab:** under Public inbound ports, select **RDP (3389)**
4. Click **Review + create** → **Create**

**Connect via RDP from the Portal:**
1. Once deployed, click **Connect** → **RDP**
2. Download the `.rdp` file
3. Open it — enter username `winadmin` and your password
4. Click through the certificate warning

### CLI
```bash
az vm create \
  --resource-group rg-vm-lab \
  --name vm-windows-lab \
  --image Win2022Datacenter \
  --size Standard_B2s \
  --admin-username winadmin \
  --admin-password "WinL@b2026!!" \
  --public-ip-sku Standard

az vm open-port \
  --resource-group rg-vm-lab \
  --name vm-windows-lab \
  --port 3389

WIN_IP=$(az vm show \
  --resource-group rg-vm-lab \
  --name vm-windows-lab \
  --show-details \
  --query publicIps \
  --output tsv)
echo "Windows VM IP: $WIN_IP"

# Connect from Windows
mstsc /v:$WIN_IP
```

> **Security note:** In a real environment, never expose RDP directly to the internet. Use Azure Bastion or a VPN.

---

## Step 11 — Clean Up

### Portal (GUI)
1. Navigate to `rg-vm-lab`
2. Click **Delete resource group**
3. Type the name to confirm → **Delete**

### CLI
```bash
az group delete --name rg-vm-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| IaaS | You manage OS upward; Azure manages hardware |
| Deallocate | Stops compute billing — different from OS shutdown |
| VM Size | CPU + RAM; bigger = more cost |
| Managed Disk | Azure-managed disk storage — no storage account to handle |
| Run Command | Execute scripts on a VM without SSH/RDP — emergency access |
| Data Disk | Separate from OS disk — always separate OS from data |
| NSG | Firewall controlling inbound/outbound traffic |

---

## Common Interview Questions

1. **"What is the difference between stopping and deallocating a VM?"** — OS shutdown or clicking Stop inside the OS leaves the compute allocated — you are still billed for the VM size. Deallocating via the portal Stop button or `az vm deallocate` releases the compute hardware — billing stops. OS disk storage costs continue regardless.

2. **"How would you connect to a VM that you are locked out of?"** — Use **Run Command** in the Portal (or `az vm run-command invoke` in CLI) to run scripts through the VM Agent without needing SSH/RDP. You can also reset credentials via the VM's **Reset password** blade.

3. **"Why should you never expose RDP to the internet?"** — Port 3389 is constantly scanned by bots running brute-force attacks. In production, use Azure Bastion (a managed jump host) or a VPN gateway, and keep RDP restricted to internal networks only.

---

## Next Steps
Move on to **Project 03 — Azure Storage Accounts**.
