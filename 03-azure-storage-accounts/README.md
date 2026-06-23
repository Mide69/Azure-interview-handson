# Project 03 — Azure Storage Accounts

## What You Will Learn
- The four Azure storage services and when to use each
- How to create and configure storage accounts via Portal and CLI
- How to upload, download, and manage blobs
- How to create Azure File Shares (cloud network drives)
- How to generate SAS tokens for secure time-limited access
- How to set access tiers and enable soft delete

---

## Background — Read This First

A **Storage Account** is the container for all Azure storage services. Inside one account you get four services:

| Service | What It Stores | Real-World Use |
|---|---|---|
| **Blob Storage** | Unstructured objects (any file) | Backups, images, log files, VHDs |
| **File Storage** | SMB/NFS file shares | Replacing on-prem network shares |
| **Queue Storage** | Message queues | Async communication between apps |
| **Table Storage** | Key-value NoSQL data | Structured non-relational data |

**Replication options** (how many copies Azure keeps):
- `LRS` — 3 copies in one datacentre. Cheapest.
- `ZRS` — 3 copies across 3 availability zones in one region.
- `GRS` — LRS + async copies to a paired region (UK South → UK West).
- `RAGRS` — GRS + you can read from the secondary region.

For NHS critical data: use at minimum GRS.

---

## Prerequisites
- Azure subscription, CLI logged in

---

## Step 1 — Create a Resource Group and Storage Account

### Portal (GUI)
1. Search **Storage accounts** → click **+ Create**
2. Fill in **Basics** tab:
   - Resource group: `rg-storage-lab` (create new)
   - Storage account name: `stlabstorage001` (globally unique, lowercase, 3–24 chars)
   - Region: `UK South`
   - Performance: **Standard**
   - Redundancy: **Locally-redundant storage (LRS)**
3. **Advanced tab** — note the options: require HTTPS, minimum TLS version, blob access tier (Hot/Cool)
4. **Data protection tab** — note soft delete options (enable these in production)
5. Click **Review + create** → **Create**
6. Click **Go to resource** — explore the left panel

### CLI
```bash
az group create --name rg-storage-lab --location uksouth

STORAGE_NAME="stlabstorage$(date +%s | tail -c 6)"
echo "Storage name: $STORAGE_NAME"

az storage account create \
  --name $STORAGE_NAME \
  --resource-group rg-storage-lab \
  --location uksouth \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot

az storage account show \
  --name $STORAGE_NAME \
  --resource-group rg-storage-lab \
  --output table
```

**What you just learned:** `StorageV2` is the recommended kind — it supports all services and features. `Hot` access tier is for frequently accessed data; `Cool` costs less for storage but more per read.

---

## Step 2 — Get the Storage Account Key

### Portal (GUI)
1. On the storage account page, click **Access keys** in the left panel (under Security + networking)
2. Click **Show keys** — you see two keys (key1 and key2) and their connection strings
3. Click the copy icon next to **key1** — this is your full-access credential
4. **Warning:** Treat these like passwords. Anyone with this key has complete access to everything in the account.

### CLI
```bash
az storage account keys list \
  --account-name $STORAGE_NAME \
  --resource-group rg-storage-lab \
  --output table

STORAGE_KEY=$(az storage account keys list \
  --account-name $STORAGE_NAME \
  --resource-group rg-storage-lab \
  --query "[0].value" \
  --output tsv)
```

**What you just learned:** Storage account keys grant **full access to everything** in the account. In real environments, use Managed Identities or SAS tokens instead of exposing the key. The key1/key2 split allows key rotation — rotate one while the other remains active, then rotate the second.

---

## Step 3 — Blob Storage: Create a Container

### Portal (GUI)
1. On the storage account page, click **Containers** in the left panel (under Data storage)
2. Click **+ Container**
3. Name: `mycontainer`
4. Public access level: **Private (no anonymous access)**
5. Click **Create**
6. The container appears in the list — click it to open

