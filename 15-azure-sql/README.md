# Project 15 — Azure SQL & SQL Administration

## What You Will Learn
- How to create an Azure SQL Server and database via Portal and CLI
- How to connect and run queries using the Portal Query Editor and sqlcmd
- How to manage SQL users, roles, and permissions
- How to perform backup and restore
- How Transparent Data Encryption (TDE) works

---

## Background — Read This First

**Azure SQL** is Microsoft's managed relational database service. Azure handles backups, high availability, patching, and hardware. You manage the schema, users, and data.

In the NHS, SQL databases back clinical systems (patient records, prescriptions, imaging systems). Understanding SQL administration is essential for the infrastructure engineer who supports these systems.

**Azure SQL concepts:**
| Term | What It Is |
|---|---|
| SQL Server (logical) | The container for databases — defines admin and firewall rules |
| SQL Database | A single database within a logical server |
| DTU pricing | Performance units — simpler, bundled compute and IO |
| vCore pricing | More granular — choose CPU cores and memory separately |
| TDE | Transparent Data Encryption — all data at rest is encrypted by default |

---

## Prerequisites
- Azure subscription, CLI logged in
- `sqlcmd` installed — available in Azure Cloud Shell by default

---

## Step 1 — Create a SQL Server and Database

### Portal (GUI)
1. Search **SQL databases** → **+ Create**
2. **Basics** tab:
   - Resource group: `rg-sql-lab` (create new)
   - Database name: `db-nhs-lab`
   - Server: click **Create new**
     - Server name: `sql-server-nhs-lab-<unique>` (must be globally unique)
     - Location: `UK South`
     - Authentication: **Use SQL authentication**
     - Admin login: `sqladmin`
     - Password: `SqlP@ss2026!!`
     - Click **OK**
   - Compute + storage: click **Configure database** → select **Basic** tier → **Apply**
3. **Networking** tab:
   - Connectivity: **Public endpoint**
   - Allow Azure services: **Yes**
   - Add current client IP: **Yes**
4. Click **Review + create** → **Create**

### CLI
```bash
az group create --name rg-sql-lab --location uksouth

SQL_SERVER_NAME="sql-lab-$(cat /proc/sys/kernel/random/uuid | head -c 8)"

az sql server create \
  --resource-group rg-sql-lab \
  --name $SQL_SERVER_NAME \
  --location uksouth \
  --admin-user sqladmin \
  --admin-password "SqlP@ss2026!!"

# Allow your current IP through the firewall
MY_IP=$(curl -s https://api.ipify.org)
az sql server firewall-rule create \
  --resource-group rg-sql-lab \
  --server $SQL_SERVER_NAME \
  --name AllowMyIP \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP

# Allow Azure services (for Cloud Shell and Azure services)
az sql server firewall-rule create \
  --resource-group rg-sql-lab \
  --server $SQL_SERVER_NAME \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Create the database
az sql db create \
  --resource-group rg-sql-lab \
  --server $SQL_SERVER_NAME \
  --name db-nhs-lab \
  --edition Basic \
  --capacity 5

echo "SQL Server: $SQL_SERVER_NAME.database.windows.net"
```

---

## Step 2 — Connect via Portal Query Editor

### Portal (GUI)
1. Navigate to `db-nhs-lab` database
2. In the left panel, click **Query editor (preview)** (under SQL database)
3. Log in with: `sqladmin` / `SqlP@ss2026!!`
4. You now have an interactive SQL editor in the browser — no tools to install

---

## Step 3 — Connect via sqlcmd

### Portal (GUI)
Use the Portal Query Editor (Step 2) as an alternative.

### CLI
```bash
# From Azure Cloud Shell (sqlcmd pre-installed) or your local machine
sqlcmd \
  -S $SQL_SERVER_NAME.database.windows.net \
  -U sqladmin \
  -P "SqlP@ss2026!!" \
  -d db-nhs-lab

# You are now in the sqlcmd prompt (1>)
```

---

## Step 4 — Create Tables and Insert Data

Run these in the Portal Query Editor or after connecting with sqlcmd:

```sql
-- Create a sample patient table (not real patient data!)
CREATE TABLE Patients (
    PatientId       INT PRIMARY KEY IDENTITY(1,1),
    NHSNumber       CHAR(10) NOT NULL,
    FirstName       NVARCHAR(50) NOT NULL,
    LastName        NVARCHAR(50) NOT NULL,
    DateOfBirth     DATE NOT NULL,
    WardId          INT,
    AdmissionDate   DATETIME DEFAULT GETDATE()
);

-- Insert sample records
INSERT INTO Patients (NHSNumber, FirstName, LastName, DateOfBirth, WardId)
VALUES
    ('1234567890', 'John',  'Smith',  '1985-03-15', 1),
    ('9876543210', 'Jane',  'Doe',    '1972-11-22', 2),
    ('5555555555', 'Bob',   'Jones',  '1990-07-04', 1);

-- Query the table
SELECT * FROM Patients;

-- Patients admitted to ward 1
SELECT FirstName, LastName, AdmissionDate
FROM Patients
WHERE WardId = 1
ORDER BY AdmissionDate DESC;
```

---

## Step 5 — Manage SQL Users and Permissions

**Principle of least privilege** — applications and users should only have the permissions they need.

