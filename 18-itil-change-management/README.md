# Project 18 — ITIL, Change Management & NHS Operations

## What You Will Learn
- What ITIL is and why every NHS infrastructure role uses it
- The difference between Incidents, Problems, Changes, and Service Requests
- How the Change Advisory Board (CAB) works
- RPO and RTO — what they mean and how to calculate them
- How to write a change request and incident report
- NHS values and how to demonstrate them in an interview

---

## Background — Read This First

**ITIL (Information Technology Infrastructure Library)** is a set of practices for IT service management. The NHS mandates ITIL because clinical systems are safety-critical — an unmanaged change to a patient administration system could affect patient care.

In the SLaM job spec: *"Good working knowledge of ITIL Service Management processes"* is listed as an essential requirement.

**The four ITIL service management processes you must know:**

| Process | Triggered by | Goal |
|---|---|---|
| **Incident Management** | Something is broken right now | Restore service as fast as possible |
| **Problem Management** | Root cause investigation | Prevent incidents from recurring |
| **Change Management** | Planned modification to infrastructure | Make changes safely with minimal risk |
| **Service Request** | User wants something | Deliver standard requests efficiently |

---

## Process 1 — Incident Management

An incident is any unplanned interruption or degradation of an IT service.

**Priority matrix:**
| Priority | Impact | Urgency | Example | Target Resolution |
|---|---|---|---|---|
| **P1 Critical** | High | High | EPR (patient records) system down | 1 hour |
| **P2 High** | High | Medium or High | Email down for whole trust | 4 hours |
| **P3 Medium** | Medium | Medium | Single department printer offline | 8 hours |
| **P4 Low** | Low | Low | One user's desktop wallpaper won't set | Next business day |

**The Incident lifecycle:**
```
Detection → Logging → Classification → Investigation → Resolution → Closure
```

**What you do as an infrastructure engineer during a P1:**
1. Acknowledge the incident in the ITSM tool (ServiceNow/Cherwell) — this stops the escalation timer
2. Triage: is the service fully down or degraded? How many users affected?
3. Escalate if needed (Level 2 → Level 3 → vendor)
4. Communicate: update the incident every 30 minutes with what you've found and what you're doing
5. Resolve: restore service using the quickest path — reboot, failover, rollback
6. Document: what was the issue, what was the fix, timeline of actions

**Azure Portal equivalent — Service Health:**
1. Search **Service Health** in the Azure Portal
2. Click **Service issues** — see any active Azure platform outages affecting your subscription
3. Click **Health advisories** — see planned maintenance or upcoming changes
4. Click **Health alerts** — set up alerts for Azure service degradation

---

## Process 2 — Problem Management

A problem is the unknown root cause of one or more incidents.

**Example:** Three P2 incidents in one week where the clinical imaging system became slow. Each time, restarting a service fixed it in 15 minutes. Problem management investigates WHY the service keeps crashing — is it a memory leak? A database connection pool exhaustion? A nightly job that doesn't clean up?

**Problem Management steps:**
1. **Identify** the problem (pattern of incidents, user complaints)
2. **Log** it in the ITSM system
3. **Investigate** root cause — check logs, metrics, application traces
4. **Raise a Known Error** with a workaround (so helpdesk can apply the fix without escalating)
5. **Resolve** through a change request (permanent fix — code patch, config change, hardware upgrade)

**KQL for problem investigation (Azure Monitor):**
```kusto
// Find all incidents involving a specific server in the last 30 days
Heartbeat
| where Computer contains "imaging-server"
| where TimeGenerated > ago(30d)
| summarize count() by bin(TimeGenerated, 1d), Computer
| render timechart
```

---

## Process 3 — Change Management

A change is any addition, modification, or removal of anything that could affect IT services.

**Change types:**
| Type | Description | Approval Required | Example |
|---|---|---|---|
| **Standard** | Pre-approved, routine, low risk | Automatic | Adding a user, applying a pre-approved patch |
| **Normal** | Planned, assessed, CAB-approved | CAB | Upgrading a server OS, changing firewall rules |
| **Emergency** | Urgent, skips CAB | ECAB (Emergency CAB) or senior manager | Applying a zero-day patch, emergency config fix |

**The CAB (Change Advisory Board)** meets weekly (usually Tuesday/Thursday in NHS trusts). Stakeholders review all Normal changes planned for the coming week. They assess risk, confirm testing is done, and approve or reject.

