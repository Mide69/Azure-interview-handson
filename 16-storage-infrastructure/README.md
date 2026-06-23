# Project 16 — Storage Infrastructure (SAN, NAS, RAID & Azure Disks)

## What You Will Learn
- The difference between SAN, NAS, and DAS — and when each is used
- RAID levels and which to choose for different scenarios
- How Azure managed disks work (Portal and CLI)
- How to expand a disk, manage LVM on Linux, and snapshot disks
- How to identify and fix common storage problems

---

## Background — Read This First

Storage infrastructure is one of the most critical areas for NHS systems — clinical records, imaging (DICOM files are enormous), lab results, and prescriptions all need reliable, fast, and resilient storage.

**The three storage types:**

| Type | Full Name | Connection | Best For |
|---|---|---|---|
| **DAS** | Direct Attached Storage | Direct cable to server | Small workloads, single server |
| **NAS** | Network Attached Storage | Ethernet (NFS/SMB) | File sharing across many users |
| **SAN** | Storage Area Network | Fibre Channel or iSCSI | Databases, virtualisation, high performance |

**The NHS and SAN:** Large NHS trusts run SAN arrays (NetApp, Pure Storage, HPE 3PAR) for their VMware datastores and SQL databases. Virtual machines live on SAN LUNs. The SAN provides thin-provisioning, snapshots, and replication to DR sites.

---

## RAID Levels — Learn These Cold

| Level | Description | Min Disks | Can Lose | Use Case |
|---|---|---|---|---|
| **RAID 0** | Striping — data split across disks for speed | 2 | 0 | Fast scratch storage — NO redundancy |
| **RAID 1** | Mirroring — identical copy on each disk | 2 | 1 | Boot volumes, OS drives |
| **RAID 5** | Striping + 1 parity disk | 3 | 1 | General-purpose — NHS file servers |
| **RAID 6** | Striping + 2 parity disks | 4 | 2 | Critical data — tolerates 2 simultaneous disk failures |
| **RAID 10** | Mirrored stripes | 4 | 1 per mirror pair | Databases — high performance + redundancy |

**NHS standard:** RAID 5 or RAID 6 for file servers and application data. RAID 10 for SQL databases (because RAID 10 has better write performance). RAID 0 is never used in production — one disk failure means complete data loss.

---

## Prerequisites
- Azure subscription, CLI logged in
- An Ubuntu VM — see Step 1

---

## Step 1 — Deploy a Linux VM for Disk Practice

### Portal (GUI)
Create `vm-storage` in `rg-storage-lab` (Ubuntu 22.04, Standard_B2s). See Project 02 for full steps.

### CLI
```bash
az group create --name rg-storage-lab --location uksouth

az vm create \
  --resource-group rg-storage-lab \
  --name vm-storage \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

VM_IP=$(az vm show \
  --resource-group rg-storage-lab --name vm-storage \
  --show-details --query publicIps --output tsv)
echo "VM IP: $VM_IP"
```

---

## Step 2 — Attach a Managed Disk

### Portal (GUI)
1. Navigate to `vm-storage` → **Disks** (under Settings in the left panel)
2. Click **+ Create and attach a new disk**
3. Fill in:
   - Name: `disk-data-01`
   - Storage type: **Standard SSD**
   - Size: `32 GiB`
4. Click **Save** (at the top of the Disks blade)
5. The disk is hot-attached — no reboot needed

### CLI
```bash
az vm disk attach \
  --resource-group rg-storage-lab \
  --vm-name vm-storage \
  --name disk-data-01 \
  --size-gb 32 \
  --sku StandardSSD_LRS \
  --new

# List all disks on the VM
az vm show \
  --resource-group rg-storage-lab \
  --name vm-storage \
  --query "storageProfile.dataDisks" \
  --output table
```

---

## Step 3 — Partition, Format, and Mount (Inside the VM)

```bash
ssh -i ~/.ssh/azure_lab_key azureuser@$VM_IP
```

Inside the VM:
```bash
# List available disks — find your new 32GB disk
lsblk
# You should see sdb with 32G and no partitions

# Identify the disk
ls /dev/sd*

# Create a partition table and partition
sudo fdisk /dev/sdb
# In fdisk: n (new partition), p (primary), 1, enter, enter, w (write)

# Verify the partition was created
lsblk /dev/sdb

# Format with ext4 filesystem
sudo mkfs.ext4 /dev/sdb1

# Create a mount point
sudo mkdir -p /data/nhs-data

# Mount the disk
sudo mount /dev/sdb1 /data/nhs-data

# Verify mount
df -h /data/nhs-data

# Make it persistent across reboots (add to /etc/fstab)
sudo blkid /dev/sdb1
# Note the UUID from the output, e.g. UUID="abc123..."

# Add to /etc/fstab (replace UUID with yours)
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb1) /data/nhs-data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab

# Test the fstab entry
sudo umount /data/nhs-data
sudo mount -a
df -h /data/nhs-data
```

**What you just learned:** `/etc/fstab` controls which disks are automatically mounted at boot. The `nofail` option tells the system to continue booting even if this disk is not available — important for VMs that might restart without the data disk attached.

---

## Step 4 — LVM (Logical Volume Manager)

LVM lets you resize volumes without downtime and span volumes across multiple disks. Used in production Linux systems.

