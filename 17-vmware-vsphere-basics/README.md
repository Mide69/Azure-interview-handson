# Project 17 — VMware vSphere Basics

## What You Will Learn
- Core VMware vSphere architecture and terminology
- How ESXi hosts, vCenter, clusters, and datastores relate to each other
- How to perform VM management tasks in the vSphere Client (GUI)
- How to use PowerCLI for scripted VMware administration
- VMware HA, DRS, vMotion, and FT — and when to use each
- How to take and manage VM snapshots

---

## Background — Read This First

The SLaM job spec explicitly requires VMware vSphere experience. NHS trusts run on-premises VMware infrastructure to host clinical systems — patient administration, imaging (PACS), electronic prescribing, and laboratory systems all typically run as VMware VMs.

**Architecture stack (bottom to top):**
```
Physical hardware (servers, storage, network)
└── ESXi (bare-metal hypervisor — installed directly on physical server)
    └── VMs (guest operating systems — Windows/Linux)
vCenter Server (management layer — manages multiple ESXi hosts)
└── Cluster (group of ESXi hosts that share resources and VMs)
    ├── Datastore (shared storage — SAN LUNs where VM files live)
    ├── vMotion Network (allows live migration between hosts)
    └── Management Network
```

**Key concepts:**
| Term | What It Is |
|---|---|
| ESXi | VMware's bare-metal hypervisor — installed on physical servers |
| vCenter | Centralised management for all ESXi hosts and VMs |
| Cluster | Group of ESXi hosts sharing resources, enabling HA and DRS |
| Datastore | Storage volume (SAN LUN or NFS share) where VM files (.vmdk) live |
| vMotion | Live migration of a running VM between ESXi hosts — zero downtime |
| DRS | Distributed Resource Scheduler — automatically balances VMs across hosts |
| HA | High Availability — automatically restarts VMs on another host if a host fails |
| FT | Fault Tolerance — zero downtime failover, VM runs simultaneously on two hosts |

---

## Prerequisites
For the hands-on part, you have two options:
1. **Free trial**: VMware vSphere 8 Evaluation from vmware.com (full 60-day lab)
2. **Nested lab**: deploy VMware Cloud Foundation on Azure (expensive, not required for this lab)

The GUI steps in this project describe the **vSphere Client** (browser-based, connects to vCenter). The PowerCLI steps work against any vCenter.

---

## Step 1 — vSphere Client Navigation

### vSphere Client (GUI)
1. Open a browser → go to `https://<vcenter-ip>` → log in as administrator
2. The **Home** page shows: Hosts and Clusters, VMs and Templates, Storage, Networking
3. Click **Hosts and Clusters** in the left panel:
   - You see your **Datacenter** at the top
   - Below it: one or more **Clusters** (e.g. `NHS-Cluster-01`)
   - Below clusters: **ESXi hosts** (e.g. `esxi01.lab.local`)
   - Below hosts: VMs running on that host

**Explore:**
- Click an ESXi host → **Summary** tab: see CPU/memory usage, uptime, VM count
- Click an ESXi host → **Monitor** tab → **Performance**: CPU, memory, disk, network charts
- Click a VM → **Summary** tab: see power state, IP address, tools status, resource usage

---

## Step 2 — Create a Virtual Machine

### vSphere Client (GUI)
1. Right-click a cluster or ESXi host → **New Virtual Machine**
2. **Creation type**: Create a new virtual machine → **Next**
3. Name: `vm-nhs-test` | Select the datacenter → **Next**
4. Select the **compute resource** (cluster or host) → **Next**
5. Select the **datastore** → **Next**
6. Compatibility: ESXi 7.0 or later → **Next**
7. Guest OS: **Microsoft Windows** | Version: **Windows Server 2022** → **Next**
8. Customise hardware:
   - CPU: 2
   - Memory: 4096 MB
   - Hard disk 1: 60 GB (thin provisioned)
   - Network adapter: VM Network
9. Click **Finish**
10. The VM appears in the inventory — right-click → **Power On**

