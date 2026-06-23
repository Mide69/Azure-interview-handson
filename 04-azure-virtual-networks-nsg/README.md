# Project 04 — Azure Virtual Networks & NSGs

## What You Will Learn
- How Azure networking works (VNets, subnets, NICs)
- How to create VNets and subnets via Portal and CLI
- How to build NSG rules to control traffic (Azure's firewall)
- How to peer two VNets so they communicate privately
- How to inspect effective security rules on a NIC
- Network segmentation — separating web, app, and data tiers

---

## Background — Read This First

**Virtual Network (VNet)** — A private network inside Azure. Your resources talk to each other here. Nothing comes in from the internet unless you explicitly allow it.

**Subnet** — A logical subdivision of a VNet. You split a VNet into subnets to apply different security rules. A classic 3-tier pattern:
- `subnet-web` — web/front-end servers (allow HTTP/HTTPS from internet)
- `subnet-app` — application servers (only allow traffic from web subnet)
- `subnet-data` — databases (only allow traffic from app subnet)

**NSG (Network Security Group)** — A set of inbound and outbound firewall rules. Lower priority number = processed first. NSGs can attach to a **subnet** (affects all resources in it) or a **NIC** (affects only that VM).

**CIDR notation** — IP ranges written as `10.0.0.0/16`. The `/16` means 65,536 addresses. `/24` means 256 addresses.

---

## Prerequisites
- Azure subscription, CLI logged in
- Completed Project 01 (understands resource groups)

---

## Step 1 — Create a Resource Group

### Portal (GUI)
Search **Resource groups** → **+ Create** → Name: `rg-network-lab` | Region: `UK South` → **Create**

### CLI
```bash
az group create --name rg-network-lab --location uksouth
```

---

## Step 2 — Create a Virtual Network with Two Subnets

### Portal (GUI)
1. Search **Virtual networks** → **+ Create**
2. **Basics** tab:
   - Resource group: `rg-network-lab`
   - Name: `vnet-lab`
   - Region: `UK South`
3. **IP Addresses** tab:
   - IPv4 address space: `10.10.0.0/16`
   - Delete the default subnet
   - Click **+ Add subnet**:
     - Name: `subnet-web`
     - Address range: `10.10.1.0/24`
     - Click **Add**
   - Click **+ Add subnet** again:
     - Name: `subnet-app`
     - Address range: `10.10.2.0/24`
     - Click **Add**
4. Click **Review + create** → **Create**
5. Once created, click **Go to resource** → click **Subnets** in the left panel to confirm both subnets exist

### CLI
```bash
az network vnet create \
  --name vnet-lab \
  --resource-group rg-network-lab \
  --location uksouth \
  --address-prefix 10.10.0.0/16

az network vnet subnet create \
  --name subnet-web \
  --vnet-name vnet-lab \
  --resource-group rg-network-lab \
  --address-prefix 10.10.1.0/24

az network vnet subnet create \
  --name subnet-app \
  --vnet-name vnet-lab \
  --resource-group rg-network-lab \
  --address-prefix 10.10.2.0/24

az network vnet subnet list \
  --vnet-name vnet-lab \
  --resource-group rg-network-lab \
  --output table
```

**What you just learned:** VNet uses `10.10.0.0/16` — a private IP range. The web subnet gets `10.10.1.x` addresses, the app subnet gets `10.10.2.x`. Resources in different subnets in the same VNet can communicate by default — NSGs are what restrict that.

---

## Step 3 — Create an NSG for the Web Subnet

### Portal (GUI)
1. Search **Network security groups** → **+ Create**
2. Resource group: `rg-network-lab` | Name: `nsg-web` | Region: `UK South`
3. Click **Review + create** → **Create** → **Go to resource**

**Add inbound rules:**
4. Click **Inbound security rules** in the left panel → **+ Add**
5. Rule 1 — Allow HTTP:
   - Source: `Any` | Source port: `*`
   - Destination: `Any` | Destination port: `80`
   - Protocol: `TCP` | Action: `Allow`
   - Priority: `100` | Name: `Allow-HTTP`
   - Click **Add**
6. Rule 2 — Allow HTTPS (priority 110, port 443, same as above)
7. Rule 3 — Allow SSH from your IP only:
   - Source: **IP Addresses** | Source IP: `your-ip-address/32`
   - Destination port: `22` | Priority: `120` | Name: `Allow-SSH-MyIP`
   - Click **Add**
8. Rule 4 — Deny everything else:
   - Source: `Any` | Destination port: `*`
   - Action: `Deny` | Priority: `4000` | Name: `Deny-All-Inbound`
   - Click **Add**

### CLI
```bash
az network nsg create \
  --name nsg-web \
  --resource-group rg-network-lab \
  --location uksouth

MY_IP=$(curl -s https://ifconfig.me)

az network nsg rule create --nsg-name nsg-web --resource-group rg-network-lab \
  --name Allow-HTTP --priority 100 --protocol Tcp --direction Inbound \
  --source-address-prefixes Internet --destination-port-ranges 80 --access Allow

az network nsg rule create --nsg-name nsg-web --resource-group rg-network-lab \
  --name Allow-HTTPS --priority 110 --protocol Tcp --direction Inbound \
  --source-address-prefixes Internet --destination-port-ranges 443 --access Allow

az network nsg rule create --nsg-name nsg-web --resource-group rg-network-lab \
  --name Allow-SSH-MyIP --priority 120 --protocol Tcp --direction Inbound \
  --source-address-prefixes $MY_IP --destination-port-ranges 22 --access Allow

az network nsg rule create --nsg-name nsg-web --resource-group rg-network-lab \
  --name Deny-All-Inbound --priority 4000 --protocol "*" --direction Inbound \
  --source-address-prefixes "*" --destination-port-ranges "*" --access Deny
```

**What you just learned:** NSG rules are processed lowest-priority-number-first. Azure has default rules at 65000+ that you cannot delete. Your custom rules at 100–4000 override them. The deny-all at 4000 makes the policy explicit — anything not in your allow list is blocked.

---

## Step 4 — Create an NSG for the App Subnet

The app subnet should only accept traffic from the web subnet — not from the internet.

### Portal (GUI)
1. Create a second NSG named `nsg-app`
2. Add inbound rule:
   - Source: **IP Addresses** | Source IP: `10.10.1.0/24` (the web subnet range)
   - Destination port: `*` | Protocol: `Any` | Action: `Allow` | Priority: `100` | Name: `Allow-From-WebSubnet`
3. Add deny-all rule: priority `4000`, action `Deny`, all ports

### CLI
```bash
az network nsg create \
  --name nsg-app \
  --resource-group rg-network-lab \
  --location uksouth

az network nsg rule create --nsg-name nsg-app --resource-group rg-network-lab \
  --name Allow-From-WebSubnet --priority 100 --protocol Tcp --direction Inbound \
  --source-address-prefixes 10.10.1.0/24 --destination-port-ranges "*" --access Allow

az network nsg rule create --nsg-name nsg-app --resource-group rg-network-lab \
  --name Deny-All-Inbound --priority 4000 --protocol "*" --direction Inbound \
  --source-address-prefixes "*" --destination-port-ranges "*" --access Deny
```

**What you just learned:** This is **network segmentation** — a security best practice mandated by NHS Cyber Security standards. Even if an attacker compromises a web server, they hit another firewall before reaching the app layer. Defence in depth.

---

## Step 5 — Associate NSGs with Subnets

### Portal (GUI)
1. Navigate to `vnet-lab` → click **Subnets**
2. Click `subnet-web`
3. Under **Network security group**, click the dropdown and select `nsg-web`
4. Click **Save**
5. Repeat for `subnet-app` → select `nsg-app` → **Save**

### CLI
```bash
az network vnet subnet update \
  --name subnet-web \
  --vnet-name vnet-lab \
  --resource-group rg-network-lab \
  --network-security-group nsg-web

az network vnet subnet update \
  --name subnet-app \
  --vnet-name vnet-lab \
  --resource-group rg-network-lab \
  --network-security-group nsg-app
```

---

## Step 6 — Deploy a VM into the Web Subnet

### Portal (GUI)
1. Create a VM (as in Project 02) but on the **Networking** tab:
   - Virtual network: `vnet-lab`
   - Subnet: `subnet-web`
   - NIC network security group: **None** (the subnet NSG handles it)
   - Public IP: create new
2. Complete the VM creation

### CLI
```bash
az vm create \
  --resource-group rg-network-lab \
  --name vm-web \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name vnet-lab \
  --subnet subnet-web \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --nsg ""

WEB_IP=$(az vm show \
  --resource-group rg-network-lab \
  --name vm-web \
  --show-details \
  --query publicIps \
  --output tsv)
echo "Web VM IP: $WEB_IP"
```

---

## Step 7 — Inspect Effective Security Rules

This is your #1 troubleshooting tool when connectivity is broken.

### Portal (GUI)
1. Navigate to the `vm-web` VM
2. In the left panel, click **Networking** (under Settings)
3. You see the NIC listed — click the NIC name (e.g. `vm-web729_z1`)
4. In the NIC's left panel, click **Effective security rules**
5. This shows the combined view of all NSG rules applying to this NIC — subnet NSG + NIC NSG merged, with priority order. This is exactly what Azure is enforcing.
6. You can see which rule allows or blocks each type of traffic

### CLI
```bash
NIC_NAME=$(az vm show \
  --resource-group rg-network-lab \
  --name vm-web \
  --query "networkProfile.networkInterfaces[0].id" \
  --output tsv | xargs basename)

az network nic show-effective-nsg \
  --name $NIC_NAME \
  --resource-group rg-network-lab \
  --output json
```

**What you just learned:** Effective security rules is always your first stop when a VM can't be reached or is blocking unexpected traffic. It shows you the combined result of all NSGs — no guessing.

---

## Step 8 — Create a Second VNet and Peer Them

VNet peering connects two VNets so resources communicate using private IPs — without going over the internet.

### Portal (GUI)
**Create second VNet:**
1. Search **Virtual networks** → **+ Create**
2. Name: `vnet-lab2` | Address space: `10.20.0.0/16`
3. Add subnet: `subnet-main` / `10.20.1.0/24`
4. Create it

**Create peering from vnet-lab to vnet-lab2:**
5. Navigate to `vnet-lab` → **Peerings** in the left panel → **+ Add**
6. Fill in:
   - Peering link name (this VNet): `peer-lab-to-lab2`
   - Peering link name (remote VNet): `peer-lab2-to-lab`
   - Virtual network: `vnet-lab2`
   - Leave all other defaults (Allow traffic checked)
7. Click **Add** — this creates **both directions** of the peering automatically
8. Both peerings appear in the list with status **Connected**

### CLI
```bash
az network vnet create \
  --name vnet-lab2 \
  --resource-group rg-network-lab \
  --location uksouth \
  --address-prefix 10.20.0.0/16

az network vnet subnet create \
  --name subnet-main \
  --vnet-name vnet-lab2 \
  --resource-group rg-network-lab \
  --address-prefix 10.20.1.0/24

# Peering: vnet-lab → vnet-lab2
az network vnet peering create \
  --name peer-lab-to-lab2 \
  --vnet-name vnet-lab \
  --resource-group rg-network-lab \
  --remote-vnet vnet-lab2 \
  --allow-vnet-access

# Peering: vnet-lab2 → vnet-lab (must create both directions)
az network vnet peering create \
  --name peer-lab2-to-lab \
  --vnet-name vnet-lab2 \
  --resource-group rg-network-lab \
  --remote-vnet vnet-lab \
  --allow-vnet-access

az network vnet peering list \
  --vnet-name vnet-lab \
  --resource-group rg-network-lab \
  --output table
```

**What you just learned:** VNet address ranges must **not overlap** — that is why we used `10.10.0.0/16` and `10.20.0.0/16`. VNet peering is NOT transitive: if A peers with B, and B peers with C, A cannot reach C via B. You must peer A and C directly.

---

## Step 9 — Enable Network Watcher

Network Watcher provides diagnostics tools for your network.

### Portal (GUI)
1. Search **Network Watcher** in the top bar → click it
2. In the left panel, click **IP flow verify**
3. Fill in: VM `vm-web`, Direction `Inbound`, Protocol `TCP`, Remote IP (your IP), Remote port `any`, Local port `22`
4. Click **Check** — it tells you which NSG rule allows or denies this traffic
5. Also explore **Topology** in the left panel — it draws a visual map of your network resources

### CLI
```bash
az network watcher configure \
  --resource-group NetworkWatcherRG \
  --locations uksouth \
  --enabled true
```

---

## Step 10 — Clean Up

### Portal (GUI)
Navigate to `rg-network-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-network-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| VNet | Private network in Azure — isolated from other customers |
| Subnet | Segment of a VNet — apply different NSG rules per segment |
| NSG | Stateful firewall — lower priority number = processed first |
| Effective security rules | Combined view of all NSGs on a NIC — first debugging step |
| VNet Peering | Private connectivity between VNets — no internet, no gateway needed |
| Network segmentation | Separating web/app/data tiers with different NSGs — defence in depth |
| CIDR | IP range notation — ranges must not overlap for peering |

---

## Common Interview Questions

1. **"What is the difference between an NSG on a subnet versus on a NIC?"** — A subnet NSG applies to all resources in that subnet. A NIC NSG applies only to that specific VM. When both exist, Azure evaluates them together — Effective Security Rules shows the combined result. Subnet NSGs are recommended for consistent policy; NIC NSGs for per-VM exceptions.

2. **"Two VMs in different VNets cannot communicate. What would you check?"** — Are the VNets peered? Are the address ranges non-overlapping? Are NSG rules on both sides allowing the traffic? Use Network Watcher **IP flow verify** to identify which rule is blocking. Check effective security rules on both NICs.

3. **"Why should you never expose port 22 or 3389 directly to the internet?"** — These ports are constantly scanned and brute-forced by automated bots. Instead: use Azure Bastion (a managed jump host with no public IP on the VM), or restrict to your organisation's specific IP range only.

---

## Next Steps
Move on to **Project 05 — Azure Entra ID**.