**Writing a Change Request — what it must contain:**
```
Title: Upgrade SQL Server 2019 → SQL Server 2022 on PROD-DB-01
CI (Configuration Item): PROD-DB-01
Change Type: Normal
Risk: Medium
Impact: Maintenance window required — PROD-DB-01 offline for 2 hours
Implementation window: Sunday 14 July 2026, 02:00–04:00
Testing done: Tested on TEST-DB-01 (17 June), all apps verified working
Rollback plan: Restore from snapshot taken before upgrade begins
Backout time: 30 minutes
Dependencies: Notify application teams to be available during window
```

**Azure DevOps Board — create a work item for the change:**
1. In Azure DevOps → **Boards** → **Work Items** → **+ New Work Item** → **Change Request**
2. Fill in: Title, Description, Priority, Impact, Risk
3. Assign to yourself
4. Link to related incidents or problems

---

## Process 4 — Service Requests

A service request is a formal request for something new — not a break/fix.

Examples:
- New starter needs a laptop and M365 account
- Department needs a new shared mailbox
- Developer needs a VM in the test environment
- User needs read access to a shared drive

Service requests follow a standard fulfilment process. In Azure:

```bash
# Common service requests you will fulfil as an infrastructure engineer

# "New starter needs a VM"
az vm create --resource-group rg-production --name vm-newstarter ...

# "Grant read access to storage"
az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee "user@nhs.net" \
  --scope /subscriptions/.../resourceGroups/rg-data/...

# "Increase VM size for application team"
az vm deallocate --resource-group rg-prod --name vm-app01
az vm resize --resource-group rg-prod --name vm-app01 --size Standard_D4s_v3
az vm start --resource-group rg-prod --name vm-app01
```

---

## RPO and RTO — Know These Definitions

**RTO (Recovery Time Objective):** How long the service can be down before it causes unacceptable harm.
- Example: "The EPR system RTO is 4 hours" — if the EPR is down for more than 4 hours, patient safety is at risk.

**RPO (Recovery Point Objective):** How much data can be lost (in time) without unacceptable harm.
- Example: "The EPR database RPO is 1 hour" — if the database crashes, the worst case is losing 1 hour of data.

**How these drive infrastructure decisions:**
| System | RTO | RPO | Architecture Required |
|---|---|---|---|
| Patient Administration (EPR) | 4 hours | 1 hour | Azure Backup (hourly), HA cluster |
| Email | 8 hours | 24 hours | Exchange Online geo-redundancy |
| Staff intranet | 24 hours | 24 hours | Daily backup sufficient |
| Patient monitoring | 0 minutes | 0 minutes | VMware FT, synchronous replication |

**Practice calculation:**
> "Your backup runs nightly at 02:00 and takes 1 hour. A server fails at 09:00. Your RPO is 7 hours (data since 02:00 backup is lost). Your RTO depends on how fast you can restore."

---

## NHS Values — Use These in Behavioural Interview Answers

The NHS Constitution defines six core values. Interviewers at SLaM will ask STAR-format questions that test these. Map your technical answers to these values.

| Value | How to demonstrate it in an IT context |
|---|---|
| Working together for patients | "I ensured the clinical system was restored first because it directly affected patient care" |
| Respect and dignity | "I communicated updates every 30 minutes so clinical staff could manage their work" |
| Commitment to quality | "I followed the change management process to prevent an outage, even under time pressure" |
| Compassion | "I stayed late to resolve the issue, knowing clinical staff were waiting" |
| Improving lives | "The monitoring system I implemented gave the team early warning and prevented a patient safety incident" |
| Everyone counts | "I made sure the fix worked for the ward with the oldest equipment, not just the modern systems" |

---

## STAR Format for Behavioural Questions

ITIL questions in interviews are typically STAR format. Prepare a story for each.

**Structure:**
- **S**ituation: set the scene (what system, what environment)
- **T**ask: what was your responsibility
- **A**ction: what you did (specific, technical, first-person)
- **R**esult: the outcome (restored in X minutes, prevented Y incidents, saved Z hours)

**Example prepared answer for "Tell me about a time you managed an incident":**

