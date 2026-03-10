# Azure SQL Database Deployment via Azure CLI

![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![SQL](https://img.shields.io/badge/Azure_SQL-CC2927?style=for-the-badge&logo=microsoft-sql-server&logoColor=white)
![CLI](https://img.shields.io/badge/Azure_CLI-0078D4?style=for-the-badge&logo=windows-terminal&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=for-the-badge)

> **Enterprise-grade, repeatable deployment of a publicly accessible Azure SQL Database instance using Azure CLI. Covers server provisioning, firewall configuration, database creation, and full verification.**

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Known Issues and Resolutions](#known-issues-and-resolutions)
- [Deployment Guide](#deployment-guide)
  - [Phase 0 -- Authentication](#phase-0----authentication)
  - [Phase 1 -- SQL Server Provisioning](#phase-1----sql-server-provisioning)
  - [Phase 2 -- Public Network Access](#phase-2----public-network-access)
  - [Phase 3 -- Database Creation](#phase-3----database-creation)
  - [Phase 4 -- Verification](#phase-4----verification)
- [Configuration Reference](#configuration-reference)
- [Screenshots](#screenshots)
- [Security Considerations](#security-considerations)

---

## Overview

This runbook documents the end-to-end automated deployment of an **Azure SQL Database** instance configured for public accessibility, Basic compute tier, and locally-redundant backup storage. The deployment is executed entirely through the **Azure CLI** and is designed to be reproducible, auditable, and suitable for integration into CI/CD pipelines.

**Use Case:** Infrastructure migration to Azure using incremental provisioning steps, suitable for teams onboarding cloud-native database workloads.

---

## Architecture

```
Azure Subscription (Azure Free Labs)
   └── Resource Group: kml_rg_main-36c801c9838649ab
         └── Region: Central US
               └── Azure SQL Server: datacenter-server-17900
                     ├── Firewall Rule: AllowAllPublicIPs (0.0.0.0 -- 255.255.255.255)
                     └── Azure SQL Database: datacenter-sqldb
                           ├── Edition: Basic
                           ├── Max Size: 2 GB
                           ├── Backup Redundancy: Locally Redundant
                           └── Status: Online
```

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Azure CLI | >= 2.50.0 | `az --version` to verify |
| Active Azure Subscription | N/A | With SQL resource provider registered |
| Contributor Role | N/A | On the target resource group |
| Bash or Zsh Shell | Any | PowerShell requires syntax adjustments |

**Register the SQL resource provider if not already active:**

```bash
az provider register --namespace Microsoft.Sql
az provider show --namespace Microsoft.Sql --query "registrationState"
```

---

## Known Issues and Resolutions

This section documents all encountered errors during deployment and their verified resolutions.

---

### Issue 1 -- PasswordNotComplex

**Error Code:** `PasswordNotComplex`

**Full Error Message:**
```
(PasswordNotComplex) Password validation failed. The password does not meet
policy requirements because it is not complex enough.
```

**Failed Command:**
```bash
az sql server create \
  --name "datacenter-server-17900" \
  --resource-group $(az group list --query "[0].name" -o tsv) \
  --location "centralus" \
  --admin-user "datacenter-admin" \
  --admin-password "Admin@1234567!"
```

**Root Cause:**

Azure SQL Server enforces strict password complexity policies. The password `Admin@1234567!` was rejected because it contains a substring (`Admin`) that matches or closely resembles the admin username (`datacenter-admin`). Azure explicitly prohibits passwords that contain the login name or parts of it.

**Resolution:**

Use a password that shares **zero character overlap** with the admin username. The password must independently satisfy all four complexity classes: uppercase letters, lowercase letters, digits, and special characters.

```bash
az sql server create \
  --name "datacenter-server-17900" \
  --resource-group $(az group list --query "[0].name" -o tsv) \
  --location "centralus" \
  --admin-user "datacenter-admin" \
  --admin-password "Str0ng#Pass$2026!"
```

**Password Policy Checklist:**

| Rule | Requirement | "Str0ng#Pass$2026!" |
|---|---|---|
| Minimum length | 8 characters | 18 chars |
| Uppercase letter | At least 1 | S, P |
| Lowercase letter | At least 1 | tr, ng, ass |
| Digit | At least 1 | 0, 2026 |
| Special character | At least 1 | #, $, ! |
| Must NOT contain username | No substring match | No overlap with "datacenter-admin" |

**Screenshot -- Password Failure:**

> ***[SCREENSHOT: Terminal output showing the red PasswordNotComplex error message after the first server creation attempt]***

**Screenshot -- Successful Server Creation:**

> ***[SCREENSHOT: Terminal output showing the full JSON response with "state": "Ready" and "publicNetworkAccess": "Enabled" after the corrected command]***

---

## Deployment Guide

### Phase 0 -- Authentication

**Step 1 -- Sign in to Azure**

```bash
az login \
  --username "your-username@azurefreekmlprod.onmicrosoft.com" \
  --password "YourPassword"
```

**Step 2 -- Verify active subscription**

```bash
az account show
```

Expected fields to confirm:

```json
{
  "name": "Azure Free Labs",
  "state": "Enabled",
  "isDefault": true
}
```

**Step 3 -- Set default location**

```bash
az configure --defaults location=centralus
```

> ***[SCREENSHOT: Terminal output of `az account show` confirming the correct subscription is active]***
<img width="1032" height="564" alt="image" src="https://github.com/user-attachments/assets/559e9ad1-d3a4-485d-a287-aa1ec45349e0" />

---

### Phase 1 -- SQL Server Provisioning

**Step 4 -- Create the Azure SQL Server**

```bash
az sql server create \
  --name "datacenter-server-17900" \
  --resource-group "kml_rg_main-36c801c9838649ab" \
  --location "centralus" \
  --admin-user "datacenter-admin" \
  --admin-password "Str0ng#Pass$2026!"
```

**Expected Response (key fields):**

```json
{
  "administratorLogin": "datacenter-admin",
  "fullyQualifiedDomainName": "datacenter-server-17900.database.windows.net",
  "location": "centralus",
  "name": "datacenter-server-17900",
  "publicNetworkAccess": "Enabled",
  "state": "Ready",
  "version": "12.0"
}
```

**Step 5 -- Verify server provisioning**

```bash
az sql server show \
  --name "datacenter-server-17900" \
  --resource-group "kml_rg_main-36c801c9838649ab" \
  --query "{Name:name, State:state, PublicAccess:publicNetworkAccess, FQDN:fullyQualifiedDomainName}" \
  --output table
```

> ***[SCREENSHOT: Full JSON output of the successful server creation showing state "Ready"]***

---

### Phase 2 -- Public Network Access

**Step 6 -- Create firewall rule for public accessibility**

```bash
az sql server firewall-rule create \
  --server "datacenter-server-17900" \
  --resource-group "kml_rg_main-36c801c9838649ab" \
  --name "AllowAllPublicIPs" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 255.255.255.255
```

**Expected Response:**

```json
{
  "endIpAddress": "255.255.255.255",
  "name": "AllowAllPublicIPs",
  "startIpAddress": "0.0.0.0",
  "type": "Microsoft.Sql/servers/firewallRules"
}
```

> ***[SCREENSHOT: Terminal output confirming the firewall rule was created with the correct IP range]***

---

### Phase 3 -- Database Creation

**Step 7 -- Create the Azure SQL Database**

```bash
az sql db create \
  --name "datacenter-sqldb" \
  --server "datacenter-server-17900" \
  --resource-group "kml_rg_main-36c801c9838649ab" \
  --edition "Basic" \
  --service-objective "Basic" \
  --max-size "2GB" \
  --backup-storage-redundancy "Local"
```

**Parameter Reference:**

| Parameter | Value | Rationale |
|---|---|---|
| `--edition` | `Basic` | Maps to "Basic (For less demanding workloads)" in the portal |
| `--service-objective` | `Basic` | Must match edition for the Basic tier |
| `--max-size` | `2GB` | Task requirement; valid Basic tier option |
| `--backup-storage-redundancy` | `Local` | Locally-redundant backup storage as specified |

**Expected Response (key fields):**

```json
{
  "currentBackupStorageRedundancy": "Local",
  "currentServiceObjectiveName": "Basic",
  "edition": "Basic",
  "maxSizeBytes": 2147483648,
  "name": "datacenter-sqldb",
  "requestedBackupStorageRedundancy": "Local",
  "status": "Online"
}
```

> ***[SCREENSHOT: Full JSON output from the `az sql db create` command showing "status": "Online"]***

---

### Phase 4 -- Verification

**Step 8 -- Run final verification query**

```bash
az sql db show \
  --name "datacenter-sqldb" \
  --server "datacenter-server-17900" \
  --resource-group "kml_rg_main-36c801c9838649ab" \
  --query "{Name:name, Status:status, Edition:edition, MaxSizeBytes:maxSizeBytes, BackupRedundancy:requestedBackupStorageRedundancy, Location:location}" \
  --output table
```

**Expected Output:**

```
Name              Status    Edition    MaxSizeBytes    BackupRedundancy    Location
----------------  --------  ---------  --------------  ------------------  ----------
datacenter-sqldb  Online    Basic      2147483648      Local               centralus
```

**Verification Checklist:**

| Requirement | Expected | Actual | Status |
|---|---|---|---|
| Database Name | `datacenter-sqldb` | `datacenter-sqldb` | PASS |
| Server Name | `datacenter-server-17900` | `datacenter-server-17900` | PASS |
| Location | `centralus` | `centralus` | PASS |
| Edition | `Basic` | `Basic` | PASS |
| Max Size | 2 GB (2147483648 bytes) | `2147483648` | PASS |
| Backup Redundancy | `Local` | `Local` | PASS |
| Public Access | Enabled | Firewall: `0.0.0.0 -- 255.255.255.255` | PASS |
| Admin Login | `datacenter-admin` | `datacenter-admin` | PASS |
| Database Status | Online | `Online` | PASS |

> ***[SCREENSHOT: Terminal output of the final `az sql db show` table command showing all fields correctly populated]***
<img width="1036" height="460" alt="image" src="https://github.com/user-attachments/assets/9df63aa4-ec86-4605-97a7-3196b82bd475" />

---

## Configuration Reference

### Full Deployment Parameters

```yaml
server:
  name: datacenter-server-17900
  location: centralus
  resource_group: kml_rg_main-36c801c9838649ab
  admin_user: datacenter-admin
  tls_version: "1.2"
  public_network_access: Enabled
  version: "12.0"

firewall:
  rule_name: AllowAllPublicIPs
  start_ip: 0.0.0.0
  end_ip: 255.255.255.255

database:
  name: datacenter-sqldb
  edition: Basic
  service_objective: Basic
  max_size_bytes: 2147483648
  backup_redundancy: Local
  collation: SQL_Latin1_General_CP1_CI_AS
  zone_redundant: false
```

---

## Screenshots

| # | Description | Location |
|---|---|---|
| 1 | PasswordNotComplex error on first attempt | [See Issue 1 above](#issue-1----passwordnotcomplex) |
| 2 | Successful server creation JSON output | [See Phase 1](#phase-1----sql-server-provisioning) |
| 3 | Firewall rule creation confirmation | [See Phase 2](#phase-2----public-network-access) |
| 4 | Database creation full JSON response | [See Phase 3](#phase-3----database-creation) |
| 5 | Final verification table output | [See Phase 4](#phase-4----verification) |

---

## Security Considerations

> **Note:** The firewall rule `0.0.0.0 -- 255.255.255.255` grants public access from any IP. This configuration was intentionally applied per task requirements. For production workloads, apply the principle of least privilege.

**Production Hardening Recommendations:**

* Restrict firewall rules to known IP ranges or use Private Endpoints
* Rotate the admin password immediately after initial provisioning and store in Azure Key Vault
* Enable Microsoft Defender for SQL for threat detection
* Enable Auditing to a Storage Account or Log Analytics Workspace
* Consider switching backup redundancy to `Geo` or `Zone` for DR requirements
* Enforce Azure AD authentication instead of SQL authentication where possible

---




<img width="1041" height="412" alt="image" src="https://github.com/user-attachments/assets/83b83d0f-f5ea-445c-b772-46021a6ee85a" />
<img width="1035" height="691" alt="image" src="https://github.com/user-attachments/assets/01dabebd-1f15-4e02-95ea-4d0e912024d6" />
<img width="1038" height="686" alt="image" src="https://github.com/user-attachments/assets/1135afb5-de29-4fad-b114-857fbb49ffda" />
<img width="1043" height="857" alt="image" src="https://github.com/user-attachments/assets/9dbd4cda-0d83-43aa-a9f8-ca70eae13f5b" />
<img width="1035" height="860" alt="image" src="https://github.com/user-attachments/assets/ba732fb3-5aee-4773-b64d-66e63f5a9703" />
<img width="1040" height="864" alt="image" src="https://github.com/user-attachments/assets/fe281919-6b34-4cc0-bba6-9361daebe1a9" />