### CLI
```bash
az storage container create \
  --name mycontainer \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY \
  --public-access off

az storage container list \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY \
  --output table
```

**What you just learned:** `public-access off` means nobody can read without authentication. Never set public access for NHS patient data. The two other options — `blob` (anonymous read for blobs) and `container` (anonymous list + read) — should only be used for truly public data like a public-facing website.

---

## Step 4 — Upload Files to Blob Storage

### Portal (GUI)
1. Inside `mycontainer`, click **Upload**
2. Browse to a file on your computer (any file — a text file, image, document)
3. Expand **Advanced** — notice options: blob type (Block/Append/Page), access tier
4. Click **Upload**
5. The blob appears in the list — click it to see its URL, size, and metadata

### CLI
```bash
echo "Hello from Azure Storage Lab" > testfile.txt
echo "Second test file" > testfile2.txt

az storage blob upload \
  --container-name mycontainer \
  --file testfile.txt \
  --name testfile.txt \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY

az storage blob upload \
  --container-name mycontainer \
  --file testfile2.txt \
  --name testfile2.txt \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY

az storage blob list \
  --container-name mycontainer \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY \
  --output table
```

**What you just learned:** Blobs are flat (no real folders) but you can simulate directory structure using `/` in the blob name — e.g. `backups/2026/july/server01.bak`. The Portal renders these as folders but they're just name prefixes.

---

## Step 5 — Download a Blob

### Portal (GUI)
1. In the container, click on `testfile.txt`
2. Click **Download** at the top of the blob properties panel
3. Your browser downloads the file

### CLI
```bash
az storage blob download \
  --container-name mycontainer \
  --name testfile.txt \
  --file downloaded-testfile.txt \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY

cat downloaded-testfile.txt
```

---

## Step 6 — Generate a SAS Token (Secure Time-Limited URL)

A SAS (Shared Access Signature) token gives someone temporary, scoped access to a blob without sharing the account key.

### Portal (GUI)
1. Click on `testfile.txt` in the container
2. In the blob properties panel, click **Generate SAS** tab
3. Set:
   - Permissions: **Read** only (uncheck everything else)
   - Start: now
   - Expiry: 2 hours from now
4. Click **Generate SAS token and URL**
5. Copy the **Blob SAS URL** — paste it into a new browser tab — the file downloads without any authentication
6. After the expiry time, the URL stops working

### CLI
```bash
END_TIME=$(date -u -d "+2 hours" +%Y-%m-%dT%H:%MZ 2>/dev/null || \
           date -u -v+2H +%Y-%m-%dT%H:%MZ)

SAS_TOKEN=$(az storage blob generate-sas \
  --container-name mycontainer \
  --name testfile.txt \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY \
  --permissions r \
  --expiry $END_TIME \
  --output tsv)

BLOB_URL="https://${STORAGE_NAME}.blob.core.windows.net/mycontainer/testfile.txt?${SAS_TOKEN}"
echo "URL: $BLOB_URL"
curl "$BLOB_URL"
```

**What you just learned:** SAS tokens follow the principle of least privilege. You grant only: the permission needed (r=read, w=write, d=delete, l=list), on only the specific resource needed, for only as long as needed. This is far safer than handing someone the account key. Use SAS tokens whenever you need to share Azure storage access with a third party.

---

## Step 7 — Azure File Shares (Cloud Network Drive)

### Portal (GUI)
1. On the storage account page, click **File shares** in the left panel
2. Click **+ File share**
3. Name: `labfileshare`
4. Quota: `10` GiB
5. Click **Create**
6. Click `labfileshare` to open it
7. Click **+ Add directory** → name it `documents` → click **OK**
8. Click into `documents` → click **Upload** → upload a file
9. To get the mount command: click **Connect** at the top — the Portal shows you the exact PowerShell or Linux mount command for your OS