> *Situation:* At [previous role], our clinical imaging system PACS went offline at 14:00 on a Tuesday — 200 radiologists across three hospitals were affected.
>
> *Task:* As the on-call infrastructure engineer, I was the first responder. My job was to triage, communicate, and restore service.
>
> *Action:* I logged a P1 in ServiceNow and acknowledged within 5 minutes (stopping escalation). I checked Azure Monitor and found the VM's disk had hit 100% — the PACS log files had filled it. I attached a new 100GB managed disk, moved the log files, and restarted the PACS service. I sent a stakeholder update every 15 minutes throughout.
>
> *Result:* Service was restored in 47 minutes. After the incident, I raised a Problem record and implemented a Log Analytics alert to fire when disk hits 80%. The same incident hasn't recurred in 8 months.

---

## Azure Tools That Map to ITIL Processes

| ITIL Process | Azure Tool |
|---|---|
| Incident detection | Azure Monitor alerts, Service Health |
| Incident investigation | Log Analytics KQL queries, Metrics charts |
| Problem root cause analysis | Azure Diagnostic logs, Activity Log, Network Watcher |
| Change documentation | Azure Activity Log (records all API calls) |
| Change deployment | Azure DevOps pipelines with approval gates |
| Service request fulfilment | Azure Portal, Azure CLI, PowerShell automation |
| Backup (supports RPO) | Azure Backup, Recovery Services Vault |
| DR (supports RTO) | Azure Site Recovery, geo-redundant deployments |

---

## Activity Log — Your Audit Trail

The Azure Activity Log records every change to every resource — who did what and when. This satisfies ITIL Change Management audit requirements.

### Portal (GUI)
1. Search **Monitor** → **Activity log** in the left panel
2. Filter by: Subscription, Resource group, time range
3. Click any event to see: who initiated it, what changed, the before/after values, result (Succeeded/Failed)

### CLI
```bash
# Last 50 operations in a resource group
az monitor activity-log list \
  --resource-group rg-production \
  --max-events 50 \
  --output table

# Find who deleted a resource
az monitor activity-log list \
  --resource-group rg-production \
  --status Failed \
  --offset 7d \
  --query "[?operationName.value=='Microsoft.Resources/subscriptions/resourceGroups/delete']" \
  --output table
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Incident | Unplanned outage or degradation — restore service ASAP |
| Problem | Root cause investigation to prevent recurrence |
| Change (Normal) | CAB-approved modification — planned and tested |
| Change (Emergency) | Urgent change, skips full CAB, approved by ECAB |
| CAB | Change Advisory Board — weekly meeting approving Normal changes |
| RPO | Recovery Point Objective — maximum acceptable data loss in time |
| RTO | Recovery Time Objective — maximum acceptable downtime |
| STAR | Situation, Task, Action, Result — interview answer format |

---

## Common Interview Questions

1. **"What is the difference between an incident and a problem?"** — An incident is an unplanned interruption — the focus is restoring service as quickly as possible. A problem is the underlying root cause. After the incident is resolved, problem management investigates WHY it happened and raises a change request to fix the root cause permanently. The same incident recurring multiple times means problem management was not completed.

2. **"Walk me through the change management process for upgrading a production server."** — Raise a change request in ServiceNow detailing the change, risk assessment, testing evidence, rollback plan, and maintenance window. Submit it to CAB at least 5 days in advance. CAB reviews and approves. On the day: take a VM snapshot, notify stakeholders, execute during the agreed window, verify the upgrade succeeded. Close the change with notes. If it fails, execute the rollback plan and open a new change to retry.

3. **"What are RPO and RTO and how do they affect infrastructure decisions?"** — RPO is how much data loss is acceptable — drives backup frequency (hourly backup = max 1-hour RPO). RTO is how long the service can be down — drives the recovery architecture (hot standby VM = minutes RTO, restore from backup = hours RTO). Together they determine what you build: Azure Backup for moderate RPO/RTO, Site Recovery for tighter RTO, FT for near-zero RTO.

---

## Congratulations

You have completed all 18 projects. You now have hands-on experience with:
- Azure fundamentals (ARM, VMs, Storage, VNet, Entra ID, Backup)
- Windows and Linux server administration
- Kubernetes and AKS
- Active Directory and Group Policy
- Patch management with Azure Update Manager
- Azure Monitor, KQL, and alerting
- Azure DevOps CI/CD pipelines
- Microsoft 365 and Hybrid Exchange
- Azure SQL administration
- Storage infrastructure (RAID, SAN, NAS, LVM, managed disks)
- VMware vSphere (ESXi, vCenter, vMotion, HA, DRS, FT)
- ITIL change management processes

Good luck with your interview at South London and Maudsley NHS Foundation Trust on 2 July 2026.