### PowerCLI
```powershell
# Install PowerCLI
Install-Module -Name VMware.PowerCLI -Force -AllowClobber
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false

# Connect to vCenter
Connect-VIServer -Server vcenter.lab.local -User administrator@vsphere.local -Password 'YourPassword'

# Create a VM
New-VM `
  -Name 'vm-nhs-test' `
  -ResourcePool (Get-ResourcePool 'NHS-Cluster-01') `
  -Datastore (Get-Datastore 'NHS-Datastore-01') `
  -DiskGB 60 `
  -MemoryGB 4 `
  -NumCpu 2 `
  -GuestId 'windows2019srvNext64Guest' `
  -NetworkName 'VM Network' `
  -DiskStorageFormat Thin

# Power it on
Start-VM -VM 'vm-nhs-test'

# Check power state
Get-VM 'vm-nhs-test' | Select-Object Name, PowerState
```

---

## Step 3 — Common VM Management Tasks

### vSphere Client (GUI)
| Task | How |
|---|---|
| Power on | Right-click VM → Power → Power On |
| Power off (graceful) | Right-click VM → Power → Shut Down Guest OS |
| Power off (force) | Right-click VM → Power → Power Off |
| Restart | Right-click VM → Power → Restart Guest OS |
| Take snapshot | Right-click VM → Snapshots → Take Snapshot |
| Delete snapshot | Right-click VM → Snapshots → Manage Snapshots |
| Edit settings | Right-click VM → Edit Settings |
| Open console | Right-click VM → Open Remote Console |
| Clone VM | Right-click VM → Clone → Clone to Virtual Machine |

### PowerCLI
```powershell
# List all VMs with power state and resource usage
Get-VM | Select-Object Name, PowerState, NumCpu, MemoryGB | Format-Table

# Stop a VM gracefully (sends shutdown signal to OS)
Stop-VMGuest -VM 'vm-nhs-test' -Confirm:$false

# Force power off (like pulling the plug)
Stop-VM -VM 'vm-nhs-test' -Confirm:$false

# Restart guest OS
Restart-VMGuest -VM 'vm-nhs-test' -Confirm:$false

# Get a VM's IP address (requires VMware Tools installed)
(Get-VM 'vm-nhs-test').Guest.IPAddress

# Find VMs with no snapshots
Get-VM | Where-Object {(Get-Snapshot -VM $_).Count -eq 0} | Select-Object Name
```

---

## Step 4 — Snapshots

Snapshots capture the VM's entire state (disk, memory, power state) at a point in time.

> **Warning:** Snapshots are NOT backups. A VM with many snapshots slows down over time. Snapshots should be short-lived — hours, not days.

### vSphere Client (GUI)
1. Right-click a VM → **Snapshots** → **Take Snapshot**
2. Name: `pre-patch-snapshot`
3. Description: `Before applying June 2026 patches`
4. Tick **Snapshot the virtual machine's memory** if you want to restore to exact running state
5. Click **OK**

**To revert (rollback):**
6. Right-click VM → **Snapshots** → **Manage Snapshots**
7. Select `pre-patch-snapshot` → click **Revert To** → **OK** → the VM returns to its pre-patch state

**To delete a snapshot (commit changes permanently):**
8. Right-click VM → **Snapshots** → **Manage Snapshots**
9. Select the snapshot → **Delete** → the snapshot is removed and its changes are merged into the base disk

### PowerCLI
```powershell
# Take a snapshot
New-Snapshot `
  -VM 'vm-nhs-test' `
  -Name 'pre-patch-snapshot' `
  -Description 'Before June 2026 patching' `
  -Memory `
  -Quiesce

# List snapshots on a VM
Get-Snapshot -VM 'vm-nhs-test' | Select-Object Name, Description, Created, SizeGB

# Revert to a snapshot
Set-VM -VM 'vm-nhs-test' -Snapshot (Get-Snapshot -VM 'vm-nhs-test' -Name 'pre-patch-snapshot') -Confirm:$false

# Delete a specific snapshot (merge changes permanently)
Remove-Snapshot -Snapshot (Get-Snapshot -VM 'vm-nhs-test' -Name 'pre-patch-snapshot') -Confirm:$false

# Find all VMs with snapshots older than 7 days (DANGEROUS — these should be cleaned up)
Get-VM | ForEach-Object {
  Get-Snapshot -VM $_ | Where-Object {$_.Created -lt (Get-Date).AddDays(-7)}
} | Select-Object VM, Name, Created, SizeGB | Format-Table
```