### CLI
```bash
az storage share create \
  --name labfileshare \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY \
  --quota 10

az storage directory create \
  --share-name labfileshare \
  --name documents \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY

az storage file upload \
  --share-name labfileshare \
  --path documents/testfile.txt \
  --source testfile.txt \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY

az storage file list \
  --share-name labfileshare \
  --path documents \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY \
  --output table
```

**What you just learned:** Azure Files is a managed SMB/NFS file share. Windows machines can map it as a drive letter (`Z:\`). Linux machines can mount it as a CIFS share. This is a direct replacement for an on-premises NAS or file server — no hardware to manage, and it replicates automatically.

---

## Step 8 — Change Blob Access Tier

Move blobs to cheaper storage tiers when they are accessed less frequently.

### Portal (GUI)
1. Click `testfile2.txt` in the container
2. In the blob properties panel, click **Change tier**
3. Select **Cool** → click **Save**
4. The tier changes immediately — notice the change in the blob details

### CLI
```bash
az storage blob set-tier \
  --container-name mycontainer \
  --name testfile2.txt \
  --tier Cool \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY

az storage blob show \
  --container-name mycontainer \
  --name testfile2.txt \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY \
  --query "properties.blobTier" \
  --output tsv
```

**What you just learned:** Hot = frequent access (higher storage cost, lower read cost). Cool = infrequent access (lower storage, higher read). Archive = very rare access (cheapest storage, hours to rehydrate, high read cost). Move old backups and logs to Cool or Archive automatically using Lifecycle Management policies.

---

## Step 9 — Enable Soft Delete (Protect Against Accidental Deletion)

### Portal (GUI)
1. On the storage account page, click **Data protection** in the left panel
2. Under **Blob soft delete**, tick **Enable soft delete for blobs**
3. Set retention: **30 days**
4. Under **Container soft delete**, also enable it for 30 days
5. Click **Save**
6. Test it: delete `testfile2.txt` from the container
7. In the container toolbar, click **Show deleted blobs** — the file still appears with a **Deleted** status and can be **Undeleted** within 30 days

### CLI
```bash
az storage blob service-properties delete-policy update \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY \
  --enable true \
  --days-retained 30

az storage blob service-properties delete-policy show \
  --account-name $STORAGE_NAME \
  --account-key $STORAGE_KEY
```

**What you just learned:** In healthcare, accidentally deleting patient data is a serious incident. Soft delete gives you a 30-day recovery window. Always enable this on every production storage account. It is a quick win for a new starter — you can enable it without any downtime.

---

## Step 10 — Clean Up

### Portal (GUI)
Navigate to `rg-storage-lab` → **Delete resource group** → type name → **Delete**

### CLI
```bash
az group delete --name rg-storage-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Blob Storage | Object store — any file, accessed via HTTP/HTTPS |
| File Storage | SMB/NFS share — mount as a drive letter or filesystem |
| SAS Token | Time-limited scoped URL — safer than sharing the account key |
| Access Tier | Hot/Cool/Archive — controls cost vs. access speed |
| Soft Delete | 30-day recovery window after deletion — protect NHS data |
| LRS / GRS | Replication — GRS copies to a paired region |
| Container | Flat namespace for blobs — simulate folders with `/` in names |

---

## Common Interview Questions

1. **"What is the difference between Blob and File storage?"** — Blob is object storage accessed via HTTP/HTTPS — good for backups, images, and files consumed by apps. File storage is SMB/NFS accessed like a network share — good for migrating on-premises file servers. Blob cannot be mounted as a drive; File can.

2. **"How would you securely give a third party temporary access to a specific file?"** — Generate a SAS token scoped to that one blob, with read-only permission and an expiry time. Share the SAS URL. After expiry it automatically stops working — no key rotation needed.

3. **"What storage replication would you use for NHS patient data backups?"** — At minimum GRS (Geo-Redundant Storage) — 6 copies total, automatically replicated to the UK West paired region. If you need to read from the secondary during a regional outage, use RAGRS.

---

## Next Steps
Move on to **Project 04 — Azure Virtual Networks & NSGs**.