```sql
-- Check current users
SELECT name, type_desc FROM sys.server_principals WHERE type IN ('S','U');

-- Create a read-only user for the clinical reporting app
CREATE LOGIN reporting_user WITH PASSWORD = 'R3p0rtPass!';
CREATE USER reporting_user FOR LOGIN reporting_user;

-- Grant read-only access
EXEC sp_addrolemember 'db_datareader', 'reporting_user';

-- Create an application user with read/write access
CREATE LOGIN app_user WITH PASSWORD = 'App0User!Pass';
CREATE USER app_user FOR LOGIN app_user;
EXEC sp_addrolemember 'db_datawriter', 'app_user';
EXEC sp_addrolemember 'db_datareader', 'app_user';

-- List users and roles
SELECT 
    u.name AS UserName,
    r.name AS RoleName
FROM sys.database_role_members rm
JOIN sys.database_principals r ON rm.role_principal_id = r.principal_id
JOIN sys.database_principals u ON rm.member_principal_id = u.principal_id;
```

**What you just learned:** `db_datareader` = SELECT only. `db_datawriter` = INSERT, UPDATE, DELETE. `db_owner` = full control. The clinical reporting app should only need `db_datareader` — it should never be able to modify patient records.

---

## Step 6 — Backup and Restore

### Portal (GUI)
**View automatic backups:**
1. Navigate to `db-nhs-lab` → **Backups** in the left panel (under Data management)
2. You see available restore points — Azure SQL automatically backs up every database:
   - Full backup: weekly
   - Differential backup: every 12 hours
   - Transaction log backup: every 5–12 minutes (enables point-in-time restore)
3. Click **Restore** at the top → set the restore point date/time → name the new database → **OK**

### CLI
```bash
# View available restore points
az sql db list-deleted-backups \
  --resource-group rg-sql-lab \
  --server $SQL_SERVER_NAME

# Restore to a point in time (creates a new database)
RESTORE_TIME=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)

az sql db restore \
  --resource-group rg-sql-lab \
  --server $SQL_SERVER_NAME \
  --name db-nhs-lab-restored \
  --dest-name db-nhs-lab-restored \
  --time $RESTORE_TIME \
  --source-database db-nhs-lab

# Check restore status
az sql db show \
  --resource-group rg-sql-lab \
  --server $SQL_SERVER_NAME \
  --name db-nhs-lab-restored \
  --query status
```

**What you just learned:** Azure SQL's automatic backups give you point-in-time restore to any second within the retention period (default 7 days, max 35 days). This satisfies NHS DR requirements. The restored database gets a new name — you then verify the data and optionally swap names.

---

## Step 7 — Transparent Data Encryption (TDE)

### Portal (GUI)
1. Navigate to `db-nhs-lab`
2. In the left panel, click **Transparent data encryption** (under Security)
3. You see TDE is **ON** by default — all data files, backup files, and log files are encrypted at rest
4. By default, Azure manages the encryption key (service-managed key)
5. For higher compliance requirements (NHS DSPT), click **Customer-managed key** to use your own key stored in Azure Key Vault

### CLI
```bash
# Verify TDE is enabled
az sql db tde show \
  --resource-group rg-sql-lab \
  --server $SQL_SERVER_NAME \
  --database db-nhs-lab

# Enable if not already on (it is on by default)
az sql db tde set \
  --resource-group rg-sql-lab \
  --server $SQL_SERVER_NAME \
  --database db-nhs-lab \
  --status Enabled
```

---

## Step 8 — Monitor Database Performance

### Portal (GUI)
1. Navigate to `db-nhs-lab` → **Overview**
2. The overview pane shows: DTU utilisation, Connections, Failed connections, Deadlocks
3. Click **Query Performance Insight** (under Intelligent Performance in the left panel)
4. You see top queries by CPU, duration, and execution count — useful for diagnosing slow queries

### CLI
```bash
# Get database metrics
az monitor metrics list \
  --resource $(az sql db show \
    --resource-group rg-sql-lab \
    --server $SQL_SERVER_NAME \
    --name db-nhs-lab --query id --output tsv) \
  --metric "dtu_consumption_percent" \
  --interval PT5M \
  --output table
```

---

## Step 9 — Clean Up

### Portal (GUI)
Navigate to `rg-sql-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-sql-lab --yes --no-wait
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Logical SQL Server | Container resource — holds databases, defines admin and firewall |
| SQL Database | The actual database inside the logical server |
| Firewall rules | Control which IPs can connect to the SQL Server |
| db_datareader | SQL role — SELECT permissions only |
| db_datawriter | SQL role — INSERT, UPDATE, DELETE permissions |
| TDE | Transparent Data Encryption — all data at rest encrypted by default |
| Point-in-time restore | Restore to any second within the retention period |
| Auto backups | Azure SQL backs up automatically — full weekly, differential 12h, logs every 5-12min |

---

## Common Interview Questions

1. **"A clinical system database is running slowly. How do you start diagnosing it?"** — Check Query Performance Insight in the Portal for slow or high-CPU queries. Check DTU utilisation — if consistently at 100%, the database is under-provisioned. Check for blocking and deadlocks in the Activity Monitor. Run `SELECT * FROM sys.dm_exec_requests` to see currently running queries.

2. **"Explain Transparent Data Encryption and why it matters in the NHS."** — TDE encrypts all data at rest — database files, backup files, transaction logs. This means if someone steals the physical disk or backup file, they cannot read the data without the encryption key. For the NHS, TDE satisfies DSPT requirements for protecting patient data at rest.

3. **"A developer accidentally deleted a whole table. How do you recover it?"** — Use point-in-time restore to create a new copy of the database from a timestamp before the deletion. Verify the table exists in the restored database. Extract just the table data (`SELECT INTO` or `INSERT SELECT`) and import it into the production database. Never just swap databases without verifying the restore first.

---

## Next Steps
Move on to **Project 16 — Storage Infrastructure**.
