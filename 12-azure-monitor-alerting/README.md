# Project 12 — Azure Monitor & Alerting

## What You Will Learn
- How Azure Monitor collects metrics and logs
- How to set up a Log Analytics workspace
- How to query logs with KQL (Kusto Query Language)
- How to create metric alerts and log alerts
- How to configure action groups (email/SMS notifications)
- How to build a monitoring dashboard in the Portal

---

## Background — Read This First

**Azure Monitor** is the umbrella service for all Azure observability. It feeds data to:
- **Metrics** — numeric time-series data (CPU%, disk IOPS, requests/sec). Stored for 93 days.
- **Logs** — structured event data stored in **Log Analytics** workspace. Stored for 30 days (default), up to 2 years. Queried with KQL.
- **Alerts** — rules that fire when a metric or log query crosses a threshold.

In the NHS, monitoring is non-negotiable for DSPT compliance. Systems affecting patient care must have alerting in place so incidents are detected automatically — not when a user calls the helpdesk.

---

## Prerequisites
- Azure subscription, CLI logged in
- A VM deployed — use `rg-monitor-lab` with a Linux VM (see Step 1)

---

## Step 1 — Deploy a VM to Monitor

### Portal (GUI)
Create `vm-monitor` in `rg-monitor-lab` (Ubuntu 22.04, Standard_B1s). See Project 02 for full steps.

### CLI
```bash
az group create --name rg-monitor-lab --location uksouth

az vm create \
  --resource-group rg-monitor-lab \
  --name vm-monitor \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

---

## Step 2 — Create a Log Analytics Workspace

### Portal (GUI)
1. Search **Log Analytics workspaces** → **+ Create**
2. Resource group: `rg-monitor-lab`
3. Name: `law-monitor-lab`
4. Region: `UK South`
5. Click **Review + create** → **Create**

### CLI
```bash
az monitor log-analytics workspace create \
  --resource-group rg-monitor-lab \
  --workspace-name law-monitor-lab \
  --location uksouth \
  --sku PerGB2018

WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-monitor-lab \
  --workspace-name law-monitor-lab \
  --query id --output tsv)
echo "Workspace ID: $WORKSPACE_ID"
```

---

## Step 3 — Connect the VM to Log Analytics

This installs the Azure Monitor Agent on the VM so it starts sending logs to your workspace.

### Portal (GUI)
1. Navigate to `vm-monitor` → **Insights** (under Monitoring in the left panel)
2. Click **Enable**
3. Select your Log Analytics workspace `law-monitor-lab` → **Enable**
4. Wait 3–5 minutes for the agent to deploy and start collecting data

### CLI
```bash
VM_RESOURCE_ID=$(az vm show \
  --resource-group rg-monitor-lab \
  --name vm-monitor \
  --query id --output tsv)

az monitor diagnostic-settings create \
  --name vm-monitor-diag \
  --resource $VM_RESOURCE_ID \
  --workspace $WORKSPACE_ID \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

**What you just learned:** The diagnostic settings push VM metrics and platform logs to Log Analytics. The Azure Monitor Agent (AMA) additionally collects OS-level data (syslog, performance counters) from inside the VM.

---

## Step 4 — View Platform Metrics

### Portal (GUI)
1. Navigate to `vm-monitor`
2. Click **Metrics** in the left panel (under Monitoring)
3. In the chart area, set:
   - **Metric Namespace**: `Virtual Machine Host`
   - **Metric**: `Percentage CPU`
   - **Aggregation**: `Average`
4. Change the time range to **Last 1 hour** (top right)
5. Click **+ Add metric** to overlay another metric: `Network In Total`
6. Click **Pin to dashboard** → **Create new** → name it `NHS-Infra-Dashboard`

### CLI
```bash
# Get CPU average over last hour
az monitor metrics list \
  --resource $VM_RESOURCE_ID \
  --metric "Percentage CPU" \
  --interval PT5M \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --output table
```

---

## Step 5 — Create a Metric Alert (CPU > 80%)

### Portal (GUI)
1. Navigate to `vm-monitor` → **Alerts** in the left panel (under Monitoring)
2. Click **+ Create** → **Alert rule**
3. **Condition** tab:
   - Signal: **Percentage CPU**
   - Threshold type: **Static**
   - Operator: **Greater than**
   - Threshold value: `80`
   - Aggregation granularity: `5 minutes`
   - Frequency: `1 minute`
4. **Actions** tab: click **+ Create action group** (see Step 6)
5. **Details** tab:
   - Alert rule name: `alert-vm-cpu-high`
   - Severity: `Sev 2 - Warning`
6. Click **Review + create** → **Create**

### CLI
```bash
az monitor metrics alert create \
  --name alert-vm-cpu-high \
  --resource-group rg-monitor-lab \
  --scopes $VM_RESOURCE_ID \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "VM CPU is above 80% for 5 minutes"
```

**What you just learned:** The alert fires when CPU *averages* over 80% for 5 consecutive minutes — not just a momentary spike. This prevents false alarms from normal short bursts. Severity 1 = Critical, 2 = Warning, 3 = Informational.

---

## Step 6 — Create an Action Group (Notifications)

An action group tells Azure *who* to notify when an alert fires.

### Portal (GUI)
1. Search **Monitor** → click **Alerts** in the left panel → **Action groups** → **+ Create**
2. Resource group: `rg-monitor-lab`
3. Action group name: `ag-nhs-infra`
4. Display name: `NHS-Infra`
5. **Notifications** tab:
   - Type: **Email/SMS/Push/Voice**
   - Name: `EmailOnCall`
   - Email: your email address
   - Tick **Email** → click **OK**
