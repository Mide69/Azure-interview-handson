# Project 14 — Microsoft 365 & Hybrid Exchange

## What You Will Learn
- What Hybrid Exchange is and why NHS organisations use it
- How to manage M365 users and mailboxes from the Admin Centre (GUI) and PowerShell
- How to manage Exchange Online — shared mailboxes, distribution groups, transport rules
- What Entra Connect (Azure AD Connect) does and how it works
- How to trace mail flow and investigate mail delivery issues

---

## Background — Read This First

**Hybrid Exchange** means the NHS trust runs Exchange Server on-premises AND Exchange Online (Microsoft 365) at the same time. Some mailboxes live on-premises, some in the cloud.

**Why the NHS uses it:** Migrating 50,000 mailboxes to the cloud takes years. Hybrid lets you migrate in waves while keeping email working for everyone throughout the migration.

**Entra Connect (formerly Azure AD Connect):** Synchronises on-premises Active Directory users to Entra ID (Azure AD) every 30 minutes. This is what allows the same username and password to work for both the on-premises PC login and Microsoft 365.

**Key components:**
| Component | What It Does |
|---|---|
| Exchange Server (on-prem) | Hosts on-premises mailboxes |
| Exchange Online | Hosts cloud mailboxes in Microsoft 365 |
| Entra Connect | Syncs AD users from on-prem to Entra ID |
| Exchange Online PowerShell | Command-line management of Exchange Online |
| Microsoft 365 Admin Centre | GUI portal for managing M365 |

---

## Prerequisites
- Microsoft 365 trial subscription (sign up free at microsoft.com/en-gb/microsoft-365/business/compare-all-plans)
- Exchange Online PowerShell module installed

---

## Step 1 — Install Exchange Online PowerShell Module

### Portal (GUI)
All Exchange Online management can be done from the Exchange Admin Centre — no module installation needed.

### PowerShell
```powershell
# Install the Exchange Online module
Install-Module -Name ExchangeOnlineManagement -Force -AllowClobber

# Verify installation
Get-Module -ListAvailable ExchangeOnlineManagement

# Connect (opens a browser for authentication)
Connect-ExchangeOnline -UserPrincipalName admin@yourtenant.onmicrosoft.com

# Verify connected
Get-ConnectionInformation
```

---

## Step 2 — Explore the Microsoft 365 Admin Centre

### Portal (GUI)
1. Go to **admin.microsoft.com** → sign in as your admin account
2. Navigate the left panel:
   - **Users** → **Active users** — see all licensed users
   - **Groups** → **Active groups** — see distribution lists and security groups
   - **Billing** → **Licenses** — see assigned licence counts
   - **Settings** → **Domains** — see your verified domains
3. Click **Show all** at the bottom of the left panel to reveal:
   - **Exchange** (Exchange Admin Centre)
   - **Security** (Microsoft Defender)
   - **Azure Active Directory** (Entra ID)

---

## Step 3 — Manage Users via Admin Centre and PowerShell

### Portal (GUI) — Microsoft 365 Admin Centre
1. **Users** → **Active users** → **+ Add a user**
2. Fill in: First name, Last name, Username (jsmith@yourtenant.onmicrosoft.com)
3. Click **Next** → assign a licence (e.g. Microsoft 365 Business Basic)
4. Click **Add** → user receives a welcome email

### PowerShell
```powershell
# List all users
Get-MsolUser -All | Select-Object DisplayName, UserPrincipalName, IsLicensed | Format-Table

# Get a specific user
Get-MsolUser -UserPrincipalName jsmith@yourtenant.onmicrosoft.com

# Create a new user
New-MsolUser `
  -DisplayName "Jane Smith" `
  -FirstName "Jane" `
  -LastName "Smith" `
  -UserPrincipalName jsmith@yourtenant.onmicrosoft.com `
  -Password "Welcome@2026!!" `
  -ForceChangePassword $true `
  -PasswordNeverExpires $false

# Assign a licence
$licence = New-Object -TypeName Microsoft.Open.AzureAD.Model.AssignedLicense
$licences = New-Object -TypeName Microsoft.Open.AzureAD.Model.AssignedLicenses
# Use Get-MsolAccountSku to find your SKU name
Get-MsolAccountSku | Select-Object SkuPartNumber
```

---

## Step 4 — Manage Mailboxes in Exchange Admin Centre

### Portal (GUI)
1. In admin.microsoft.com → **Exchange** (opens Exchange Admin Centre in a new tab)
2. **Recipients** → **Mailboxes** — see all user mailboxes
3. Click a mailbox → see its settings: **Email addresses**, **Mailbox features**, **Member of**
4. Click **Quotas** — see inbox storage limits (NHS typically sets these to control costs)

**Common tasks from the Exchange Admin Centre GUI:**
- Add an email alias: click a mailbox → **Email addresses** → **+ Add**
- Set an out-of-office: click a mailbox → **Mailbox features** → **Out of office**
- View sent items from another person's mailbox: set up Send As or Full Access permissions

### PowerShell
```powershell
# List all mailboxes
Get-Mailbox | Select-Object DisplayName, PrimarySmtpAddress, RecipientTypeDetails | Format-Table

