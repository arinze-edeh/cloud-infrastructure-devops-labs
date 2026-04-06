# Apache Static Website Deployment on App Server 2 (stapp02) -- xFusionCorp Industries

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Prerequisites](#prerequisites)
- [Step 1: Transfer Website Backups from Jump Host to App Server 2](#step-1-transfer-website-backups-from-jump-host-to-app-server-2)
- [Step 2: Install Apache HTTP Server](#step-2-install-apache-http-server)
- [Step 3: Configure Apache to Listen on Port 8085](#step-3-configure-apache-to-listen-on-port-8085)
- [Step 4: Start and Enable Apache Service](#step-4-start-and-enable-apache-service)
- [Step 5: Move Websites to DocumentRoot](#step-5-move-websites-to-documentroot)
- [Step 6: Set Ownership and Permissions](#step-6-set-ownership-and-permissions)
- [Step 7: Validate Local Access via Curl](#step-7-validate-local-access-via-curl)
- [Validation Checklist](#validation-checklist)
- [Final Outcome](#final-outcome)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)

---

## Overview

This document covers the end-to-end deployment of two static websites -- **ecommerce** and **demo** -- on **App Server 2 (stapp02)** within the Stratos Datacenter. Apache HTTP Server is installed and configured to serve both sites from a custom port (**8085**) on CentOS Stream 9. Website assets originate on the jump host and are migrated via SCP prior to deployment. The final state is verified using `curl` against both endpoints.

---

## Problem Statement

Two static website packages (`ecommerce` and `demo`) are stored on the jump host (`thor@jumphost`) and must be made accessible via HTTP on App Server 2 (`stapp02`, `172.16.238.11`). The default Apache port (80) is not suitable for this environment; the requirement is port **8085**. The deployment must be repeatable, correctly permissioned, and locally verifiable before any upstream routing is configured.

---

## Architecture

```
             +------------------+
             |   Jump Host      |
             |  (thor@jumphost) |
             +--------+---------+
                      |
                      |  SCP Transfer (port 22)
                      v
             +------------------+
             |  App Server 2    |
             |    stapp02       |
             |  Apache : 8085   |
             +--------+---------+
                      |
       +--------------+--------------+
       |   Web Root (/var/www/html)  |
       |  +------------+ +---------+ |
       |  | ecommerce/ | |  demo/  | |
       |  +------------+ +---------+ |
       +-----------------------------+
```

**Traffic flow:** All inbound HTTP requests on port 8085 are served by Apache (`httpd`). Both sites share the same DocumentRoot under subdirectories, eliminating the need for virtual host configuration at this stage.

---

## Technologies Used

- **Apache HTTP Server** (`httpd`) -- web server
- **CentOS Stream 9** -- operating system on stapp02
- **Linux Systemd** -- service management
- **SCP** (Secure Copy Protocol) -- cross-host file transfer
- **Bash** -- command execution

---

## Prerequisites

- SSH access from the jump host to stapp02 (`steve@172.16.238.11`)
- Website packages (`ecommerce/`, `demo/`) present under `/home/thor/` on the jump host
- `sudo` privileges for the `steve` user on stapp02
- Outbound access from stapp02 to package repositories (for `yum install`)

---

## Step 1: Transfer Website Backups from Jump Host to App Server 2

### 1.1 Execute SCP Transfers from the Jump Host

Both website directories are copied from the jump host to the `/tmp/` staging directory on stapp02. Using `/tmp/` as a staging area separates the transfer step from the deployment step, reducing the risk of partial deployments landing directly in the web root.

```bash
# Transfer the ecommerce site
scp -r /home/thor/ecommerce steve@172.16.238.11:/tmp/

# Transfer the demo site
scp -r /home/thor/demo steve@172.16.238.11:/tmp/
```

**Operational note:** On first connection, the SSH client will prompt to accept the host fingerprint. Confirm by typing `yes`. The fingerprint is then persisted in `~/.ssh/known_hosts` on the jump host, so subsequent connections skip this prompt.

**Screenshot -- SCP transfers completed from jump host to stapp02:**

![SCP transfer from jump_host to stapp02](https://github.com/user-attachments/assets/3192f0e8-070b-4298-a452-e75709d4f092)

*Both `ecommerce` and `demo` directories transferred successfully at 196.8 KB/s and 223.4 KB/s respectively, with 100% completion confirmed.*

---

### 1.2 SSH into App Server 2

After the transfers complete, open an interactive session on stapp02 to perform all remaining steps locally.

```bash
ssh steve@172.16.238.11
```

**Screenshot -- Active SSH session on stapp02:**

![SSH session established on stapp02](https://github.com/user-attachments/assets/8db66fff-f303-4a49-a3ca-367abd851bab)

*Shell prompt changes to `[steve@stapp02 ~]$`, confirming successful authentication and session establishment.*

---

## Step 2: Install Apache HTTP Server

Install the `httpd` package and all required dependencies using the system package manager. The `yum` transaction resolves and installs 13 packages automatically, including APR libraries, `httpd-core`, `mod_http2`, and `mod_lua`.

```bash
sudo yum install -y httpd
```

**Screenshot -- Apache installation: dependency resolution and download:**

![Apache yum install dependency resolution](https://github.com/user-attachments/assets/c88f4c76-b8d6-4624-8bef-3e85b080c884)

*Package manager resolves 13 packages totalling 9.5 MB installed size.*

**Screenshot -- Apache installation: transaction completion:**

![Apache yum install transaction complete](https://github.com/user-attachments/assets/a593cf76-4f93-4b93-97d8-4e90bb2b8231)

*All 13 packages installed and verified. Transaction completes with `Complete!` status.*

---

## Step 3: Configure Apache to Listen on Port 8085

The default Apache configuration listens on port 80. This must be changed to port 8085 to meet the environment requirement. A `sed` in-place substitution modifies the directive directly in `httpd.conf`, avoiding manual file editing which introduces risk of syntax errors.

### 3.1 Update the Listening Port

```bash
sudo sed -i 's/Listen 80/Listen 8085/g' /etc/httpd/conf/httpd.conf
```

### 3.2 Verify the Configuration Change

```bash
grep "Listen 8085" /etc/httpd/conf/httpd.conf
```

**Expected output:**
```
Listen 8085
```

**Screenshot -- Port configuration applied and verified:**

![Apache port 8085 configuration verified](https://github.com/user-attachments/assets/306bd313-3d6a-486b-aa65-295b0dbe76a6)

*`grep` confirms `Listen 8085` is present in `httpd.conf`. The `sed` substitution was successful.*

**Operational note:** If the environment enforces SELinux, port 8085 must also be added to the `http_port_t` type using `semanage port -a -t http_port_t -p tcp 8085`. Verify SELinux status with `getenforce` before proceeding. In this environment, the configuration applied without SELinux intervention.

---

## Step 4: Start and Enable Apache Service

Start the Apache service immediately and configure it to start automatically on system boot. Both operations are issued sequentially.

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

The `enable` command creates a systemd symlink in `multi-user.target.wants/`, ensuring `httpd` starts at every boot without manual intervention.

**Screenshot -- Apache started and enabled at boot:**

![Apache systemctl start and enable](https://github.com/user-attachments/assets/3f9fefa7-6ffd-43dd-8a0a-80448f67a547)

*Symlink created at `/etc/systemd/system/multi-user.target.wants/httpd.service`, confirming boot-time persistence.*

**Verification command (recommended after enabling):**
```bash
sudo systemctl status httpd
```

Expected status: `active (running)`.

---

## Step 5: Move Websites to DocumentRoot

Relocate both website directories from the `/tmp/` staging area to Apache's DocumentRoot at `/var/www/html/`. This makes the content immediately available under the default site root.

```bash
sudo mv /tmp/ecommerce /var/www/html/
sudo mv /tmp/demo /var/www/html/
```

**Screenshot -- Website directories moved to DocumentRoot:**

![Move websites to /var/www/html](https://github.com/user-attachments/assets/578240d7-883c-4421-94f0-77db2aea525c)

*Both `mv` commands execute without error. Directories are now under `/var/www/html/ecommerce/` and `/var/www/html/demo/`.*

---

## Step 6: Set Ownership and Permissions

Assign ownership of both website directories to the `apache` system user and group. Apache (`httpd`) runs under this account, and without correct ownership the process will return HTTP 403 Forbidden errors when attempting to serve files.

```bash
sudo chown -R apache:apache /var/www/html/ecommerce
sudo chown -R apache:apache /var/www/html/demo
```

The `-R` flag applies ownership recursively across all files and subdirectories within each site.

**Screenshot -- Ownership set for both website directories:**

![chown apache ownership applied](https://github.com/user-attachments/assets/3567b2fe-b0c4-4f96-b55b-6a13aef6835a)

*Ownership transferred to `apache:apache` for both directories. No errors returned.*

**Best practice:** File permissions should also be validated. Web content directories should be `755` (directories) and `644` (files) to prevent world-writable conditions while allowing Apache to read and serve content.

```bash
# Recommended post-deployment permission check
find /var/www/html/ecommerce -type d -exec chmod 755 {} \;
find /var/www/html/ecommerce -type f -exec chmod 644 {} \;
find /var/www/html/demo -type d -exec chmod 755 {} \;
find /var/www/html/demo -type f -exec chmod 644 {} \;
```

---

## Step 7: Validate Local Access via Curl

Confirm that both websites are served correctly over HTTP on port 8085 by issuing `curl` requests from within stapp02. This validates the full stack: Apache is running, listening on the correct port, reading from the correct DocumentRoot, and returning valid HTML content.

### 7.1 Ecommerce Site

```bash
curl http://localhost:8085/ecommerce/
```

**Screenshot -- Curl response for ecommerce site:**

![Curl validation - ecommerce site](https://github.com/user-attachments/assets/1dced30a-cbc0-44f5-89f8-90c69b0780da)

*Apache returns valid HTML for the ecommerce site, including the `<h1>KodeKloud</h1>` heading and paragraph content. HTTP response is served correctly on port 8085.*

### 7.2 Demo Site

```bash
curl http://localhost:8085/demo/
```

**Screenshot -- Curl response for both ecommerce and demo sites:**

![Curl validation - ecommerce and demo sites](https://github.com/user-attachments/assets/1d252416-c8b9-4b3d-ad29-9e75cdfbf708)

*Both `curl` commands return valid HTML. The demo site returns its `<p>This is a sample page for our demo website</p>` content, confirming full end-to-end delivery.*

---

## Validation Checklist

| Check | Status |
|---|---|
| Apache installed and running | Pass |
| Apache listening on port 8085 | Pass |
| Website backups transferred via SCP | Pass |
| Websites moved to `/var/www/html/` | Pass |
| Ownership set to `apache:apache` | Pass |
| Ecommerce site reachable via curl | Pass |
| Demo site reachable via curl | Pass |

---

## Final Outcome

Both static websites are deployed and verified on **stapp02** with the following configuration:

- **Ecommerce site:** `http://localhost:8085/ecommerce/`
- **Demo site:** `http://localhost:8085/demo/`
- **Web server:** Apache `httpd 2.4.62` on CentOS Stream 9
- **Custom port:** 8085 (modified from default 80)
- **Service persistence:** Enabled via systemd at boot
- **File ownership:** `apache:apache` applied recursively
- **Deployment state:** Ready for upstream routing or reverse proxy integration

---

## Key Decisions

- **SCP to `/tmp/` first:** Staging files in `/tmp/` before moving to the web root isolates the transfer from the deployment. A failed SCP will not leave a broken or partial directory in `/var/www/html/`.
- **`sed` for port modification:** Using `sed -i` for in-place substitution is safer and more repeatable than manual file editing, particularly in scripted or automated deployment scenarios.
- **Subdirectory serving (no virtual hosts):** Both sites are served as subdirectories under a single DocumentRoot. This is appropriate for same-server, same-port multi-site deployments where virtual hosting is not required.
- **Recursive `chown`:** Applying ownership at the directory level with `-R` ensures all nested assets (CSS, images, JS) are covered without requiring per-file intervention.

---

## Errors and Resolutions

**Issue:** SSH host fingerprint prompt on first SCP connection.
**Resolution:** Type `yes` to accept and persist the fingerprint. All subsequent connections proceed without prompting.

**Risk:** If SELinux is in `Enforcing` mode, Apache cannot bind to non-standard ports without an explicit policy addition.
**Resolution:** Run `semanage port -a -t http_port_t -p tcp 8085` before starting `httpd`. Confirm enforcement status with `getenforce`.

**Risk:** Incorrect file ownership results in HTTP 403 Forbidden responses.
**Resolution:** Always apply `chown -R apache:apache` to the web root content directories before testing with `curl`.

---

## Lessons Learned

- **SCP is reliable for point-in-time asset transfer** between Linux hosts but is not idempotent. For repeatable deployments, consider Ansible or rsync with checksumming.
- **Apache subdirectory serving** requires no additional virtual host configuration, making it a fast path for multi-site deployments on a single server and port.
- **Ownership and permissions are a leading cause of 403 errors.** Always validate file ownership immediately after moving content to the web root, before any service restart.
- **Custom ports require explicit `Listen` directives.** The `httpd.conf` change must be in place before the service starts; changing the port on a running instance requires a service reload or restart to take effect.
- **Using `systemctl enable` alongside `systemctl start`** is the correct production pattern. Omitting `enable` means the service will not survive a host reboot.




















# Static Websites Deployment on App Server 2 – xFusionCorp Industries

## Project Overview

- This project documents the deployment of two static websites (ecommerce and demo) on App Server 2 (stapp02) in the Stratos Datacenter. Apache HTTP Server is installed and configured to serve the websites on custom port 8085.

- Website backups are initially stored on the jump_host and need to be migrated to stapp02 before deployment. The objective is to ensure local accessibility of the websites via curl commands.

## Architecture Summary

             ┌───────────────┐
             │ Jump Host     │
             │ (thor@jump)  │
             └───────┬──────┘
                     │ SCP Transfer
                     ▼
             ┌───────────────┐
             │ App Server 2  │
             │ stapp02       │
             │ Apache:8085   │
             └───────┬──────┘
                     │
      ┌──────────────┴──────────────┐
      │ Web Root (/var/www/html)    │
      │ ┌───────────┐  ┌─────────┐ │
      │ │ ecommerce │  │ demo    │ │
      │ └───────────┘  └─────────┘ │
      └────────────────────────────┘

## 🔧 Technologies Used

- Apache HTTP Server (httpd)

- CentOS Stream 9

- Linux Systemd Services

- SCP (Secure Copy Protocol)

- Bash Shell Scripting

## Step 1: Transfer Website Backups from Jump Host to App Server 2

### 1.1 Execute SCP Commands from Jump Host (thor@jump_host)

- Transfer the ecommerce backup
 
`scp -r /home/thor/ecommerce steve@172.16.238.11:/tmp/`

- Transfer the demo backup

`scp -r /home/thor/demo steve@172.16.238.11:/tmp/`

📸 Screenshot: `SCP transfer from jump_host to stapp02`
<img width="1028" height="593" alt="image" src="https://github.com/user-attachments/assets/3192f0e8-070b-4298-a452-e75709d4f092" />

### 1.2 SSH into App Server 2

`ssh steve@172.16.238.11`

📸 Screenshot: `SCP transfer from jump_host to stapp02`
<img width="1032" height="607" alt="image" src="https://github.com/user-attachments/assets/8db66fff-f303-4a49-a3ca-367abd851bab" />

## Step 2: Install Apache HTTP Server
- `sudo yum install -y httpd`

📸 Screenshot: `Apache installation`
<img width="1031" height="862" alt="image" src="https://github.com/user-attachments/assets/c88f4c76-b8d6-4624-8bef-3e85b080c884" />
<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/a593cf76-4f93-4b93-97d8-4e90bb2b8231" />

## Step 3: Configure Apache to Listen on Port 8085

### Update the listening port from 80 to 8085
`sudo sed -i 's/Listen 80/Listen 8085/g' /etc/httpd/conf/httpd.conf`

### Verify the change
`grep "Listen 8085" /etc/httpd/conf/httpd.conf`

📸 Screenshot: `Apache port configuration`

<img width="1031" height="867" alt="image" src="https://github.com/user-attachments/assets/306bd313-3d6a-486b-aa65-295b0dbe76a6" />

## Step 4: Start and Enable Apache Service
````
sudo systemctl start httpd
sudo systemctl enable httpd
````

📸 Screenshot:`Apache start and enable`
<img width="1033" height="566" alt="image" src="https://github.com/user-attachments/assets/3f9fefa7-6ffd-43dd-8a0a-80448f67a547" />

## Step 5: Move Websites to DocumentRoot
````
sudo mv /tmp/ecommerce /var/www/html/
sudo mv /tmp/demo /var/www/html/
````
📸 Screenshot: `Move websites`
<img width="1036" height="472" alt="image" src="https://github.com/user-attachments/assets/578240d7-883c-4421-94f0-77db2aea525c" />

## Step 6: Set Ownership and Permissions
````
sudo chown -R apache:apache /var/www/html/ecommerce
sudo chown -R apache:apache /var/www/html/demo
````

📸 Screenshot: ` set ownership`
<img width="1036" height="429" alt="image" src="https://github.com/user-attachments/assets/3567b2fe-b0c4-4f96-b55b-6a13aef6835a" />

## Step 7: Validate Local Access via Curl

### 7.1 Ecommerce Site
`curl http://localhost:8085/ecommerce/`

### 7.2 Demo Site
`curl http://localhost:8085/demo/`

Expected Output: HTML content of respective websites

📸 Screenshot: `Curl validation - ecommerce`
<img width="1030" height="565" alt="image" src="https://github.com/user-attachments/assets/1dced30a-cbc0-44f5-89f8-90c69b0780da" />

📸 Screenshot: `Curl validation - demo`
<img width="1025" height="755" alt="image" src="https://github.com/user-attachments/assets/1d252416-c8b9-4b3d-ad29-9e75cdfbf708" />

## ✅ Validation Checklist
| **Check**                      | **Status** |
| ------------------------------ | ---------- |
| Apache installed & running     | ✅          |
| Apache listening on port 8085  | ✅          |
| Website backups transferred    | ✅          |
| Websites moved to DocumentRoot | ✅          |
| Ownership & permissions set    | ✅          |
| Websites reachable via curl    | ✅          |


## Final Outcome

- Two static websites deployed on stapp02

Websites accessible locally on:

- `http://localhost:8085/ecommerce/`

- `http://localhost:8085/demo/`

- Apache configured on custom `port 8085`

- Ownership and permissions configured for proper access

- Ready for further integration with production environment

## Key Learnings

- SCP is essential for transferring files across servers

- Apache allows serving multiple websites via subdirectories

- Correct ownership and permissions prevent common access errors

- Custom ports require explicit configuration in `httpd.conf`
