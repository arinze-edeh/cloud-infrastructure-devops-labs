# WordPress Multi-Host Deployment with Centralized MariaDB on Stratos Datacenter

> **Enterprise-style multi-tier web application deployment** across a horizontally scaled Apache/PHP cluster backed by a dedicated MariaDB instance, with shared NFS storage and load-balanced ingress validation.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution Overview](#solution-overview)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Infrastructure Inventory](#infrastructure-inventory)
- [Shared Storage Design](#shared-storage-design)
- [Implementation](#implementation)
  - [Phase 1: Database Server Provisioning (stdb01)](#phase-1-database-server-provisioning-stdb01)
  - [Phase 2: Application Server Provisioning (stapp01, stapp02, stapp03)](#phase-2-application-server-provisioning-stapp01-stapp02-stapp03)
  - [Phase 3: WordPress Database Configuration](#phase-3-wordpress-database-configuration)
  - [Phase 4: Service Restart Across All Application Hosts](#phase-4-service-restart-across-all-application-hosts)
- [Validation](#validation)
- [Validation Checklist](#validation-checklist)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Key Learnings](#key-learnings)

---

## Problem Statement

A multi-tier web application required deployment across the Stratos Datacenter with the following constraints:

- WordPress files must be served consistently from **three independent application servers** without content drift or duplication overhead.
- A **single centralized database** must handle all application connections with proper access controls.
- Apache must be configured to **listen on a non-standard port (3002)** to satisfy load balancer routing rules.
- The deployment must be validated through the **Load Balancer endpoint** to confirm full-stack connectivity, not just individual host availability.

Without a shared storage layer and a consistent configuration baseline across all hosts, serving coherent WordPress content through a load balancer becomes operationally fragile and difficult to maintain.

---

## Solution Overview

The solution provisions a **three-node Apache/PHP cluster** backed by a **dedicated MariaDB host**, with a pre-mounted **shared NFS volume** at `/var/www/html` ensuring content consistency across all application servers. WordPress database credentials are deployed via `wp-config.php` into the shared volume, making the configuration immediately available to all three hosts without redundant file placement.

Validation is performed at two levels: HTTP header inspection at the application layer and database connectivity verification through the Load Balancer UI.

---

## Architecture

```
                  ┌───────────────────┐
                  │   Load Balancer   │
                  │  (LBR Endpoint)   │
                  └─────────┬─────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
   ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
   │  stapp01    │   │  stapp02    │   │  stapp03    │
   │  tony       │   │  steve      │   │  banner     │
   │  Apache:3002│   │  Apache:3002│   │  Apache:3002│
   │  PHP 8.0    │   │  PHP 8.0    │   │  PHP 8.0    │
   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                   Shared NFS Mount
                   /var/www/html
                   (wp-config.php)
                            │
                   ┌────────▼────────┐
                   │    stdb01       │
                   │    peter        │
                   │    MariaDB 10.5 │
                   │    kodekloud_db5│
                   └─────────────────┘
```

**Key design characteristics:**

- All three application hosts mount the same NFS share at `/var/www/html`, so `wp-config.php` is written once and served from all nodes simultaneously.
- MariaDB grants database access via the wildcard host (`'%'`), allowing connections from any application server IP without per-host ACL management.
- Apache is reconfigured to listen on port 3002 on all hosts to match the upstream load balancer routing configuration.

---

## Technologies Used

| Component | Technology | Version |
|---|---|---|
| Web Server | Apache HTTP Server (`httpd`) | 2.4.62 |
| Scripting Runtime | PHP + extensions (`php-mysqlnd`, `php-gd`, `php-mbstring`) | 8.0.30 |
| Database Engine | MariaDB Server | 10.5.29 |
| Operating System | CentOS Stream 9 | el9 |
| Package Manager | `yum` (DNF backend) | |
| Storage | NFS Shared Volume | |
| Service Manager | systemd | |
| Access Path | Load Balancer (LBR) | |

---

## Infrastructure Inventory

| Role | Hostname | User | IP Address |
|---|---|---|---|
| Database Server | stdb01 | peter | 172.16.239.10 |
| Application Server 1 | stapp01 | tony | 172.16.238.10 |
| Application Server 2 | stapp02 | steve | 172.16.238.11 |
| Application Server 3 | stapp03 | banner | 172.16.238.12 |
| Jumphost | jumphost | thor | |

All SSH sessions are initiated from `thor@jumphost` using password authentication.

---

## Shared Storage Design

The shared NFS volume is pre-provisioned and mounted at `/var/www/html` across all three application servers. This architecture provides:

- **Content consistency:** A single `wp-config.php` write propagates to all hosts without manual replication.
- **Operational simplicity:** WordPress core files are managed from one storage location, eliminating drift between nodes.
- **Horizontal scalability:** Additional application nodes can be added to the NFS mount without requiring content bootstrapping.

**Note:** The storage server exports `/var/www/html`. Each application host consumes this export at the same path, making the Apache document root identical across the cluster.

---

## Implementation

### Phase 1: Database Server Provisioning (stdb01)

#### 1.1 Establish SSH Session

Connect to the database server from the jumphost:

```bash
ssh peter@172.16.239.10
```

Accept the host key fingerprint on first connection. The prompt confirms successful login as `[peter@stdb01 ~]$`.

**Screenshot: SSH into stdb01**

![SSH connection to stdb01](https://github.com/user-attachments/assets/6f26cdb4-4926-48b4-8417-3418eab7eec0)

---

#### 1.2 Install MariaDB Server

Install the MariaDB server package and all required dependencies from the CentOS AppStream and EPEL repositories:

```bash
sudo yum install -y mariadb-server
```

This pulls in the full MariaDB ecosystem including `mariadb-common`, `mariadb-connector-c`, `mariadb-errmsg`, `mariadb-gssapi-server`, and all required Perl dependencies for administration tooling.

**Screenshot: MariaDB package installation beginning**

![MariaDB yum install package resolution](https://github.com/user-attachments/assets/0a44a21e-5397-432d-9309-e824aef942b5)

**Screenshot: MariaDB installation complete (75/75 packages verified)**

![MariaDB installation verified and complete](https://github.com/user-attachments/assets/cffbbb38-fa55-4346-9b68-043da2c5774f)

---

#### 1.3 Start and Enable MariaDB

Start the MariaDB service immediately and configure it to start automatically on system boot:

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

The `enable` command creates the necessary systemd symlinks under `/etc/systemd/system/`:

```
Created symlink /etc/systemd/system/mysql.service -> /usr/lib/systemd/system/mariadb.service
Created symlink /etc/systemd/system/mysqld.service -> /usr/lib/systemd/system/mariadb.service
Created symlink /etc/systemd/system/multi-user.target.wants/mariadb.service -> /usr/lib/systemd/system/mariadb.service
```

**Screenshot: MariaDB service started and enabled with systemd symlinks confirmed**

![MariaDB systemctl start and enable output](https://github.com/user-attachments/assets/f03b1e76-4ebf-44d2-98c1-81838af57a7b)

---

#### 1.4 Configure Database and Application User

Access the MariaDB shell as root using `sudo`:

```bash
sudo mysql
```

Execute the following SQL statements to provision the application database and its dedicated user:

```sql
CREATE DATABASE kodekloud_db5;

CREATE USER 'kodekloud_cap'@'%' IDENTIFIED BY 'ksH85UJjhb';

GRANT ALL PRIVILEGES ON kodekloud_db5.* TO 'kodekloud_cap'@'%';

FLUSH PRIVILEGES;

EXIT;
```

**Design rationale:**

- The `'%'` wildcard host allows the application user to connect from any IP, which is appropriate in an NFS-backed cluster where all application hosts must reach the same database. In environments with stricter controls, this should be scoped to the specific application subnet.
- `FLUSH PRIVILEGES` ensures grant table changes take effect immediately without requiring a service restart.
- A dedicated `kodekloud_cap` user with scoped permissions (`kodekloud_db5.*` only) follows the principle of least privilege.

**Screenshot: Database and user creation confirmed with "Query OK" responses**

![MariaDB database and user provisioning](https://github.com/user-attachments/assets/60cdf792-9b8e-4be7-917e-fc385bfded1a)

---

### Phase 2: Application Server Provisioning (stapp01, stapp02, stapp03)

The following steps are executed identically on all three application servers. Each server is accessed individually from the jumphost.

---

#### 2.1 stapp01 (tony @ 172.16.238.10)

**Connect to stapp01:**

```bash
ssh tony@172.16.238.10
```

**Install Apache and PHP dependencies:**

```bash
sudo yum install -y httpd php php-mysqlnd php-gd php-mbstring
```

This installs the Apache web server (`httpd 2.4.62`), the PHP runtime (`8.0.30`), and the following PHP extensions required for WordPress operation:

- `php-mysqlnd`: MySQL native driver for database connectivity
- `php-gd`: Image processing library
- `php-mbstring`: Multibyte string support for internationalization

**Screenshot: stapp01 Apache and PHP package installation initiated**

![stapp01 yum install httpd php packages](https://github.com/user-attachments/assets/2efd2453-0c8d-4708-8a9e-baedb72b0e7b)

**Screenshot: stapp01 installation complete (42/42 packages verified)**

![stapp01 httpd php installation complete](https://github.com/user-attachments/assets/9cc8556c-d09a-4002-84b7-244d31f5129e)

---

#### 2.2 stapp02 (steve @ 172.16.238.11)

**Connect to stapp02:**

```bash
ssh steve@172.16.238.11
```

**Install Apache and PHP dependencies:**

```bash
sudo yum install -y httpd php php-mysqlnd php-gd php-mbstring
```

**Screenshot: stapp02 Apache and PHP package installation initiated**

![stapp02 yum install httpd php packages](https://github.com/user-attachments/assets/2ed76a89-04d1-4940-bee9-6845ca7ac8ff)

**Screenshot: stapp02 installation complete (42/42 packages verified)**

![stapp02 httpd php installation complete](https://github.com/user-attachments/assets/44f3586c-07ee-469f-bf6d-50b96c44f16a)

---

#### 2.3 stapp03 (banner @ 172.16.238.12)

**Connect to stapp03:**

```bash
ssh banner@172.16.238.12
```

**Install Apache and PHP dependencies:**

```bash
sudo yum install -y httpd php php-mysqlnd php-gd php-mbstring
```

**Screenshot: stapp03 Apache and PHP package installation initiated**

![stapp03 yum install httpd php packages](https://github.com/user-attachments/assets/17fd52d4-a11d-4952-9696-b2ac3d55d17e)

**Screenshot: stapp03 installation complete (42/42 packages verified)**

![stapp03 httpd php installation complete](https://github.com/user-attachments/assets/1e308fe0-3e38-4ef6-8f40-6a8371d8660d)

---

#### 2.4 Configure Apache Port on All Application Servers

The load balancer routes traffic to port `3002` on each application host. Apache defaults to port `80` and must be reconfigured. This change is applied to all three hosts.

**On each application server, execute:**

```bash
sudo sed -i 's/Listen 80/Listen 3002/g' /etc/httpd/conf/httpd.conf
```

**Then start and enable the Apache service:**

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

**Screenshot: stapp01 port reconfigured, Apache started and enabled**

![stapp01 sed port change, httpd start and enable](https://github.com/user-attachments/assets/3422a69d-ced5-44b6-a0b0-4243c91ac126)

**Screenshot: stapp02 port reconfigured, Apache started and enabled**

![stapp02 sed port change, httpd start and enable](https://github.com/user-attachments/assets/655b8912-5aa0-4cac-806b-0ee6326fc982)

**Screenshot: stapp03 port reconfigured, Apache started and enabled**

![stapp03 sed port change, httpd start and enable](https://github.com/user-attachments/assets/b9745257-3321-4834-b9ac-a16c7928229c)

**Operational note:** The `sed -i` inline substitution modifies `httpd.conf` in place. Verify the change was applied correctly with `grep "Listen" /etc/httpd/conf/httpd.conf` before starting the service in production environments.

---

### Phase 3: WordPress Database Configuration

Because all three application servers mount the same NFS share at `/var/www/html`, writing `wp-config.php` from **any single application server** makes it immediately available to all nodes. This is performed on `stapp01`.

**Connect to stapp01:**

```bash
ssh tony@172.16.238.10
```

**Write the WordPress database configuration file:**

```bash
cat << 'EOF' | sudo tee /var/www/html/wp-config.php
<?php
define( 'DB_NAME', 'kodekloud_db5' );
define( 'DB_USER', 'kodekloud_cap' );
define( 'DB_PASSWORD', 'ksH85UJjhb' );
define( 'DB_HOST', '172.16.239.10' );
EOF
```

**Parameter mapping:**

| WordPress Constant | Value | Source |
|---|---|---|
| `DB_NAME` | `kodekloud_db5` | Database created in Phase 1 |
| `DB_USER` | `kodekloud_cap` | Application user created in Phase 1 |
| `DB_PASSWORD` | `ksH85UJjhb` | Password set during user creation |
| `DB_HOST` | `172.16.239.10` | IP address of stdb01 |

**Screenshot: wp-config.php written to shared NFS volume, content confirmed**

![wp-config.php heredoc written to /var/www/html](https://github.com/user-attachments/assets/0ba7f6c2-44e2-44f1-8437-c83dd22781b6)

The terminal output shows the file content echoed back by `tee`, confirming successful write. Since `/var/www/html` is the NFS-shared document root, all three application servers immediately have access to this configuration.

---

### Phase 4: Service Restart Across All Application Hosts

A final Apache restart is performed on all three application servers to ensure the running service picks up the current configuration state and any NFS-backed file changes.

**stapp01:**

```bash
ssh tony@172.16.238.10
sudo systemctl restart httpd
```

**stapp02:**

```bash
ssh steve@172.16.238.11
sudo systemctl restart httpd
```

**stapp03:**

```bash
ssh banner@172.16.238.12
sudo systemctl restart httpd
```

**Screenshot: Apache restarted on stapp01, stapp02, and stapp03 in sequence**

![httpd restart on all three application servers](https://github.com/user-attachments/assets/c96d5254-0749-4c75-b87b-919f952e7b86)

**Screenshot: Final restart sequence across all three app servers confirmed**

![Final httpd restart confirmation from jumphost](https://github.com/user-attachments/assets/27b88a05-1bba-4b23-9344-5345ae9a8388)

---

## Validation

### HTTP Reachability Check

From the jumphost, issue an HTTP HEAD request to confirm Apache is serving on port 3002:

```bash
curl -I http://172.16.238.10:3002
```

**Expected response:**

```
HTTP/1.1 200 OK
Date: Tue, 24 Feb 2026 06:20:30 GMT
Server: Apache/2.4.62 (CentOS Stream)
X-Powered-By: PHP/8.0.30
Content-Type: text/html; charset=UTF-8
```

The presence of `X-Powered-By: PHP/8.0.30` confirms that PHP is active and the web server is correctly processing PHP requests.

**Screenshot: curl -I confirms HTTP 200 with Apache and PHP headers from stapp01**

![curl -I http response confirming Apache and PHP on port 3002](https://github.com/user-attachments/assets/8a3f2a43-11fa-4a36-a789-7c2a9516501f)

---

### Load Balancer Application Verification

Navigate to the Load Balancer UI and click the **App** button. The PHP application reads `wp-config.php` from the shared NFS volume, attempts a connection to MariaDB on `stdb01` using the `kodekloud_cap` credentials, and renders the connection result.

**Expected output:**

```
App is able to connect to the database using user kodekloud_cap
```

This message confirms:

- The NFS share is correctly mounted and readable by Apache.
- `wp-config.php` contains valid database credentials.
- The MariaDB instance on `stdb01` is reachable from the application servers.
- The `kodekloud_cap` user has the correct privileges on `kodekloud_db5`.
- The load balancer is routing traffic correctly to the backend cluster.

**Screenshot: LBR UI confirms successful database connectivity via PHP application**

![Browser showing "App is able to connect to the database using user kodekloud_cap"](https://github.com/user-attachments/assets/561d7ffd-54cd-4f6d-b188-a6297f75e704)

---

## Validation Checklist

| Validation Check | Status |
|---|---|
| SSH access confirmed to all hosts from jumphost | Passed |
| MariaDB installed and service running on stdb01 | Passed |
| `kodekloud_db5` database created | Passed |
| `kodekloud_cap` user created with wildcard host and correct password | Passed |
| Full privileges granted on `kodekloud_db5.*` | Passed |
| `FLUSH PRIVILEGES` executed | Passed |
| Apache and PHP installed on stapp01, stapp02, stapp03 | Passed |
| Apache reconfigured to listen on port 3002 on all app hosts | Passed |
| `httpd` service started and enabled on all app hosts | Passed |
| `wp-config.php` written to shared NFS volume at `/var/www/html` | Passed |
| Apache restarted on all hosts after config write | Passed |
| `curl -I` returns HTTP 200 with Apache and PHP headers | Passed |
| LBR confirms database connectivity via `kodekloud_cap` | Passed |

---

## Operational Considerations

**Port conflict awareness:** Port 3002 was selected to satisfy the load balancer's routing rules. In KodeKloud environments, `ttyd` permanently occupies port 8080 on lab hosts. Always confirm that your chosen port is not already bound before starting the web server.

**Wildcard database host:** Granting `'kodekloud_cap'@'%'` allows connections from any IP. In production environments, restrict this to the application subnet CIDR (e.g., `'172.16.238.%'`) to limit the database attack surface.

**Single wp-config.php write:** Because all application hosts share the same NFS mount, writing `wp-config.php` once is sufficient. Writing the file redundantly from each host is unnecessary and risks race conditions if the NFS mount is temporarily unavailable on any node.

**Service dependency ordering:** MariaDB must be fully running before any application server attempts a PHP database connection. In orchestrated deployments, use systemd dependency declarations or readiness probes to enforce this ordering.

**NFS mount persistence:** Verify that the NFS mount is configured in `/etc/fstab` on each application host to survive reboots. A lost mount after a restart would cause all three servers to serve from an empty or default document root.

---

## Troubleshooting Reference

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `curl -I` returns connection refused | Apache not started or wrong port in `httpd.conf` | Verify with `grep Listen /etc/httpd/conf/httpd.conf` and `systemctl status httpd` |
| LBR shows database connection error | Incorrect credentials in `wp-config.php` or MariaDB user not granted | Re-verify `wp-config.php` values and run `SHOW GRANTS FOR 'kodekloud_cap'@'%';` in MariaDB |
| Apache starts but PHP not processing | PHP not installed or module not loaded | Confirm with `php -v` and check `httpd -M | grep php` |
| `wp-config.php` not visible on all app hosts | NFS mount missing or stale | Run `df -h /var/www/html` on each host to confirm the NFS mount is active |
| MariaDB refuses remote connections | `bind-address` set to `127.0.0.1` in my.cnf | Comment out or set `bind-address = 0.0.0.0` in `/etc/my.cnf.d/mariadb-server.cnf` and restart MariaDB |

---

## Key Learnings

- **Port consistency is non-negotiable in load-balanced deployments.** All backend hosts must listen on the identical port that the load balancer expects. A single host on the wrong port causes intermittent failures that are difficult to debug under round-robin routing.

- **Shared storage simplifies multi-host configuration management.** Writing application configuration to an NFS-mounted document root once eliminates the need for configuration management tools or manual file distribution across nodes.

- **Wildcard database grants enable cluster flexibility but require network-level controls.** Using `'%'` as the host in the user grant removes per-host ACL maintenance overhead but shifts security responsibility to firewall rules and network segmentation.

- **LBR-level validation is the authoritative end-to-end test.** Individual host reachability does not confirm full-stack correctness. Only a successful response through the load balancer confirms that routing, application logic, shared storage, and database access are all functioning together.

- **systemd enable is as important as systemd start.** Starting a service is sufficient for immediate operation. Enabling it ensures the service survives host reboots, which is essential in shared infrastructure environments where hosts may be cycled independently.




























# WordPress Multi-Host Deployment on Stratos Datacenter

## Project Overview

This project documents the deployment of a **highly available WordPress application** across multiple application servers with a centralized MariaDB backend inside the **Stratos Datacenter**.
The infrastructure was preconfigured with a **shared NFS volume** mounted at `/var/www/html` across all application hosts, enabling consistent WordPress content delivery behind a Load Balancer (LBR).

The objective was to validate **end-to-end application-to-database connectivity** through the LBR endpoint.

---

## Architecture Summary

```
                  ┌───────────────┐
                  │ Load Balancer │
                  │   (LBR UI)    │
                  └───────┬───────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
 ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
 │ App Server  │   │ App Server  │   │ App Server  │
 │  stapp01    │   │  stapp02    │   │  stapp03    │
 │ Apache:3002 │   │ Apache:3002 │   │ Apache:3002 │
 └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
        │                 │                 │
        └────────── Shared NFS ─────────────┘
                   /var/www/html
                          │
                   ┌──────▼──────┐
                   │ DB Server   │
                   │ MariaDB     │
                   │ kodekloud_db5│
                   └─────────────┘
```

---

## 🔧 Technologies Used

* **Apache HTTP Server (httpd)**
* **PHP 8.x**
* **MariaDB 10.5**
* **CentOS Stream 9**
* **NFS Shared Storage**
* **Linux Systemd Services**
* **Load Balancer (LBR)**

---

## 📂 Shared Storage

* **Storage Server Path:** `/vaw/www/html`
* **Mounted On App Hosts:** `/var/www/html`
* Purpose: Single source of truth for WordPress files across all app servers

---

## 🚀 Implementation Steps

### 1️⃣ Database Server Configuration (stdb01)

#### Install & Enable MariaDB

```bash
sudo yum install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

📸 **Screenshots:**
<img width="1028" height="486" alt="image" src="https://github.com/user-attachments/assets/6f26cdb4-4926-48b4-8417-3418eab7eec0" />
<img width="1036" height="859" alt="image" src="https://github.com/user-attachments/assets/0a44a21e-5397-432d-9309-e824aef942b5" />
<img width="1032" height="863" alt="image" src="https://github.com/user-attachments/assets/cffbbb38-fa55-4346-9b68-043da2c5774f" />
<img width="1032" height="655" alt="image" src="https://github.com/user-attachments/assets/f03b1e76-4ebf-44d2-98c1-81838af57a7b" />


---

#### Database & User Setup

```sql
CREATE DATABASE kodekloud_db5;
CREATE USER 'kodekloud_cap'@'%' IDENTIFIED BY 'ksH85UJjhb';
GRANT ALL PRIVILEGES ON kodekloud_db5.* TO 'kodekloud_cap'@'%';
FLUSH PRIVILEGES;
```

📸 **Screenshot:**
<img width="1032" height="598" alt="image" src="https://github.com/user-attachments/assets/60cdf792-9b8e-4be7-917e-fc385bfded1a" />

---

### 2️⃣ Application Servers Setup (stapp01, stapp02, stapp03)

#### Install Apache & PHP Dependencies

```bash
sudo yum install -y httpd php php-mysqlnd php-gd php-mbstring
```

📸 **Screenshots:**
<img width="1033" height="853" alt="image" src="https://github.com/user-attachments/assets/2efd2453-0c8d-4708-8a9e-baedb72b0e7b" />
<img width="1031" height="859" alt="image" src="https://github.com/user-attachments/assets/9cc8556c-d09a-4002-84b7-244d31f5129e" />

<img width="1032" height="869" alt="image" src="https://github.com/user-attachments/assets/2ed76a89-04d1-4940-bee9-6845ca7ac8ff" />
<img width="1036" height="865" alt="image" src="https://github.com/user-attachments/assets/44f3586c-07ee-469f-bf6d-50b96c44f16a" />
<img width="1036" height="867" alt="image" src="https://github.com/user-attachments/assets/17fd52d4-a11d-4952-9696-b2ac3d55d17e" />
<img width="1037" height="865" alt="image" src="https://github.com/user-attachments/assets/1e308fe0-3e38-4ef6-8f40-6a8371d8660d" />

---

#### Configure Apache to Listen on Port 3002

```bash
sudo sed -i 's/Listen 80/Listen 3002/g' /etc/httpd/conf/httpd.conf
```

---

#### Start, Restart, & Enable Apache

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl restart httpd
```

📸 **Screenshot:**

<img width="1033" height="438" alt="image" src="https://github.com/user-attachments/assets/3422a69d-ced5-44b6-a0b0-4243c91ac126" />


<img width="1035" height="453" alt="image" src="https://github.com/user-attachments/assets/655b8912-5aa0-4cac-806b-0ee6326fc982" />

<img width="1031" height="451" alt="image" src="https://github.com/user-attachments/assets/b9745257-3321-4834-b9ac-a16c7928229c" />
<img width="1035" height="467" alt="image" src="https://github.com/user-attachments/assets/c96d5254-0749-4c75-b87b-919f952e7b86" />
<img width="1032" height="293" alt="image" src="https://github.com/user-attachments/assets/27b88a05-1bba-4b23-9344-5345ae9a8388" />
<img width="1025" height="459" alt="image" src="https://github.com/user-attachments/assets/da73f4f8-28a4-4eff-ba16-3dc800b536ce" />

---

### 3️⃣ WordPress Database Configuration

Create `wp-config.php` inside the shared directory:

```bash
sudo tee /var/www/html/wp-config.php <<EOF
<?php
define( 'DB_NAME', 'kodekloud_db5' );
define( 'DB_USER', 'kodekloud_cap' );
define( 'DB_PASSWORD', 'ksH85UJjhb' );
define( 'DB_HOST', '172.16.239.10' );
EOF
```

📸 **Screenshot:**
<img width="1041" height="428" alt="image" src="https://github.com/user-attachments/assets/0ba7f6c2-44e2-44f1-8437-c83dd22781b6" />

---

### 4️⃣ Service Restart (All App Servers)

```bash
sudo systemctl restart httpd
```

---

## 🔍 Validation & Health Checks

### Apache Reachability

```bash
curl -I http://<app-server-ip>:3002
```

**Expected Result**

```
HTTP/1.1 200 OK
Server: Apache/2.4.x
X-Powered-By: PHP
```

📸 **Screenshot:**

<img width="1034" height="786" alt="image" src="https://github.com/user-attachments/assets/8a3f2a43-11fa-4a36-a789-7c2a9516501f" />

---

### LBR Application Verification

* Navigate to **LBR UI**
* Click **App** button on the top bar

✅ **Expected Message**

```
App is able to connect to the database using user kodekloud_cap
```

📸 **Screenshot:**

<img width="1919" height="1024" alt="image" src="https://github.com/user-attachments/assets/561d7ffd-54cd-4f6d-b188-a6297f75e704" />

---

## ✅ Validation Checklist

| Check                              | Status |
| ---------------------------------- | ------ |
| MariaDB installed & running        | ✅      |
| Database created                   | ✅      |
| DB user created & privileged       | ✅      |
| Apache installed on all app hosts  | ✅      |
| Apache listening on port 3002      | ✅      |
| Shared storage mounted             | ✅      |
| WordPress DB connectivity verified | ✅      |
| LBR access successful              | ✅      |

---

## 🎯 Final Outcome

* WordPress successfully deployed across **multiple app servers**
* Database centralized on a **dedicated MariaDB host**
* Application accessible via **Load Balancer**
* Verified **end-to-end database connectivity**
* Architecture supports **horizontal scaling & high availability**

---

## 🧠 Key Learnings

* Multi-tier WordPress deployments require strict **port consistency**
* Shared storage simplifies multi-host content synchronization
* Explicit database grants prevent silent WordPress failures
* LBR validation is the final proof of full-stack correctness
---