# Get mailbox details
Get-Mailbox -Identity jsmith@yourtenant.onmicrosoft.com | Format-List

# Get mailbox statistics (size, item count)
Get-MailboxStatistics -Identity jsmith@yourtenant.onmicrosoft.com | 
  Select-Object DisplayName, TotalItemSize, ItemCount, LastLogonTime

# Add an email alias
Set-Mailbox -Identity jsmith@yourtenant.onmicrosoft.com `
  -EmailAddresses @{Add = "j.smith@yourtenant.onmicrosoft.com"}

# View mailbox quota
Get-Mailbox jsmith@yourtenant.onmicrosoft.com | 
  Select-Object ProhibitSendQuota, ProhibitSendReceiveQuota, IssueWarningQuota
```

---

## Step 5 — Create a Shared Mailbox

Shared mailboxes are used for team inboxes (e.g. it.support@slamdst.nhs.uk).

### Portal (GUI)
1. Exchange Admin Centre → **Recipients** → **Shared mailboxes** → **+ Add a shared mailbox**
2. Display name: `IT Support`
3. Email address: `it.support@yourtenant.onmicrosoft.com`
4. Click **Create**
5. After creation, click the mailbox → **Manage mailbox access** → **+ Add members**
6. Add the users who should access this inbox

### PowerShell
```powershell
# Create shared mailbox
New-Mailbox -Shared `
  -Name "IT Support" `
  -DisplayName "IT Support" `
  -Alias "it.support" `
  -PrimarySmtpAddress "it.support@yourtenant.onmicrosoft.com"

# Grant Full Access (allows user to open the mailbox in Outlook)
Add-MailboxPermission -Identity "it.support@yourtenant.onmicrosoft.com" `
  -User jsmith@yourtenant.onmicrosoft.com `
  -AccessRights FullAccess `
  -InheritanceType All

# Grant Send As (allows sending email as the shared mailbox)
Add-RecipientPermission -Identity "it.support@yourtenant.onmicrosoft.com" `
  -Trustee jsmith@yourtenant.onmicrosoft.com `
  -AccessRights SendAs `
  -Confirm:$false

# Verify permissions
Get-MailboxPermission -Identity "it.support@yourtenant.onmicrosoft.com" | 
  Where-Object {$_.User -notlike "*NT AUTHORITY*"} | 
  Select-Object User, AccessRights
```

**What you just learned:** Full Access lets a user open the mailbox in Outlook and read/reply to emails. Send As lets them send emails that appear to come from the shared mailbox address. In the NHS, clinical teams often share a mailbox for referral letters or test results.

---

## Step 6 — Distribution Groups and Mail-Enabled Security Groups

### Portal (GUI)
1. Exchange Admin Centre → **Recipients** → **Groups** → **+ Add a group**
2. Choose **Distribution** group type → **Next**
3. Name: `Infrastructure Team` | Email: `infrastructure.team@yourtenant.onmicrosoft.com`
4. Add members → **Create**

### PowerShell
```powershell
# Create a distribution group
New-DistributionGroup `
  -Name "Infrastructure Team" `
  -DisplayName "Infrastructure Team" `
  -Alias "infrastructure.team" `
  -PrimarySmtpAddress "infrastructure.team@yourtenant.onmicrosoft.com" `
  -MemberJoinRestriction Closed

# Add members
Add-DistributionGroupMember -Identity "infrastructure.team@yourtenant.onmicrosoft.com" `
  -Member jsmith@yourtenant.onmicrosoft.com

# List members
Get-DistributionGroupMember -Identity "infrastructure.team@yourtenant.onmicrosoft.com" | 
  Select-Object DisplayName, PrimarySmtpAddress

# List all groups
Get-DistributionGroup | Select-Object DisplayName, PrimarySmtpAddress, GroupType
```

---

## Step 7 — Transport Rules (Email Policies)

Transport rules intercept email in transit to enforce policies — add disclaimers, block content, or route specific mail.

### Portal (GUI)
1. Exchange Admin Centre → **Mail flow** → **Rules** → **+ Add a rule**
2. Name: `Add NHS Disclaimer`
3. Apply when: **The recipient is located** → **Outside the organisation**
4. Do the following: **Append a disclaimer** → enter the disclaimer text
5. Click **Save**

### PowerShell
```powershell
# Create a transport rule that adds a disclaimer to all external emails
New-TransportRule `
  -Name "NHS External Email Disclaimer" `
  -SentToScope NotInOrganization `
  -ApplyHtmlDisclaimerText "<p style='font-size:10px;color:grey;'>This email and any attachments are confidential. If you are not the intended recipient, please notify the sender immediately and delete this email. South London and Maudsley NHS Foundation Trust.</p>" `
  -ApplyHtmlDisclaimerFallbackAction Wrap `
  -ApplyHtmlDisclaimerLocation Append

