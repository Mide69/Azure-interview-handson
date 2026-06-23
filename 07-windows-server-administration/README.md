# Project 07 — Windows Server Administration

## What You Will Learn
- How to connect to and manage a Windows Server via RDP and PowerShell
- How to use Azure Portal tools (Run Command, Serial Console) for server management
- How to install server roles (IIS, DNS, DHCP)
- How to manage services, event logs, users, disk, and firewall from the console
- How to write PowerShell health-check scripts for daily ops

---

## Background — Read This First

Windows Server is the most common server OS in NHS environments. As a Junior Infrastructure Engineer, you will administer Windows Servers constantly. The golden rule: **do everything from PowerShell** — not the GUI. Scripted operations are repeatable, documentable, and reviewable in a change management system.

**Two ways to run PowerShell on a remote server:**
- **RDP then open PowerShell** — interactive, good for exploration
- **PowerShell Remoting (WinRM)** — like SSH for Windows, runs commands remotely without full RDP
- **Azure Run Command** — runs through the VM Agent, no network needed — emergency access

---

## Prerequisites
- Azure subscription, CLI logged in
- Completed Projects 01 and 02

---

## Step 1 — Deploy a Windows Server VM

### Portal (GUI)
1. Search **Virtual machines** → **+ Create**
2. Image: `Windows Server 2022 Datacenter` | Size: `Standard_B2s`
3. Username: `winadmin` | Password: `WinL@b2026!!`
4. Networking: allow RDP (3389)
5. Create in resource group `rg-winserver-lab`

### CLI
```bash
az group create --name rg-winserver-lab --location uksouth

az vm create \
  --resource-group rg-winserver-lab \
  --name vm-winserver \
  --image Win2022Datacenter \
  --size Standard_B2s \
  --admin-username winadmin \
  --admin-password "WinL@b2026!!" \
  --public-ip-sku Standard

az vm open-port \
  --resource-group rg-winserver-lab \
  --name vm-winserver \
  --port 3389 --priority 1001

WIN_IP=$(az vm show \
  --resource-group rg-winserver-lab \
  --name vm-winserver \
  --show-details --query publicIps --output tsv)
echo "Windows Server IP: $WIN_IP"
```

---

## Step 2 — Connect via RDP and Open PowerShell

### Portal (GUI)
1. Navigate to `vm-winserver` → click **Connect** → **RDP**
2. Click **Download RDP File** → open it → log in as `winadmin`
3. Once on the desktop, right-click the Start button → **Windows PowerShell (Admin)**
4. You now have an elevated PowerShell session — this is where you will run all commands

### CLI (from your local machine)
```bash
# Connect from Windows
mstsc /v:$WIN_IP
# Log in as winadmin / WinL@b2026!!
# Then open PowerShell as Administrator on the server
```

All subsequent commands in Steps 3–11 are run **inside PowerShell on the server** (either via RDP or via Azure Run Command from the Portal/CLI).

---

## Step 3 — System Information and Health Check

### Portal (GUI)
1. On the server, open PowerShell (Admin)
2. Paste and run each block below

### Azure Run Command (no RDP needed)
In the Portal: VM → **Run command** → **RunPowerShellScript** → paste the script → **Run**

Or via CLI:
```bash
az vm run-command invoke \
  --resource-group rg-winserver-lab \
  --name vm-winserver \
  --command-id RunPowerShellScript \
  --scripts "
    Write-Host '=== OS Version ==='
    (Get-WmiObject Win32_OperatingSystem).Caption
    (Get-WmiObject Win32_OperatingSystem).Version

    Write-Host '=== CPU Load ==='
    (Get-WmiObject Win32_Processor).LoadPercentage

    Write-Host '=== Memory ==='
    \$os = Get-WmiObject Win32_OperatingSystem
    \$usedGB = [Math]::Round((\$os.TotalVisibleMemorySize - \$os.FreePhysicalMemory)/1MB, 2)
    \$totalGB = [Math]::Round(\$os.TotalVisibleMemorySize/1MB, 2)
    Write-Host ('{0} GB used / {1} GB total' -f \$usedGB, \$totalGB)

    Write-Host '=== Disk Space ==='
    Get-Volume | Where-Object {\$_.DriveLetter} | ForEach-Object {
      \$pct = [Math]::Round((\$_.Size - \$_.SizeRemaining)/\$_.Size*100, 1)
      Write-Host ('Drive {0}: {1}% used' -f \$_.DriveLetter, \$pct)
    }

    Write-Host '=== Uptime ==='
    (Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
  "
```

**What you just learned:** This health-check script is the first thing you run when you log onto a server. In a real ops team, you might run this automatically each morning via a scheduled task and email the output.

---

## Step 4 — Manage Windows Services

Run inside PowerShell on the server (or via Run Command):

