# Azure Cross-VM PHP to MySQL Database Integration

[![Azure](https://img.shields.io/badge/Cloud-Microsoft%20Azure-0078D4?logo=microsoftazure)](https://azure.microsoft.com)
[![MySQL](https://img.shields.io/badge/Database-MySQL%205.7-4479A1?logo=mysql)](https://www.mysql.com)
[![PHP](https://img.shields.io/badge/Runtime-PHP%208.x-777BB4?logo=php)](https://www.php.net)
[![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%2016.04%20%7C%2022.04-E95420?logo=ubuntu)](https://ubuntu.com)
[![Status](https://img.shields.io/badge/Status-Complete-success)](https://github.com)

---

## Table of Contents

* [Problem Statement](#problem-statement)
* [Architecture Overview](#architecture-overview)
* [Prerequisites](#prerequisites)
* [Phase 1 -- Create and Configure the MySQL VM](#phase-1----create-and-configure-the-mysql-vm)
* [Phase 2 -- Setup the MySQL Database](#phase-2----setup-the-mysql-database)
* [Phase 3 -- Configure the PHP Application VM](#phase-3----configure-the-php-application-vm)
* [Phase 4 -- Validation](#phase-4----validation)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [Troubleshooting Reference](#troubleshooting-reference)

---

## Problem Statement

The **Nautilus DevOps team** required integration of a PHP application hosted on an Azure VM (`devops-php-vm`, East US) with a MySQL database hosted on a separate Azure VM (`devops-mysql-vm`, Central US). The objective was to validate end-to-end cloud database connectivity between two geographically separated virtual machines using Azure's public networking infrastructure.

**Goal:** The PHP application at `/var/www/html/db_test.php` must return `Connected successfully` when accessed via a browser, confirming live connectivity to the remote MySQL instance.

---

## Architecture Overview

```
+---------------------------+          Port 3306/TCP          +-----------------------------+
|    devops-php-vm          | ==============================> |    devops-mysql-vm          |
|    Region: East US        |       Public Internet           |    Region: Central US       |
|    OS: Ubuntu 22.04       |                                 |    OS: Ubuntu 16.04         |
|    IP: 172.173.241.148    |                                 |    IP: 52.146.18.163        |
|    Apache2 + PHP          |                                 |    MySQL 5.7 (Jetware)      |
|    /var/www/html/         |                                 |    devops_db                |
|    db_test.php            |                                 |    devops_user              |
+---------------------------+                                 +-----------------------------+
          |                                                             |
          |                                                             |
   NSG: Port 22 (SSH)                                    NSG: Port 22 (SSH)
   NSG: Port 80 (HTTP)                                   NSG: Port 3306 (MySQL) <-- Added
```

**Resource Summary:**

| Resource | Value |
|---|---|
| MySQL VM Name | `devops-mysql-vm` |
| MySQL VM Region | Central US |
| MySQL Public IP | `52.146.18.163` |
| PHP VM Name | `devops-php-vm` |
| PHP VM Region | East US |
| PHP VM Public IP | `172.173.241.148` |
| MySQL Image | MySQL 5.7 on Ubuntu (Jetware) |
| MySQL Database | `devops_db` |
| MySQL App User | `devops_user` |
| Admin Username | `devops_admin` |

---

## Prerequisites

* An active **Microsoft Azure** subscription with permission to create VMs and modify NSG rules
* A terminal with SSH client (Linux/macOS native, or Windows with Git Bash / WSL)
* Access to **Azure Portal** at [portal.azure.com](https://portal.azure.com)
* An existing `devops-php-vm` running Ubuntu 22.04 with Apache2 and PHP installed in the **East US** region

---

## Phase 1 -- Create and Configure the MySQL VM

### 1.1 Create the VM via Azure Marketplace

1. Sign in to [portal.azure.com](https://portal.azure.com)
2. Click **"Create a resource"** and search for **MySQL** in the Marketplace
3. Select **"MySQL 5.7 on Ubuntu"** (Jetware image) and click **"Create"**
4. Fill in the **Basics** tab:

| Field | Value |
|---|---|
| Virtual machine name | `devops-mysql-vm` |
| Region | `(US) Central US` |
| Availability zone | `Zone 2` |
| Security type | `Standard` |
| Image | `MySQL 5.7 on Ubuntu - x64 Gen1` |
| VM architecture | `x64` |
| Size | `Standard_D2s_v3` |
| Authentication type | `Password` |
| Username | `devops_admin` |
| Password | `Namin@123456` |
| Public inbound ports | `Allow selected ports` |
| Select inbound ports | `SSH (22)` |

> **IMPORTANT:** If you receive the error *"This size is not available in zone 1. Zones 2 are supported"*, change the Availability zone from `Zone 1` to **`Zone 2`** before proceeding.

**Screenshots:**

<img width="1919" height="947" alt="Image" src="https://github.com/user-attachments/assets/c4445cb0-5147-4474-8fd3-0811c89414a7" />

<img width="1919" height="949" alt="Image" src="https://github.com/user-attachments/assets/fb67e00a-0c8a-4e9a-9531-32b90c3ddf19" />

>Azure portal VM creation Basics tab with all fields populated 

### 1.2 Configure the Disks Tab

No changes are required on the Disks tab. Accept all defaults:

| Field | Value |
|---|---|
| OS disk size | Image default (30 GiB) |
| OS disk type | Standard SSD (locally-redundant storage) |
| Delete with VM | Checked |
| Key management | Platform-managed key |

Click **"Next: Networking >"**

**Screenshot:**

<img width="1919" height="951" alt="Image" src="https://github.com/user-attachments/assets/53733321-d3ed-4c3c-a888-ace6ae8fa7af" />

>Disks tab with default settings

### 1.3 Configure the Networking Tab

| Field | Value |
|---|---|
| Virtual network | `devops-php-vmVNET` (auto-selected) |
| Subnet | `devops-php-vmSubnet (10.0.0.0/24)` |
| Public IP | `(new) devops-mysql-vm-ip` |
| NIC network security group | `Basic` |
| Public inbound ports | `Allow selected ports` |
| Select inbound ports | `SSH (22)` only |

> Port 3306 is not available in the basic dropdown and will be added manually to the NSG after deployment.

Click **"Review + create"** then **"Create"** to begin deployment.

**Screenshot:**

<img width="1919" height="948" alt="Image" src="https://github.com/user-attachments/assets/784fc05b-dbaa-4bcb-a5a0-cb2555c2edd0" />

>Networking tab with SSH (22) selected as the only inbound port

### 1.4 Confirm Deployment Completion

Wait for the deployment to complete. You will see the confirmation screen:

> **"Your deployment is complete"**

**Screenshot:**

<img width="1919" height="947" alt="Image" src="https://github.com/user-attachments/assets/d4421fd6-ab33-4dd4-a0a1-6eabfc2fa5b3" />

>Deployment complete screen showing devops-mysql-vm


Click **"Go to resource"** to open the VM overview page.

### 1.5 Record the Public IP Address

On the VM Overview page, locate and copy the **Public IP address**.

> **devops-mysql-vm Public IP: `52.146.18.163`**

Save this value. It will be referenced in all subsequent phases.

### 1.6 Add NSG Inbound Rule for Port 3306

1. In the left menu of `devops-mysql-vm`, click **"Networking"** then **"Network settings"**
2. Click **"+ Create port rule"** then **"Inbound port rule"**
3. Fill in the following values:

| Field | Value |
|---|---|
| Source | `Any` |
| Source port ranges | `*` |
| Destination | `Any` |
| Destination port ranges | `3306` |
| Protocol | `TCP` |
| Action | `Allow` |
| Priority | `310` |
| Name | `MySQL_3306` |

4. Click **"Add"**

**Screenshot:**

<img width="1919" height="955" alt="Image" src="https://github.com/user-attachments/assets/44d1ffdf-dee4-4adc-8631-7e37229931d9" />

>Add inbound security rule panel with port 3306 configured

### 1.7 Verify NSG Rule Was Applied

After the rule is saved, the Inbound port rules table should show:

| Priority | Name | Port | Protocol | Source | Destination | Action |
|---|---|---|---|---|---|---|
| 300 | SSH | 22 | TCP | Any | Any | Allow |
| 310 | MySQL_3306 | 3306 | TCP | Any | Any | Allow |

**Screenshot:**

<img width="1919" height="950" alt="Image" src="https://github.com/user-attachments/assets/72b4f4af-1ef3-4efd-b97e-b5160e903ead" />

>Network settings showing MySQL_3306 rule successfully added at priority 310

---

## Phase 2 -- Setup the MySQL Database

### 2.1 SSH into devops-mysql-vm

Open a terminal and connect using the public IP recorded in Phase 1:

```bash
ssh devops_admin@52.146.18.163
```

When prompted:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
devops_admin@52.146.18.163's password: Namin@123456
```

Expected output:

```
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.11.0-1016-azure x86_64)
devops_admin@devops-mysql-vm:~$
```

**Screenshot:**

<img width="1033" height="825" alt="Image" src="https://github.com/user-attachments/assets/52f24e4e-0b56-4bef-ab1b-87d06883df66" />

>Successful SSH login to devops-mysql-vm showing Ubuntu welcome banner

### 2.2 Access the MySQL Shell

```bash
sudo /jet/enter mysql
```

Expected output:

```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 36
Server version: 5.7.18 MySQL Community Server (GPL)
mysql>
```

**Screenshot:**

<img width="1034" height="864" alt="Image" src="https://github.com/user-attachments/assets/e1bcca8e-5efb-4a04-b2f9-44bdb79530cb" />

>MySQL shell prompt after running sudo /jet/enter mysql

### 2.3 Create the Application Database

```sql
CREATE DATABASE devops_db;
```

Expected output:

```
Query OK, 1 row affected (0.00 sec)
```

### 2.4 Create the Application User

```sql
CREATE USER 'devops_user'@'%' IDENTIFIED BY 'password123';
```

> The `'%'` wildcard allows connections from any host, including the PHP VM's public IP.

Expected output:

```
Query OK, 0 rows affected (0.00 sec)
```

### 2.5 Grant Privileges on the Database

```sql
GRANT ALL PRIVILEGES ON devops_db.* TO 'devops_user'@'%';
```

Expected output:

```
Query OK, 0 rows affected (0.00 sec)
```

### 2.6 Apply Privilege Changes

```sql
FLUSH PRIVILEGES;
```

Expected output:

```
Query OK, 0 rows affected (0.00 sec)
```

### 2.7 Verify Database and Privileges

```sql
SHOW DATABASES;
```

Expected output:

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| devops_db          |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

```sql
SHOW GRANTS FOR 'devops_user'@'%';
```

Expected output:

```
+------------------------------------------------------------+
| Grants for devops_user@%                                   |
+------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'devops_user'@'%'                    |
| GRANT ALL PRIVILEGES ON `devops_db`.* TO 'devops_user'@'%' |
+------------------------------------------------------------+
2 rows in set (0.00 sec)
```

**Screenshot:**

<img width="1028" height="747" alt="Image" src="https://github.com/user-attachments/assets/3053a2b3-463f-4abf-bf21-aa6eec18e83f" />

<img width="1033" height="666" alt="Image" src="https://github.com/user-attachments/assets/5f3cca2f-82c0-4ed9-ac49-fb559d7a4ada" />

>Terminal showing SHOW DATABASES and SHOW GRANTS output confirming devops_db and devops_user

### 2.8 Exit the MySQL Shell

```sql
EXIT;
```

### 2.9 Verify MySQL is Listening on Port 3306

```bash
sudo ss -tlnp | grep 3306
```

Expected output:

```
LISTEN   0   80   :::3306   :::*   users:(("mysqld",pid=2051,fd=20))
```

This confirms MySQL is actively listening for remote connections on all interfaces.

**Screenshot Placeholder:**
```
[ SCREENSHOT: Terminal showing LISTEN on :::3306 confirming MySQL network binding ]
```

### 2.10 Exit the MySQL VM

```bash
exit
```

---

## Phase 3 -- Configure the PHP Application VM

### 3.1 Retrieve the PHP VM Public IP

From **Azure Portal**, navigate to **Virtual Machines** and click **devops-php-vm**.

> **devops-php-vm Public IP: `172.173.241.148`**

**Screenshot Placeholder:**
```
[ SCREENSHOT: devops-php-vm Overview page showing Public IP 172.173.241.148 ]
```

### 3.2 SSH into devops-php-vm

```bash
ssh devops_admin@172.173.241.148
```

> **Note:** If you encounter `Permission denied (publickey)`, the VM was provisioned with SSH key authentication. Use the Azure Portal **Connect** page and click **"Reset password or keys"** to set a password for `devops_admin`, then retry the SSH connection.

**Screenshot Placeholder:**
```
[ SCREENSHOT: Azure Portal Connect page showing Reset password or keys option ]
```

After the password reset, retry:

```bash
ssh devops_admin@172.173.241.148
# Enter password: Namin@123456
```

Expected output:

```
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1044-azure x86_64)
devops_admin@devops-php-vm:~$
```

**Screenshot Placeholder:**
```
[ SCREENSHOT: Successful SSH login to devops-php-vm showing Ubuntu 22.04 welcome banner ]
```

### 3.3 Inspect the Existing db_test.php File

```bash
cat /var/www/html/db_test.php
```

The pre-existing file contains placeholder values that must be replaced:

```php
<?php
    $servername = "<mysql-vm-public-ip>";
    $username = "nautilus_user";
    $password = "password123";
    $dbname = "nautilus_db";

    $conn = new mysqli($servername, $username, $password, $dbname);

    if ($conn->connect_error) {
        die("Connection failed: " . $conn->connect_error);
    }
    echo "Connected successfully";
?>
```

**Screenshot Placeholder:**
```
[ SCREENSHOT: Terminal showing original db_test.php with placeholder values ]
```

### 3.4 Edit db_test.php with Correct Credentials

```bash
sudo nano /var/www/html/db_test.php
```

Delete all existing content (press `CTRL+K` repeatedly to remove each line) and replace with:

```php
<?php
$host     = '52.146.18.163';
$dbname   = 'devops_db';
$username = 'devops_user';
$password = 'password123';

$conn = new mysqli($host, $username, $password, $dbname);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

echo "Connected successfully";
$conn->close();
?>
```

Save and exit:

* Press `CTRL+O` then `Enter` to write the file
* Press `CTRL+X` to exit nano

**Screenshot Placeholder:**
```
[ SCREENSHOT: nano editor showing the updated db_test.php with correct IP, database, and credentials ]
```

### 3.5 Verify the Saved File

```bash
cat /var/www/html/db_test.php
```

Confirm all values match:

| Variable | Expected Value |
|---|---|
| `$host` | `'52.146.18.163'` |
| `$dbname` | `'devops_db'` |
| `$username` | `'devops_user'` |
| `$password` | `'password123'` |

**Screenshot Placeholder:**
```
[ SCREENSHOT: Terminal showing cat output of the updated db_test.php confirming all correct values ]
```

### 3.6 Verify the mysqli PHP Extension

```bash
php -m | grep mysqli
```

Expected output:

```
mysqli
```

If no output is returned, install the extension:

```bash
sudo apt update
sudo apt install php-mysql -y
sudo systemctl restart apache2
```

### 3.7 Verify Apache2 is Running

```bash
sudo systemctl status apache2 --no-pager
```

Expected output:

```
apache2.service - The Apache HTTP Server
   Active: active (running) since Tue 2026-03-17 01:46:49 UTC; 43min ago
```

**Screenshot Placeholder:**
```
[ SCREENSHOT: Terminal showing apache2 service status as active (running) ]
```

---

## Phase 4 -- Validation

### 4.1 Browser Test

Open a web browser and navigate to:

```
http://172.173.241.148/db_test.php
```

**Expected result:**

```
Connected successfully
```

**Screenshot Placeholder:**
```
[ SCREENSHOT: Browser showing "Connected successfully" at http://172.173.241.148/db_test.php ]
```

### 4.2 Command Line Test (Alternative)

```bash
curl http://172.173.241.148/db_test.php
```

Expected output:

```
Connected successfully
```

---

## Best Practices

### Network Security

* **Restrict port 3306 by source IP** in production environments. The configuration in this lab uses `Source: Any` for demonstration purposes. In production, restrict the source to only the PHP VM's private or public IP address to minimize the attack surface.
* **Use private VNet peering** instead of public IPs for VM-to-VM communication within the same Azure environment. This eliminates exposure of the database port to the public internet entirely.
* **Enable Azure Firewall or NSG flow logs** to audit all inbound connections to the MySQL port.

### Authentication and Credentials

* **Never hardcode credentials** in application source files. Use **Azure Key Vault** or environment variables to inject secrets at runtime.
* **Use least-privilege database accounts.** The `devops_user` account in this lab has `GRANT ALL` on `devops_db`. In production, restrict to only the operations the application needs (e.g., `SELECT`, `INSERT`, `UPDATE`).
* **Rotate passwords regularly** and enforce strong password policies on both the OS and MySQL layer.

### VM Configuration

* **Use SSH key authentication** instead of password authentication for all production VMs. Password authentication on Azure VMs is acceptable for lab environments only.
* **Enable Azure Defender for Cloud** on all VMs to receive security posture assessments and threat detection alerts.
* **Set up auto-shutdown** for non-production VMs to reduce cost.

### MySQL Configuration

* **Bind MySQL to a specific interface** in production using `bind-address` in `my.cnf` rather than listening on all interfaces (`:::`).
* **Enable MySQL SSL/TLS** for all remote connections to encrypt data in transit.
* **Take regular automated backups** using Azure Backup or `mysqldump` scheduled via cron.

### PHP Application

* **Use PDO with prepared statements** instead of raw `mysqli` for better security against SQL injection.
* **Implement connection pooling** for high-traffic applications to avoid exhausting MySQL's `max_connections` limit.
* **Add error logging** to a file rather than displaying raw connection errors to end users in production.

---

## Lessons Learned

### 1. Availability Zone Constraints on VM Sizes

When creating a VM, certain sizes are only available in specific Availability Zones within a region. In this lab, `Standard_D2s_v3` was unavailable in Zone 1 for Central US but available in Zone 2. Always verify zone availability before selecting a size, especially in lab or free-tier subscriptions that may have additional policy-based restrictions.

### 2. SSH Authentication Mode Must Match VM Provisioning

The `devops-php-vm` was provisioned with SSH public key authentication, which caused `Permission denied (publickey)` when connecting with a password. Always confirm the authentication method used during VM creation. For password-based SSH access, either select "Password" during VM creation or use **Azure Portal -- Reset password or keys** post-deployment.

### 3. NSG Rules Are Required at Both the Azure and OS Level

Configuring port 3306 in the Azure NSG is necessary but not always sufficient. If the Ubuntu OS firewall (`ufw`) is active, a corresponding OS-level rule must also be added. Always verify with:

```bash
sudo ufw status
sudo ss -tlnp | grep 3306
```

### 4. MySQL User Host Wildcard Is Critical for Remote Access

Creating a MySQL user with `'devops_user'@'localhost'` will silently block all remote connections. Using `'devops_user'@'%'` permits connections from any host. In production, replace `%` with the specific source IP of the PHP VM for tighter access control.

### 5. Variable Order in mysqli Constructor

The `mysqli` constructor signature is `mysqli($host, $username, $password, $dbname)`. The original placeholder file used `$servername` as the first argument, which is functionally correct, but renaming it to `$host` improves readability and reduces confusion when debugging connection issues. Parameter order errors in mysqli do not throw a named exception and can produce cryptic connection failures.

### 6. Shared VNet Simplifies Cross-VM Connectivity

Both VMs in this lab were attached to `devops-php-vmVNET`, which means private IP communication was available. In the absence of VNet peering or private connectivity, public IPs and open NSG rules were required. Proactively planning VNet topology before deployment avoids the need for publicly exposed database ports.

---

## Troubleshooting Reference

| Symptom | Root Cause | Resolution |
|---|---|---|
| `Connection refused` on port 3306 | NSG rule missing or OS firewall blocking | Verify NSG inbound rule for 3306 exists; check `sudo ufw status` |
| `Access denied for user 'devops_user'@'...'` | User created with `@'localhost'` not `@'%'` | Recreate user with `CREATE USER 'devops_user'@'%'` |
| `php_network_getaddresses: getaddrinfo failed` | Wrong or unreachable IP in `$host` | Verify `52.146.18.163` is the correct MySQL VM public IP |
| `Permission denied (publickey)` on SSH | VM provisioned with key auth, no password set | Use Azure Portal to reset password via **Reset password or keys** |
| Blank page or 500 error on PHP file | `mysqli` extension not installed | Run `sudo apt install php-mysql -y && sudo systemctl restart apache2` |
| `Connection failed: ...` in browser | Credentials mismatch or MySQL user not flushed | Re-verify `db_test.php` values and re-run `FLUSH PRIVILEGES` on MySQL VM |
| Apache not serving the PHP file | Apache2 not running | Run `sudo systemctl start apache2` |
| Size not available in zone error | VM size not supported in selected zone | Change Availability Zone (e.g., Zone 1 to Zone 2) |

---






<img width="1032" height="590" alt="Image" src="https://github.com/user-attachments/assets/6dab2141-8b9b-4892-925c-753651ad3b95" />

<img width="1028" height="582" alt="Image" src="https://github.com/user-attachments/assets/d6ecef60-0bd2-47b0-bc8f-1b90961e2d9c" />

<img width="1029" height="649" alt="Image" src="https://github.com/user-attachments/assets/39d87478-a37d-4c9a-8665-e64b0389ba67" />

<img width="1919" height="946" alt="Image" src="https://github.com/user-attachments/assets/025962bc-5cfb-43aa-b026-5ba42f9c48a9" />

<img width="1032" height="330" alt="Image" src="https://github.com/user-attachments/assets/6c13cd55-263d-45d5-a69b-b103b72c5fcf" />

<img width="1919" height="949" alt="Image" src="https://github.com/user-attachments/assets/115a94e6-95a1-49ec-90b6-afeb7115c7ff" />

<img width="1919" height="950" alt="Image" src="https://github.com/user-attachments/assets/632e2ba4-6899-4537-bbb2-0b0efee10348" />

<img width="1918" height="949" alt="Image" src="https://github.com/user-attachments/assets/b9984872-ffd7-4fb3-9f7e-1d352e4c8964" />

<img width="1035" height="800" alt="Image" src="https://github.com/user-attachments/assets/6e348919-2a97-4070-89cf-038b3ac59f9e" />

<img width="1029" height="669" alt="Image" src="https://github.com/user-attachments/assets/fd9f5430-829e-4c5d-a04c-f8f4902473dc" />

<img width="1029" height="510" alt="Image" src="https://github.com/user-attachments/assets/b04c2d79-f366-44fe-9d3a-49867d1ad228" />

<img width="1062" height="872" alt="Image" src="https://github.com/user-attachments/assets/8d83372d-1962-476b-b3a4-8b9aca1d477c" />

<img width="1035" height="416" alt="Image" src="https://github.com/user-attachments/assets/d8105c3e-6698-470f-81ed-8072b2dc92c5" />

<img width="1024" height="808" alt="Image" src="https://github.com/user-attachments/assets/cc029bc8-e6b7-4195-8365-f9c08a9f0ab2" />

<img width="1919" height="1023" alt="Image" src="https://github.com/user-attachments/assets/5b1bfabb-c04e-4de3-b3e5-bbe5abbee353" />