**What you just learned:** Old snapshots are one of the most common causes of poor VM performance and datastore full alerts in NHS VMware environments. Always take a snapshot before patching, set a delete reminder for 48 hours later, and delete it once you confirm the patch was successful.

---

## Step 5 — vMotion (Live Migration)

vMotion migrates a running VM from one ESXi host to another — zero downtime for the VM or its users.

### vSphere Client (GUI)
1. Right-click a running VM → **Migrate**
2. Migration type: **Change compute resource only** (to move to a different host)
3. Select the destination host
4. Select a network
5. Select migration priority: **Schedule vMotion with high priority**
6. Click **Finish**
7. Watch the progress in **Recent Tasks** (bottom of the screen) — the migration takes 30–120 seconds depending on VM memory size

### PowerCLI
```powershell
# Move a VM to a different host
$sourceVM = Get-VM -Name 'vm-nhs-test'
$targetHost = Get-VMHost -Name 'esxi02.lab.local'

Move-VM -VM $sourceVM -Destination $targetHost

# Verify it moved
Get-VM 'vm-nhs-test' | Select-Object Name, VMHost
```

**What you just learned:** vMotion is how you empty an ESXi host before maintenance (firmware upgrades, memory replacement). You vMotion all VMs off the host, put the host in maintenance mode, do the work, then bring it back. Clinical systems never go down.

---

## Step 6 — HA, DRS, and FT Explained

### vSphere Client (GUI)
**HA (High Availability) — configure at cluster level:**
1. Click the cluster → **Configure** → **Services** → **vSphere HA** → **Edit**
2. Enable vSphere HA
3. Failures and Responses: Host Failures Cluster Tolerates: `1`
4. VM restart priority: Critical VMs → **Highest**
5. Click **OK**

**DRS (Distributed Resource Scheduler) — configure at cluster level:**
1. Click the cluster → **Configure** → **Services** → **vSphere DRS** → **Edit**
2. Enable DRS
3. Automation Level: **Fully Automated** (DRS migrates VMs automatically to balance load)
4. Migration Threshold: slide to balance speed vs stability
5. Click **OK**

**What they do:**

| Feature | What Happens | Example |
|---|---|---|
| **HA** | Host fails → all its VMs restart on other hosts (30–120 sec downtime) | ESXi host suffers hardware failure at 3am — VMs restart automatically |
| **DRS** | Cluster gets unbalanced → VMs migrate automatically with vMotion | One host hits 90% CPU — DRS moves some VMs to a quieter host |
| **vMotion** | You manually (or DRS) migrates a VM — zero downtime | Moving VMs before host maintenance |
| **FT** | VM runs simultaneously on two hosts — failover is instant, zero downtime | Patient monitoring system — cannot tolerate even 1 second of downtime |

**FT vs HA:** FT is zero-downtime failover (VM keeps running on the shadow host). HA involves restarting VMs (30–120 seconds of downtime). FT uses more resources (shadow VM consumes CPU/memory on the secondary host). NHS typically uses HA for most VMs and FT only for the most critical systems.

---

## Step 7 — Resource Pools and Reservations

Resource pools control how CPU and memory are shared between VMs.

### vSphere Client (GUI)
1. Right-click a cluster or host → **New Resource Pool**
2. Name: `Clinical-Systems`
3. CPU: Reservation = `4000 MHz` | Shares = `High`
4. Memory: Reservation = `8192 MB` | Shares = `High`
5. Click **OK**
6. Drag critical VMs into the `Clinical-Systems` resource pool