```powershell
# List all running services
Get-Service | Where-Object {$_.Status -eq 'Running'} | Select-Object Name, DisplayName, StartType | Format-Table

# Check a specific service
Get-Service -Name wuauserv | Select-Object Name, Status, StartType

# Stop a service
Stop-Service -Name Spooler -Force
Write-Host "Spooler status:" (Get-Service Spooler).Status

# Start a service
Start-Service -Name Spooler
Write-Host "Spooler status:" (Get-Service Spooler).Status

# Restart a service
Restart-Service -Name Spooler

# Set startup type
Set-Service -Name Spooler -StartupType Automatic

# Find services that should be running but aren't
Get-Service | Where-Object {$_.StartType -eq 'Automatic' -and $_.Status -ne 'Running'}
```

**What you just learned:** Service management is one of the most common daily tasks. `Get-Service`, `Start-Service`, `Stop-Service`, `Restart-Service` are the core commands. The last query — finding stopped auto-start services — is your first diagnostic check when users report something is not working.

---

## Step 5 — Install a Server Role (IIS Web Server)

```powershell
# Inside PowerShell on the server

# Install IIS with management tools
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

# Verify installation
Get-WindowsFeature -Name Web-Server

# Start the web service
Start-Service W3SVC

# Test IIS is serving the default page
(Invoke-WebRequest -Uri 'http://localhost' -UseBasicParsing).StatusCode
# Should return 200
```

### Portal (GUI)
To access IIS Manager remotely after installation:
1. Back in the Portal, open port 80 on the VM's NSG (Networking tab → Add inbound port rule → port 80)
2. In your browser, navigate to `http://<VM-IP>` — you see the IIS default page

```bash
# Open port 80 via CLI
az vm open-port \
  --resource-group rg-winserver-lab \
  --name vm-winserver \
  --port 80 --priority 1002

# Test from your machine
curl http://$WIN_IP
```

**What you just learned:** `Install-WindowsFeature` installs roles and features on Server Core (no GUI) and is the scripted way to configure servers. Key roles: `Web-Server` (IIS), `DNS`, `DHCP`, `AD-Domain-Services`, `File-Services`, `Hyper-V`.

---

## Step 6 — Manage Local Users and Groups

```powershell
# Create a local user
$Password = ConvertTo-SecureString 'UserP@ss2026!' -AsPlainText -Force
New-LocalUser -Name 'jsmith' -Password $Password -FullName 'John Smith' -Description 'Lab test user'

# Add user to local Administrators group
Add-LocalGroupMember -Group 'Administrators' -Member 'jsmith'

# List all local users
Get-LocalUser | Select-Object Name, Enabled, LastLogon | Format-Table

# List members of Administrators group
Get-LocalGroupMember -Group 'Administrators' | Select-Object Name, ObjectClass

# Disable a user (e.g. when someone leaves)
Disable-LocalUser -Name 'jsmith'
Write-Host "jsmith enabled:" (Get-LocalUser -Name 'jsmith').Enabled

# Re-enable
Enable-LocalUser -Name 'jsmith'

# Remove a user
Remove-LocalUser -Name 'jsmith'
```

---

## Step 7 — Check Disk Volumes

```powershell
# List all disks
Get-Disk | Select-Object Number, FriendlyName, Size, PartitionStyle, OperationalStatus

# List all volumes
Get-Volume | Select-Object DriveLetter, FileSystemLabel, FileSystem, SizeRemaining, Size | Format-Table

# Check disk utilisation
Get-PSDrive -PSProvider FileSystem | Select-Object Name, @{N='UsedGB';E={[Math]::Round($_.Used/1GB,2)}}, @{N='FreeGB';E={[Math]::Round($_.Free/1GB,2)}} | Format-Table

# Alert if any drive is over 80% used
Get-Volume | Where-Object {$_.DriveLetter} | ForEach-Object {
  $pct = ($_.Size - $_.SizeRemaining) / $_.Size * 100
  if ($pct -gt 80) {
    Write-Warning ("Drive {0} is {1:N1}% full!" -f $_.DriveLetter, $pct)
  }
}
```

**What you just learned:** Disk space monitoring is a daily task. 80% = warning, 90% = critical. Running out of disk space causes services to crash and databases to corrupt. In Azure, you can expand a managed disk while the VM is running.

---

## Step 8 — Event Logs (Your Audit Trail)

```powershell
# Last 20 System errors
Get-EventLog -LogName System -EntryType Error -Newest 20 | Select-Object TimeGenerated, Source, Message | Format-Table -Wrap

# Last 20 Application errors
Get-EventLog -LogName Application -EntryType Error -Newest 20 | Select-Object TimeGenerated, Source, Message | Format-Table -Wrap

# Security events: failed logons (Event ID 4625) — brute force indicator
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 10 -ErrorAction SilentlyContinue | 
  Select-Object TimeCreated, @{N='Message';E={$_.Message.Substring(0,200)}}

# Service crash events (ID 7034)
Get-WinEvent -FilterHashtable @{LogName='System'; Id=7034} -MaxEvents 5 -ErrorAction SilentlyContinue | 
  Select-Object TimeCreated, Message

# Events from the last hour
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=(Get-Date).AddHours(-1)} -ErrorAction SilentlyContinue | 
  Select-Object TimeCreated, LevelDisplayName, Message | Format-Table
```