```bash
# Install LVM tools
sudo apt install -y lvm2

# Attach a second disk first (do this via Portal or CLI: disk-data-02, 32GB)
# Then inside the VM:

# Create physical volumes from two disks
sudo pvcreate /dev/sdb /dev/sdc
sudo pvs   # list physical volumes

# Create a volume group (pool of storage)
sudo vgcreate vg-nhsdata /dev/sdb /dev/sdc
sudo vgs   # list volume groups

# Create a logical volume (20GB from the pool)
sudo lvcreate -L 20G -n lv-records vg-nhsdata
sudo lvs   # list logical volumes

# Format and mount
sudo mkfs.ext4 /dev/vg-nhsdata/lv-records
sudo mkdir -p /data/records
sudo mount /dev/vg-nhsdata/lv-records /data/records
df -h /data/records

# Extend the logical volume by 10GB (without unmounting)
sudo lvextend -L +10G /dev/vg-nhsdata/lv-records
sudo resize2fs /dev/vg-nhsdata/lv-records
df -h /data/records
# The volume is now 30GB — no downtime, no umount
```

**What you just learned:** LVM is why Linux storage management is so flexible. In production, you create LVM volumes on SAN LUNs so you can grow them as clinical data grows, without scheduling downtime.

---

## Step 5 — Take a Disk Snapshot

Snapshots are point-in-time copies of a disk — your safety net before making risky changes.

### Portal (GUI)
1. Search **Snapshots** → **+ Create**
2. Resource group: `rg-storage-lab`
3. Name: `snap-disk-data-01`
4. Source type: **Disk**
5. Source disk: select `disk-data-01`
6. Snapshot type: **Full**
7. Click **Review + create** → **Create**

**Restore from snapshot:**
8. Navigate to the snapshot → **+ Create disk** → this creates a new managed disk from the snapshot
9. Attach the new disk to any VM

### CLI
```bash
DISK_ID=$(az disk show \
  --resource-group rg-storage-lab \
  --name disk-data-01 \
  --query id --output tsv)

az snapshot create \
  --resource-group rg-storage-lab \
  --name snap-disk-data-01 \
  --source $DISK_ID \
  --location uksouth

# Restore: create a new disk from the snapshot
SNAPSHOT_ID=$(az snapshot show \
  --resource-group rg-storage-lab \
  --name snap-disk-data-01 \
  --query id --output tsv)

az disk create \
  --resource-group rg-storage-lab \
  --name disk-restored-from-snap \
  --source $SNAPSHOT_ID \
  --location uksouth
```

**What you just learned:** Always snapshot before patching, resizing, or changing disk configuration. The snapshot is your rollback point. In Azure, snapshots are incremental after the first — they only store changed blocks.

---

## Step 6 — Expand a Disk While the VM Runs

### Portal (GUI)
1. Navigate to `vm-storage` → **Disks** → click `disk-data-01`
2. Click **Size + performance** in the left panel
3. Change size from `32 GiB` to `64 GiB`
4. Click **Save** — the disk is expanded in Azure (the OS still sees the old size)

Inside the VM, extend the filesystem:
```bash
# The disk was expanded in Azure — now tell Linux
sudo growpart /dev/sdb 1      # grow the partition to fill the disk
sudo resize2fs /dev/sdb1      # grow the filesystem to fill the partition
df -h /data/nhs-data          # confirm new size
```

### CLI
```bash
# Resize the disk in Azure
az disk update \
  --resource-group rg-storage-lab \
  --name disk-data-01 \
  --size-gb 64

# Then SSH into the VM and run growpart + resize2fs as above
```

---

## Step 7 — Azure Managed Disk Types Reference

| Type | Use Case | Cost |
|---|---|---|
| **Standard HDD** | Backups, test environments, infrequently accessed | Cheapest |
| **Standard SSD** | General purpose — most workloads | Mid |
| **Premium SSD** | Production databases, active workloads | Higher |
| **Premium SSD v2** | Performance-critical databases, high IOPS | Highest |
| **Ultra Disk** | Extreme IOPS — SAP, SQL on VMSS | Most expensive |

For NHS clinical systems: Standard SSD minimum for production. Premium SSD for databases and active imaging systems.

---

## Step 8 — Clean Up

```bash
exit  # leave the SSH session
```

### Portal (GUI)
Navigate to `rg-storage-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-storage-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| SAN | High-performance block storage shared across servers via Fibre Channel or iSCSI |
| NAS | File-level storage shared across the network via NFS or SMB |
| RAID 5 | 3+ disks, 1 parity — can lose 1 disk. NHS file servers |
| RAID 10 | Mirrored stripes — can lose 1 disk per pair. NHS databases |
| LVM | Logical Volume Manager — flexible, resizable volumes on Linux |
| Managed disk | Azure's block storage — like a virtual SAN LUN |
| Snapshot | Point-in-time copy of a disk — cheap, incremental after first |
| `nofail` fstab option | VM boots even if the disk is missing |

---

## Common Interview Questions

1. **"Which RAID level would you choose for a hospital SQL Server database and why?"** — RAID 10. It combines mirroring (redundancy — survive 1 disk failure per pair) with striping (performance — writes go to multiple disks in parallel). SQL databases do many small random writes, and RAID 5's parity calculation slows writes significantly. RAID 10's write penalty is minimal.

2. **"What is the difference between SAN and NAS?"** — SAN (Storage Area Network) presents raw block devices (LUNs) to servers over Fibre Channel or iSCSI — the server's OS handles the filesystem. Fast, used for VM datastores and databases. NAS (Network Attached Storage) presents ready-to-use file shares over NFS or SMB — multiple clients share the same files. Easier to manage but higher latency than SAN.

3. **"A disk on a production Linux server is at 95% capacity. How do you resolve it without downtime?"** — If using LVM: `lvextend` to add space from the volume group (or add a new disk with `pvcreate`/`vgextend` first), then `resize2fs` to grow the filesystem online. If using a bare partition: use Azure Portal to grow the managed disk, then `growpart` and `resize2fs` inside the VM — both can be done without unmounting on ext4 and xfs.

---

## Next Steps
Move on to **Project 17 — VMware vSphere Basics**.
