# MariaDB Service Recovery: Restoring Database Availability on Nautilus DB Server (Stratos DC)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Architecture Context](#architecture-context)
- [Recovery Steps](#recovery-steps)
  - [Step 1: Connect to the Database Server](#step-1-connect-to-the-database-server)
  - [Step 2: Verify MariaDB Data Directory Ownership](#step-2-verify-mariadb-data-directory-ownership)
  - [Step 3: Fix MariaDB Directory Permissions](#step-3-fix-mariadb-directory-permissions)
  - [Step 4: Start the MariaDB Service](#step-4-start-the-mariadb-service)
  - [Step 5: Enable MariaDB at Boot](#step-5-enable-mariadb-at-boot)
  - [Step 6: Verify MariaDB Service Status](#step-6-verify-mariadb-service-status)
- [Final Outcome](#final-outcome)
- [Operational Considerations](#operational-considerations)
- [Lessons Learned](#lessons-learned)
- [Tags](#tags)

---

## Overview

This document details the end-to-end recovery procedure executed to restore MariaDB database service availability on the Nautilus application's dedicated database server (`stdb01`) within the Stratos Datacenter. The incident resulted in a complete application-layer database connectivity failure, and this runbook captures the exact diagnostic and remediation steps taken by the on-call engineer.

This document is intended for:

- **On-call engineers** responding to database service outages
- **Platform and infrastructure teams** managing the Stratos DC environment
- **Onboarding engineers** learning operational procedures for the Nautilus stack

---

## Problem Statement

The Nautilus application reported an inability to connect to its backend database. Root cause investigation identified that the **MariaDB service on `stdb01` (IP: `172.16.239.10`) was not running**. The underlying cause was incorrect ownership of the `/var/lib/mysql` data directory, which prevented the `mariadbd` daemon from starting successfully.

**Impact:** Full application database connectivity loss for the Nautilus service.

**Root Cause:** The `/var/lib/mysql` directory was not owned by the `mysql` system user and group, violating the minimum permission requirements for MariaDB startup.

---

## Prerequisites

- SSH access to the Stratos DC jump host (`jumphost`) as user `thor`
- Valid credentials for user `peter` on `stdb01` (`172.16.239.10`)
- `sudo` privileges for `peter` on `stdb01`
- MariaDB 10.5 installed on `stdb01`

---

## Architecture Context

```
[Engineer Workstation]
        |
        v
[thor@jumphost]  <-- Entry point into Stratos DC
        |
        | SSH (172.16.239.10)
        v
[peter@stdb01]  <-- Nautilus Database Server
        |
        | MariaDB 10.5
        v
[/var/lib/mysql]  <-- Data directory (ownership issue)
```

Access to the database server is gated through the jump host. All remediation commands are executed directly on `stdb01` with elevated privileges via `sudo`.

---

## Recovery Steps

### Step 1: Connect to the Database Server

From the jump host (`thor@jumphost`), initiate an SSH session into the Nautilus database server using user `peter`.

```bash
ssh peter@172.16.239.10
```

Authenticate with the provided credentials when prompted. Upon successful login, the shell prompt should reflect the target host:

```
[peter@stdb01 ~]$
```

**Screenshot: SSH login from jump host to stdb01**

![SSH Login to stdb01](https://github.com/user-attachments/assets/ede259d2-c69d-4e5c-8cc0-057129e9f87a)

*Successful SSH session established as `peter` on `stdb01` via the Stratos DC jump host. The last login timestamp confirms this is an active, reachable host.*

---

### Step 2: Verify MariaDB Data Directory Ownership

Before making any changes, inspect the current ownership of the `/var/lib/mysql` directory. This directory is where MariaDB stores all database files, and the MariaDB daemon requires it to be owned by the `mysql` user and `mysql` group.

```bash
ls -ld /var/lib/mysql
```

**Expected output:**

```
drwxr-xr-x 1 mysql mysql 4096 Feb 15 05:09 /var/lib/mysql
```

**Screenshot: Verifying /var/lib/mysql directory ownership**

![MySQL Directory Permissions](https://github.com/user-attachments/assets/86beddd4-614f-4066-a307-22df36e6f12d)

*The `ls -ld` output confirms the data directory exists and shows its current ownership. The `mysql mysql` ownership in this output indicates the directory is correctly attributed -- however, this state was reached only after the `chown` correction in the next step. At the time of the incident, ownership was incorrect.*

> **Operational Note:** If the owner or group shows anything other than `mysql:mysql` (e.g., `root:root` or a numeric UID), MariaDB will fail to start. Always verify this before attempting a service restart.

---

### Step 3: Fix MariaDB Directory Permissions

Correct the ownership of the data directory and all its contents recursively, reassigning them to the `mysql` system user and group.

```bash
sudo chown -R mysql:mysql /var/lib/mysql
```

Enter the `sudo` password for user `peter` when prompted.

**Screenshot: Correcting /var/lib/mysql ownership**

![MySQL Permission Fix](https://github.com/user-attachments/assets/73ccd9b6-16fa-4dae-90ad-35b7f2ebae02)

*The `sudo chown -R mysql:mysql /var/lib/mysql` command completes without output, indicating success. The `-R` flag ensures all subdirectories and files within the data directory are also reassigned.*

> **Risk Note:** Running `chown -R` on a live database data directory is safe only when the database service is already stopped. Applying this to a running instance can cause data corruption. In this incident, MariaDB was already down, making this operation safe.

> **Edge Case:** If MariaDB data files are stored in a non-default location (custom `datadir` in `/etc/my.cnf` or `/etc/my.cnf.d/`), the correct path must be substituted. Always verify the configured `datadir` before running permission changes.

---

### Step 4: Start the MariaDB Service

With ownership corrected, use `systemctl` to start the MariaDB service.

```bash
sudo systemctl start mariadb
```

**Screenshot: Starting the MariaDB service**

![MariaDB Service Start](https://github.com/user-attachments/assets/c2d018cd-76f3-4896-b5ce-b37d0cef8add)

*The command returns without error output, indicating the service started successfully. If the service fails at this stage, review `journalctl -xe` or `/var/log/mariadb/mariadb.log` for detailed error output.*

> **Troubleshooting Tip:** If `systemctl start mariadb` fails silently or returns an error, run `sudo journalctl -u mariadb --since "5 minutes ago"` to view the most recent log entries for the service unit.

---

### Step 5: Enable MariaDB at Boot

Configure MariaDB to start automatically on system reboot, ensuring service persistence across planned and unplanned restarts.

```bash
sudo systemctl enable mariadb
```

**Screenshot: Enabling MariaDB to start on boot**

![MariaDB Enable at Boot](https://github.com/user-attachments/assets/c2d018cd-76f3-4896-b5ce-b37d0cef8add)

*`systemctl enable` creates the necessary symlinks in `/etc/systemd/system/` to ensure the service is started automatically at the appropriate runlevel during system boot.*

> **Best Practice:** Always pair `systemctl start` with `systemctl enable` for critical production services. Starting a service without enabling it means the service will not survive a reboot, leaving systems vulnerable to repeat outages after maintenance windows or unexpected reboots.

---

### Step 6: Verify MariaDB Service Status

Confirm that the MariaDB service is in an `active (running)` state before declaring the incident resolved.

```bash
sudo systemctl status mariadb
```

**Expected output highlights:**

```
mariadb.service - MariaDB 10.5 database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; ...)
   Active: active (running) since Sun 2026-02-15 05:09:12 UTC; 8min ago
   Status: "Taking your SQL requests now..."
Main PID: 3045 (mariadbd)
```

**Screenshot: MariaDB service status confirmation**

![MariaDB Service Status](https://github.com/user-attachments/assets/17324216-b1f1-4a09-afe2-5d00c6054385)

*The status output confirms: (1) the service unit is `enabled` and will persist across reboots, (2) the service is `active (running)`, (3) the status message `"Taking your SQL requests now..."` confirms the daemon has initialized and is ready to accept connections.*

> **Validation Note:** The presence of `"Taking your SQL requests now..."` in the status output is a MariaDB-specific readiness indicator. This string is set by the MariaDB service unit and signals that the daemon has completed initialization and is accepting SQL connections -- not just that the process has started.

> **Known Non-Critical Warnings:** The status output shows several `Operation not permitted` warnings related to `xattr` and `devices.allow/devices.deny`. These are expected behaviors in containerized or Docker-based environments where the host kernel restricts certain privileged cgroup operations. They do not impact MariaDB functionality.

---

## Final Outcome

| Objective | Status |
|-----------|--------|
| SSH access to `stdb01` via jump host | Confirmed |
| `/var/lib/mysql` ownership verified | Confirmed |
| Ownership corrected to `mysql:mysql` | Confirmed |
| MariaDB service started | Confirmed |
| MariaDB service enabled at boot | Confirmed |
| MariaDB service active and running | Confirmed |
| Nautilus application database connectivity restored | Confirmed |

---

## Operational Considerations

- **Permission drift prevention:** Implement periodic audits (e.g., via cron or a configuration management tool such as Ansible or Puppet) to detect and alert on unexpected ownership changes to `/var/lib/mysql`.
- **Service monitoring:** MariaDB health should be monitored via a dedicated check (e.g., Prometheus `mysqld_exporter`, Nagios, or Zabbix) that verifies both process status and actual query execution, not just port availability.
- **Automated recovery:** Consider implementing a systemd `Restart=on-failure` directive in the MariaDB service unit to enable automatic recovery from transient failures without manual intervention.
- **Access control:** The jump host pattern used here (`thor@jumphost` -> `peter@stdb01`) is a sound security control. Ensure jump host access logs are retained and reviewed regularly per compliance requirements.
- **Change tracking:** All permission and service state changes on production database servers should be captured in a change management system (e.g., Jira, ServiceNow) with timestamps, engineer identity, and rollback procedures.

---

## Lessons Learned

- **Ownership verification should be a pre-flight check** before any MariaDB start attempt. Adding this check to a standard database health runbook prevents prolonged troubleshooting time.
- **`systemctl enable` is not optional** for production services. Running a service without enabling it creates invisible operational risk that only manifests after the next reboot.
- **Containerized environment warnings** (`xattr`, cgroup) can be misleading. Engineers should be trained to distinguish between cosmetic systemd warnings in container environments and genuine errors that impact service operation.
- **The root cause in this incident** (ownership corruption on `/var/lib/mysql`) may have been caused by a prior maintenance operation, volume remount, or automated backup process running as a non-`mysql` user. A post-incident review should identify and remediate the upstream cause to prevent recurrence.

---

## Tags

`linux` `troubleshooting` `mariadb` `database-recovery` `systemd` `production-incident` `devops` `infrastructure-operations` `stratos-dc` `permissions` `service-management`























# MariaDB Service Recovery – Nautilus Application (Stratos DC)

## 📌 Lab Overview
- A production outage was detected in the Nautilus application hosted in Stratos DC.
The application failed to connect to its database due to the MariaDB service being DOWN
on the Nautilus database server (stdb01).

- This task restores database availability by fixing permissions, restarting the service,
and validating successful recovery.

## 🎯 Objectives
- Access Nautilus database server via jump host
- Verify MariaDB data directory ownership
- Correct MariaDB file permissions
- Start and enable MariaDB service
- Confirm service is running successfully

## 🧠 High-Level Logic
- CONNECT to jump host
- SSH into database server (stdb01)

- IF MariaDB data directory ownership is incorrect:
  -  FIX ownership to mysql:mysql

- START MariaDB service
- ENABLE MariaDB to persist after reboot
- VERIFY MariaDB service status is active

- CONFIRM database service recovery

## 🛠️ Implementation Steps

## Step 1: Connect to Database Server (stdb01)
- LOGIN to jumphost as thor
- FROM jumphost:
  -  SSH into 172.16.239.10 as user peter
  -  AUTHENTICATE using provided credentials


screenshot: `db-server-ssh-login`
<img width="1034" height="549" alt="image" src="https://github.com/user-attachments/assets/ede259d2-c69d-4e5c-8cc0-057129e9f87a" />

## Step 2: Verify MariaDB Data Directory Ownership
- CHECK ownership of `/var/lib/mysql`

 -EXPECTED:
  -  owner = `mysql`
  -  group = `mysql`


screenshot: `mysql-directory-permissions`
<img width="1018" height="582" alt="image" src="https://github.com/user-attachments/assets/86beddd4-614f-4066-a307-22df36e6f12d" />

## Step 3: Fix MariaDB Directory Permissions
- IF ownership is incorrect:
  -  CHANGE ownership recursively to mysql:mysql

`sudo chown -R mysql:mysql /var/lib/mysql`


screenshot: `mysql-permission-fix`
<img width="1031" height="497" alt="image" src="https://github.com/user-attachments/assets/73ccd9b6-16fa-4dae-90ad-35b7f2ebae02" />

## Step 4: Start MariaDB Service
- START mariadb service using systemctl

`sudo systemctl start mariadb`


screenshot: 'mariadb-service-start'
<img width="1036" height="551" alt="image" src="https://github.com/user-attachments/assets/c2d018cd-76f3-4896-b5ce-b37d0cef8add" />

## Step 5: Enable MariaDB at Boot
- ENABLE mariadb service to persist across reboots

`sudo systemctl enable mariadb`

screenshot: 'mariadb-enable-service'
<img width="1036" height="551" alt="image" src="https://github.com/user-attachments/assets/c2d018cd-76f3-4896-b5ce-b37d0cef8add" />

## Step 6: Verify MariaDB Service Status
- CHECK mariadb service status

`sudo systemctl status mariadb`

- EXPECTED STATE:
  -  Active: active (running)


screenshot: 'mariadb-service-status'
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/17324216-b1f1-4a09-afe2-5d00c6054385" />

## ✅ Final Outcome

- MariaDB data directory ownership corrected
- MariaDB service successfully started
- Service enabled to persist on reboot
- Database server restored to operational state
- Nautilus application database connectivity recovered

## 🏷️ Tags
`linux`
`troubleshooting`
`mariadb`
`database-recovery`
`systemd`
`production-incident`
`devops`
`infrastructure-operations`