**Important Event IDs to memorise:**
| Event ID | Log | Meaning |
|---|---|---|
| 4624 | Security | Successful logon |
| 4625 | Security | Failed logon — brute force indicator |
| 4648 | Security | Logon with explicit credentials |
| 7034 | System | Service crashed |
| 7036 | System | Service started or stopped |
| 41 | System | Kernel power — unexpected restart |

---

## Step 9 — Windows Firewall Management

```powershell
# Check Windows Firewall status
Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction

# List enabled inbound rules
Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True' -and $_.Direction -eq 'Inbound'} | 
  Select-Object DisplayName, Action, Protocol | Select-Object -First 15

# Add an inbound rule to allow port 8080
New-NetFirewallRule -DisplayName 'Allow Port 8080' -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow

# Verify the rule
Get-NetFirewallRule -DisplayName 'Allow Port 8080' | Select-Object DisplayName, Enabled, Action

# Remove the rule
Remove-NetFirewallRule -DisplayName 'Allow Port 8080'
```

**What you just learned:** Windows Firewall and Azure NSG are two separate, independent firewall layers. A port allowed in the NSG but blocked by Windows Firewall = no connectivity. Always check both when troubleshooting. Use `Get-NetFirewallRule` to check the Windows layer.

---

## Step 10 — Check Running Processes

```powershell
# Top 10 processes by CPU
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, CPU, @{N='MemoryMB';E={[Math]::Round($_.WorkingSet/1MB,1)}}, Id | Format-Table

# Top 10 processes by memory
Get-Process | Sort-Object WorkingSet -Descending | Select-Object -First 10 Name, @{N='MemoryMB';E={[Math]::Round($_.WorkingSet/1MB,1)}}, CPU, Id | Format-Table

# Find a specific process
Get-Process | Where-Object {$_.Name -like '*iis*'}

# Kill a process by ID (replace with actual PID)
# Stop-Process -Id 1234 -Force
```

---

## Step 11 — Schedule a Task

```powershell
# Create a daily cleanup scheduled task
$Action = New-ScheduledTaskAction -Execute 'PowerShell.exe' -Argument '-NoProfile -Command "Get-ChildItem C:\Logs\*.log | Where-Object {$_.LastWriteTime -lt (Get-Date).AddDays(-30)} | Remove-Item"'
$Trigger = New-ScheduledTaskTrigger -Daily -At '03:00'
$Settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit (New-TimeSpan -Hours 1)
Register-ScheduledTask -TaskName 'CleanOldLogs' -Action $Action -Trigger $Trigger -Settings $Settings -RunLevel Highest -Force

# List scheduled tasks
Get-ScheduledTask | Where-Object {$_.TaskName -eq 'CleanOldLogs'} | Select-Object TaskName, State

# Run a task immediately (for testing)
Start-ScheduledTask -TaskName 'CleanOldLogs'

# Remove the task
Unregister-ScheduledTask -TaskName 'CleanOldLogs' -Confirm:$false
```

---

## Step 12 — Clean Up

### Portal (GUI)
Navigate to `rg-winserver-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-winserver-lab --yes --no-wait
```

---

## Key PowerShell Commands to Memorise

| Task | Command |
|---|---|
| List running services | `Get-Service | Where-Object {$_.Status -eq 'Running'}` |
| Stop/Start/Restart service | `Stop-Service / Start-Service / Restart-Service -Name X` |
| Install role | `Install-WindowsFeature -Name Web-Server -IncludeManagementTools` |
| Read event log errors | `Get-EventLog -LogName System -EntryType Error -Newest 20` |
| Top processes by CPU | `Get-Process | Sort-Object CPU -Descending | Select-Object -First 10` |
| List disk volumes | `Get-Volume` |
| Create local user | `New-LocalUser -Name X -Password $pass` |
| Add firewall rule | `New-NetFirewallRule -DisplayName X -Protocol TCP -LocalPort 80 -Action Allow` |
| Run on remote server | `Invoke-Command -ComputerName SERVER -ScriptBlock { Get-Service }` |

---

## Common Interview Questions

1. **"A Windows service fails to start after reboot. How do you diagnose it?"** — Check Application and System event logs for errors around reboot time using `Get-EventLog`. Check if the service account's password expired. Check service dependencies (`Get-Service -Name X -RequiredServices`). Check disk space. Try `Start-Service -Name X` and read the error message.

2. **"How would you find what's causing high CPU on a Windows Server?"** — `Get-Process | Sort-Object CPU -Descending | Select-Object -First 10`. Identify the PID, correlate with Event Log, and check whether it is a known service behaving abnormally or an unexpected process.

3. **"How do you install a server role without a GUI?"** — `Install-WindowsFeature -Name <role> -IncludeManagementTools`. This works on Server Core installations and remoted PowerShell sessions. For a list of all available features: `Get-WindowsFeature | Where-Object {$_.InstallState -eq 'Available'}`.

---

## Next Steps
Move on to **Project 08 — Linux Server Administration**.