### PowerCLI
```powershell
# Create a resource pool
$cluster = Get-Cluster -Name 'NHS-Cluster-01'
New-ResourcePool `
  -Location $cluster `
  -Name 'Clinical-Systems' `
  -CpuSharesLevel High `
  -MemSharesLevel High `
  -CpuReservationMHz 4000 `
  -MemReservationGB 8

# Move a VM into the resource pool
$pool = Get-ResourcePool -Name 'Clinical-Systems'
Move-VM -VM 'vm-nhs-test' -Destination $pool
```

---

## Step 8 — Useful PowerCLI Operational Scripts

```powershell
# Health check — all hosts and their resource usage
Get-VMHost | Select-Object Name, `
  @{N='CPU%'; E={[Math]::Round($_.CpuUsageMhz/$_.CpuTotalMhz*100,1)}}, `
  @{N='Mem%'; E={[Math]::Round($_.MemoryUsageGB/$_.MemoryTotalGB*100,1)}}, `
  @{N='VMs';  E={(Get-VM -Location $_).Count}} | Format-Table

# Find VMs with snapshots older than 3 days
Get-VM | ForEach-Object {
  Get-Snapshot -VM $_ | Where-Object {$_.Created -lt (Get-Date).AddDays(-3)}
} | Select-Object @{N='VM';E={$_.VM}}, Name, Created | Format-Table

# Find VMs with VMware Tools not up to date
Get-VM | Where-Object {$_.Guest.ToolsStatus -ne 'toolsOK'} | 
  Select-Object Name, @{N='ToolsStatus'; E={$_.Guest.ToolsStatus}}

# Datastore free space report
Get-Datastore | Select-Object Name, `
  @{N='TotalGB'; E={[Math]::Round($_.CapacityGB,0)}}, `
  @{N='FreeGB';  E={[Math]::Round($_.FreeSpaceGB,0)}}, `
  @{N='Used%';   E={[Math]::Round((1-$_.FreeSpaceGB/$_.CapacityGB)*100,1)}} | 
  Sort-Object 'Used%' -Descending | Format-Table
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| ESXi | Bare-metal hypervisor — installed on physical server, runs VMs |
| vCenter | Management platform for all ESXi hosts |
| vMotion | Live migration of a running VM — zero downtime |
| HA | Restarts VMs on surviving hosts when a host fails (30–120s downtime) |
| DRS | Auto-balances VM load across hosts using vMotion |
| FT | Zero-downtime failover — VM runs on two hosts simultaneously |
| Snapshot | Point-in-time state capture — NOT a backup |
| Thin provisioning | Disk only consumes space as data is written — saves datastore space |
| Datastore | Storage volume where VM disk files (.vmdk) live |

---

## Common Interview Questions

1. **"What is the difference between VMware HA and FT?"** — HA (High Availability) restarts VMs on another host if their host fails. There is 30–120 seconds of downtime while the VM restarts. FT (Fault Tolerance) runs the VM simultaneously on a primary and shadow host in lockstep. If the primary fails, the shadow takes over instantly with zero downtime. HA is used for most VMs; FT is reserved for the most critical systems because it doubles resource usage.

2. **"A clinician says their application was unavailable for 2 minutes last night. How do you investigate in VMware?"** — Check vCenter Events for the VM — look for vMotion, HA restart, or power state changes. Check the ESXi host it was on for host failures. Check the datastore for space issues (full datastore causes VMs to pause). Check the VM's event log inside the guest OS for application errors at that time.

3. **"What are snapshots and why should they not be left running long-term?"** — Snapshots capture the VM state at a point in time — useful before patching or risky changes. When a snapshot is active, all disk writes go to a new delta file instead of the base disk. The longer a snapshot runs, the larger this delta file grows. The VM performance degrades as vSphere has to merge multiple delta files for each read/write. Old snapshots can also fill the datastore, causing all VMs on it to pause.

---

## Next Steps
Move on to **Project 18 — ITIL, Change Management & NHS Operations**.
