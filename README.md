# Azure Interview Preparation — Junior Infrastructure & Cloud Engineer

**Interview:** South London and Maudsley NHS Foundation Trust
**Role:** Junior Infrastructure and Cloud Engineer (Band 6)
**Date:** 2 July 2026, 11:00 (Microsoft Teams)

---

## How to Use This Repo

Each project folder contains a `README.md` that teaches the topic from scratch, written for a junior engineer. Every step is **console/CLI only** — no portal clicking.

Work through the tiers in order. Tier 1 is most likely to come up in the interview. Do the hands-on commands, don't just read.

**Cost tip:** Always delete resource groups when finished with a project:
```bash
az group delete --name rg-<project>-lab --yes --no-wait
```

---

## Project Index

### Tier 1 — Must Know (Core of the Role)

| # | Project | Key Tools | Job Spec Match |
|---|---|---|---|
| [01](./01-azure-arm-resource-manager/) | Azure ARM & Resource Manager | `az deployment`, ARM JSON templates | "Azure ARM" — named directly |
| [02](./02-azure-virtual-machines/) | Azure Virtual Machines (IaaS) | `az vm create/start/stop/resize` | "Virtualized platforms" |
| [03](./03-azure-storage-accounts/) | Azure Storage Accounts | `az storage`, SAS tokens, blob/file | "Storage infrastructure" |
| [04](./04-azure-virtual-networks-nsg/) | Virtual Networks & NSGs | `az network vnet/nsg/peering` | "Proxy, networking" |
| [05](./05-azure-entra-id/) | Azure Entra ID & RBAC | `az ad`, `az role assignment` | "Identity Services, Domain Services" |
| [06](./06-azure-backup/) | Azure Backup | `az backup vault/policy/job` | "Backup services" |

### Tier 2 — Strong Supporting (You'll Be Asked Something Here)

| # | Project | Key Tools | Job Spec Match |
|---|---|---|---|
| [07](./07-windows-server-administration/) | Windows Server Administration | PowerShell, `az vm run-command` | "Windows and Linux systems" |
| [08](./08-linux-server-administration/) | Linux Server Administration | Bash, systemd, SSH, apt | "Windows and Linux systems" |
| [09](./09-kubernetes-aks/) | Kubernetes & AKS | `kubectl`, `az aks` | "Containerized platforms, Kubernetes" |
| [10](./10-active-directory-group-policy/) | Active Directory & Group Policy | `New-ADUser`, `New-GPO`, DNS/DHCP | "Domain Services, GPO, FSMO" |
| [11](./11-patch-management/) | Patch Management | `az vm assess-patches`, apt, WSUS | "Patch management solutions" |

### Tier 3 — Round It Out

| # | Project | Key Tools | Job Spec Match |
|---|---|---|---|
| [12](./12-azure-monitor-alerting/) | Azure Monitor & Alerting | KQL, `az monitor metrics alert` | "Monitoring services" |
| [13](./13-azure-devops-cicd/) | Azure DevOps & CI/CD | `az pipelines`, YAML | "DevOps, continuous deployment" |
| [14](./14-microsoft365-hybrid-exchange/) | Microsoft 365 & Hybrid Exchange | ExchangeOnline PowerShell | "Office 365, Hybrid Email" |
| [15](./15-azure-sql/) | Azure SQL & SQL Administration | `az sql`, `sqlcmd`, KQL | "SQL Administration" |
| [16](./16-storage-infrastructure/) | Storage Infrastructure (SAN/NAS/RAID) | `az disk`, LVM, fio | "Storage infrastructure Administration" |
| [17](./17-vmware-vsphere-basics/) | VMware vSphere Basics | PowerCLI, ESXi Shell, `esxtop` | "VMware VC" — essential qualification |
| [18](./18-itil-change-management/) | ITIL & Change Management | Process knowledge + STAR answers | "ITIL standard of Service Management" |

---

## Quick Reference — Commands You Must Know Cold

### Azure CLI

```bash
# Login and account
az login
az account show
az account set --subscription "NAME"

# Resource groups
az group create --name rg-name --location uksouth
az group delete --name rg-name --yes --no-wait

# Virtual Machines
az vm create --resource-group RG --name VM --image Ubuntu2204 --size Standard_B1s --admin-username user --generate-ssh-keys
az vm start/stop/deallocate/restart --resource-group RG --name VM
az vm run-command invoke --resource-group RG --name VM --command-id RunShellScript --scripts "command"
az vm assess-patches --resource-group RG --name VM

# Storage
az storage account create --name NAME --resource-group RG --sku Standard_LRS --kind StorageV2
az storage blob upload --container-name C --file FILE --name BLOBNAME --account-name A --account-key K

# Networking
az network vnet create --name vnet --resource-group RG --address-prefix 10.0.0.0/16
az network nsg rule create --nsg-name NSG --resource-group RG --name RULE --priority 100 --port 22 --access Allow --direction Inbound

# Identity
az ad user create --display-name NAME --user-principal-name UPN --password PASS
az role assignment create --assignee PRINCIPAL_ID --role "Contributor" --scope SCOPE
```

### kubectl

```bash
kubectl get pods/deployments/services/nodes -n NAMESPACE
kubectl describe pod POD_NAME -n NAMESPACE
kubectl logs POD_NAME -n NAMESPACE --previous
kubectl exec -it POD_NAME -n NAMESPACE -- /bin/bash
kubectl scale deployment DEPLOY --replicas=3 -n NAMESPACE
kubectl rollout undo deployment/DEPLOY -n NAMESPACE
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
```

### PowerShell (Active Directory)

```powershell
Get-ADUser -Filter * | Select-Object Name, Enabled
New-ADUser -Name "Name" -SamAccountName "user" -Path "OU=IT,DC=lab,DC=local" -AccountPassword $pass -Enabled $true
Get-ADGroupMember -Identity "GroupName"
Add-ADGroupMember -Identity "Group" -Members user1
New-GPO -Name "PolicyName"
New-GPLink -Name "PolicyName" -Target "OU=IT,DC=lab,DC=local"
netdom query fsmo
```

---

## The 10 Things Most Likely to Come Up in Interview

1. **Stop vs Deallocate** — Stop still bills for compute. Deallocate does not.
2. **FSMO Roles** — Know all 5, especially PDC Emulator.
3. **RAID levels** — RAID 10 for databases, RAID 5 for general use, never RAID 0 for production.
4. **Incident vs Problem vs Change** — Restore service fast (incident), find root cause (problem), controlled modification (change).
5. **vMotion** — Live VM migration between hosts with zero downtime.
6. **NSG priority** — Lower number = higher priority. Processed first.
7. **Managed Identity vs Service Principal** — Managed Identity = no credentials to manage.
8. **KQL basics** — `| where`, `| summarize`, `| project`, `ago(1h)`.
9. **Entra Connect** — Syncs on-prem AD to cloud every 30 minutes. Delta sync is the default.
10. **SAS token** — Time-limited, scoped access to Azure storage without sharing the account key.

---

*Interview date: 2 July 2026 at 11:00 via Microsoft Teams*
*Location if in-person: St Pauls, Bromley BR2 0EZ*