# Create a rule to block emails with specific keywords (data loss prevention)
New-TransportRule `
  -Name "Block External NHS Number Sharing" `
  -SentToScope NotInOrganization `
  -SubjectOrBodyMatchesPatterns "NHS.*\d{10}" `
  -RejectMessageReasonText "Sending NHS numbers externally is not permitted without encryption."

# List all transport rules
Get-TransportRule | Select-Object Name, State, Priority | Sort-Object Priority
```

---

## Step 8 — Mail Trace (Investigate Delivery Issues)

### Portal (GUI)
1. Exchange Admin Centre → **Mail flow** → **Message trace**
2. Set: Sender, Recipient, Date range
3. Click **Search**
4. Click a result to see: Status (Delivered/Failed), hops taken, any errors

### PowerShell
```powershell
# Trace messages from a sender in the last 24 hours
Get-MessageTrace `
  -SenderAddress jsmith@yourtenant.onmicrosoft.com `
  -StartDate (Get-Date).AddDays(-1) `
  -EndDate (Get-Date) | 
  Select-Object Received, SenderAddress, RecipientAddress, Subject, Status, Size

# Get detail on a specific message (use MessageTraceId from above)
Get-MessageTraceDetail -MessageTraceId "<trace-id>" -RecipientAddress "recipient@domain.com"
```

**What you just learned:** Mail trace is the first thing you run when a user says "I didn't receive that email." The Status column shows Delivered, Failed, Quarantined, or Pending. If Failed, the trace detail shows the exact error — common causes: recipient doesn't exist, transport rule blocked it, spam filter quarantined it.

---

## Step 9 — Entra Connect Overview

Entra Connect (formerly Azure AD Connect) runs on a Windows server on-premises and synchronises AD users to the cloud.

**What it syncs:**
- User accounts (samAccountName, email, phone, department)
- Password hashes (so users can log into M365 with their AD password)
- Group memberships

**The sync cycle:** By default, Entra Connect syncs every 30 minutes. You can force a sync immediately:

```powershell
# Run on the server where Entra Connect is installed
Import-Module ADSync

# Check last sync time
Get-ADSyncScheduler | Select-Object NextSyncCyclePolicyType, NextSyncCycleStartTimeInUTC

# Force an immediate delta sync (only syncs changes since last sync)
Start-ADSyncSyncCycle -PolicyType Delta

# Force a full sync (syncs everything)
Start-ADSyncSyncCycle -PolicyType Initial
```

**Common sync errors:**
| Error | Cause | Fix |
|---|---|---|
| Object already exists | Same email address in both AD and Entra ID | Use Entra Connect's merge/soft match feature |
| Attribute is not unique | Duplicate UPN or email in AD | Fix the duplicate in AD Users and Computers |
| Export errors | Licence not available | Buy more licences in M365 Admin Centre |

---

## Step 10 — Disconnect from Exchange Online

```powershell
Disconnect-ExchangeOnline -Confirm:$false
```

---

## Key Concepts Table

| Concept | What It Means |
|---|---|
| Hybrid Exchange | On-premises Exchange AND Exchange Online running simultaneously |
| Entra Connect | Syncs on-prem AD users to Entra ID/M365 every 30 minutes |
| Shared mailbox | Team inbox — multiple users can read and send as this address |
| Full Access | Permission to open and read a mailbox |
| Send As | Permission to send email from another address |
| Transport rule | Policy applied to all mail in transit |
| Message trace | Tool to investigate why an email was not delivered |
| Distribution group | Email list — one address forwards to all members |

---

## Common Interview Questions

1. **"A user says they did not receive an important email. How do you investigate?"** — Run a message trace in Exchange Admin Centre (or Get-MessageTrace) with the sender, recipient, and date range. Check the Status: Delivered means it arrived (check spam folder, Outlook rules). Failed/Rejected: read the error — transport rule, spam filter, or invalid address. Quarantined: check the quarantine in the Security portal.

2. **"What is the difference between a shared mailbox and a distribution group?"** — A shared mailbox has its own email address, inbox, sent items, and calendar. Multiple users can read and reply from it, and replies show in the shared sent items. A distribution group just forwards incoming email to all members — it has no inbox of its own. Use a shared mailbox when the team needs to track conversations; use a distribution group for one-way announcements.

3. **"What does Entra Connect do and what happens if it stops running?"** — Entra Connect syncs on-premises AD accounts to Entra ID so users can log into M365 with their AD credentials. If it stops, new users created in AD won't appear in M365, password changes won't sync, and group memberships won't update. M365 continues to work for existing users until their password hash expires.

---

## Next Steps
Move on to **Project 15 — Azure SQL**.