6. Click **Review + create** → **Create**
7. Go back to the CPU alert → edit it → **Actions** → **Select action groups** → pick `ag-nhs-infra`

### CLI
```bash
az monitor action-group create \
  --name ag-nhs-infra \
  --resource-group rg-monitor-lab \
  --short-name NHSInfra \
  --action email EmailOnCall olamidekosile@gmail.com
```

---

## Step 7 — KQL: Query Logs in Log Analytics

KQL is the query language for Log Analytics. Every NHS Azure monitoring role uses it.

### Portal (GUI)
1. Navigate to `law-monitor-lab` Log Analytics workspace
2. Click **Logs** in the left panel
3. Close the query examples popup
4. Paste each query below into the editor and click **Run**

```kusto
// Last heartbeats from your VM
Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| order by LastHeartbeat desc
```

```kusto
// Average CPU per 5 minutes
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where TimeGenerated > ago(1h)
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart
```

```kusto
// Available memory trend
Perf
| where ObjectName == "Memory" and CounterName == "Available MBytes"
| where TimeGenerated > ago(1h)
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart
```

```kusto
// Count events by severity in last 24 hours
Event
| where TimeGenerated > ago(24h)
| summarize count() by EventLevelName
| order by count_ desc
```

```kusto
// Failed SSH login attempts (Linux syslog)
Syslog
| where TimeGenerated > ago(24h)
| where Facility == "auth"
| where SyslogMessage contains "Failed"
| summarize count() by Computer
| order by count_ desc
```

**What you just learned:** KQL follows a pipeline model: table → filters → transforms → output. Key operators: `where` (filter rows), `project` (choose columns), `summarize` (aggregate), `render` (chart), `ago()` (relative time). These five operators cover 90% of real-world monitoring queries.

---

## Step 8 — Create a Log Alert (SSH Brute Force)

Log alerts fire based on a KQL query result.

### Portal (GUI)
1. In the Log Analytics workspace → **Alerts** → **+ New alert rule**
2. **Condition** tab → **Custom log search**
3. Paste the query:
   ```kusto
   Syslog
   | where TimeGenerated > ago(5m)
   | where Facility == "auth" and SyslogMessage contains "Failed password"
   | summarize count() by Computer
   | where count_ > 10
   ```
4. Alert logic: result count **greater than** `0`
5. Evaluation period: `5 minutes`
6. **Actions**: select action group `ag-nhs-infra`
7. Alert name: `alert-ssh-brute-force` | Severity: `Sev 1 - Critical`
8. **Review + create** → **Create**

### CLI
```bash
LAW_RESOURCE_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-monitor-lab \
  --workspace-name law-monitor-lab \
  --query id --output tsv)

az monitor scheduled-query create \
  --name alert-ssh-brute-force \
  --resource-group rg-monitor-lab \
  --scopes $LAW_RESOURCE_ID \
  --condition-query "Syslog | where TimeGenerated > ago(5m) | where Facility == 'auth' and SyslogMessage contains 'Failed password' | summarize count() by Computer | where count_ > 10" \
  --condition-threshold 0 \
  --condition-operator GreaterThan \
  --evaluation-frequency "PT5M" \
  --window-size "PT5M" \
  --severity 1 \
  --description "More than 10 failed SSH logins in 5 minutes"
```

---

## Step 9 — Build a Dashboard

### Portal (GUI)
1. Search **Dashboard** in the top bar → **+ New dashboard** → **Blank dashboard**
2. Name: `NHS-Infrastructure-Dashboard`
3. From the gallery, drag in:
   - **Metrics chart** → configure for CPU of `vm-monitor`
   - **Log Analytics** → paste the heartbeat query
   - **Markdown** → add a header tile: `## Infrastructure Status`
4. Click **Save**

Dashboard tiles auto-refresh every 5 minutes by default.

---

## Step 10 — Clean Up

### Portal (GUI)
Navigate to `rg-monitor-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-monitor-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Azure Monitor | Umbrella service collecting metrics and logs from all Azure resources |
| Log Analytics workspace | Repository for log data — query with KQL |
| Metrics | Numeric time-series data — 93-day retention |
| KQL | Kusto Query Language — used to query Log Analytics logs |
| Action group | Defines who gets notified when an alert fires |
| Metric alert | Fires when a metric crosses a threshold for a sustained period |
| Log alert | Fires when a KQL query returns results matching a condition |
| `ago()` | KQL relative time function — `ago(1h)` = one hour before now |

---

## Common Interview Questions

1. **"What is the difference between metrics and logs in Azure Monitor?"** — Metrics are numeric time-series data (CPU%, requests/sec) stored for 93 days — great for dashboards and threshold alerts. Logs are structured event data in Log Analytics, queryable with KQL — great for detailed investigation and complex conditions like "more than 10 failed logins in 5 minutes."

2. **"Write a KQL query to find the top 5 processes by CPU usage."** — `Perf | where ObjectName == "Process" and CounterName == "% Processor Time" | where TimeGenerated > ago(1h) | summarize avg(CounterValue) by InstanceName | top 5 by avg_CounterValue desc`

3. **"An alert fired at 03:00 saying CPU hit 95%. How do you investigate?"** — Check the metrics chart to see the duration and pattern. Run KQL to check which processes caused it. Check if it was a scheduled job (backup, patching). Check application logs for errors that might explain the spike.

---

## Next Steps
Move on to **Project 13 — Azure DevOps CI/CD**.
