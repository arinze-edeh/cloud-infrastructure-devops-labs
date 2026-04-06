# PHP 8.1 Application Deployment with Nginx and Unix Socket on CentOS Stream 9

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure Context](#infrastructure-context)
- [Objectives](#objectives)
- [Architecture](#architecture)
- [Implementation](#implementation)
  - [Step 1: Access Application Server](#step-1-access-application-server)
  - [Step 2: Enable PHP 8.1 Repository](#step-2-enable-php-81-repository)
  - [Step 3: Install Required Packages](#step-3-install-required-packages)
  - [Step 4: Configure PHP-FPM Unix Socket](#step-4-configure-php-fpm-unix-socket)
  - [Step 5: Prepare Socket Directory](#step-5-prepare-socket-directory)
  - [Step 6: Configure Nginx for PHP Application](#step-6-configure-nginx-for-php-application)
  - [Step 7: Enable and Start Services](#step-7-enable-and-start-services)
  - [Step 8: Local Application Validation](#step-8-local-application-validation)
  - [Step 9: Remote Validation from Jump Host](#step-9-remote-validation-from-jump-host)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Operational Considerations](#operational-considerations)
- [Lessons Learned](#lessons-learned)
- [Completion Checklist](#completion-checklist)

---

## Overview

This document details the end-to-end deployment of a PHP-based web application on **stapp03** within the Nautilus infrastructure at the Stratos Datacenter. The setup delivers a production-ready configuration using **Nginx** as the HTTP frontend and **PHP-FPM 8.1** as the process manager, communicating over a **Unix domain socket** for optimal inter-process performance.

The deployment covers repository configuration for a non-default PHP version, PHP-FPM pool socket tuning, Nginx virtual host configuration on a non-standard port, and full end-to-end validation from the jump host.

---

## Problem Statement

The Nautilus application team requires a PHP web application to be deployed on **stapp03** under the following constraints:

- The application must be served on **port 8096** to avoid conflicts with existing services.
- The document root must be located at **/var/www/html**, where pre-provided application files (`index.php`, `info.php`) exist and must not be modified.
- **PHP-FPM 8.1** must be installed from the Remi repository, as the CentOS Stream 9 default module stream does not carry this version.
- Nginx must forward PHP requests to PHP-FPM exclusively via a **Unix socket** at `/var/run/php-fpm/default.sock`, avoiding TCP socket overhead.
- The application must be verified as accessible from the jump host using `curl http://stapp03:8096/index.php`.

---

## Infrastructure Context

| Component | Value |
|---|---|
| **Target Server** | stapp03 |
| **Operating System** | CentOS Stream 9 |
| **Web Server** | Nginx 1.20.1 |
| **PHP Runtime** | PHP-FPM 8.1.34 (Remi modular) |
| **Listening Port** | 8096 |
| **Document Root** | /var/www/html |
| **PHP-FPM Socket** | /var/run/php-fpm/default.sock |
| **Access Method** | SSH from jump host (thor@jumphost) |

---

## Objectives

- Install and configure Nginx to serve on a non-default port
- Install PHP-FPM version 8.1 from the Remi repository on CentOS Stream 9
- Configure the PHP-FPM pool to use a Unix domain socket with correct ownership and permissions
- Integrate Nginx with PHP-FPM via the Unix socket using FastCGI
- Validate full request flow from the jump host

---

## Architecture

```
CLIENT REQUEST
  |
  v
Nginx (Port 8096)
  |  FastCGI over Unix Socket
  v
PHP-FPM 8.1 (/var/run/php-fpm/default.sock)
  |
  v
PHP Runtime Execution (/var/www/html)
  |
  v
HTTP Response to Client
```

**Why Unix Socket over TCP?**

Unix sockets bypass the full network stack for inter-process communication. When Nginx and PHP-FPM run on the same host, this eliminates TCP handshake overhead, reduces CPU utilization, and delivers measurably lower latency per request compared to a loopback TCP connection.

---

## Implementation

### Step 1: Access Application Server

From the jump host, SSH into `stapp03` as the application user `banner`. Accept the ED25519 host fingerprint on first connection, which is added permanently to `known_hosts`.

```bash
# From jump host
ssh banner@stapp03
```

Authenticate with the user password when prompted. Confirm the shell prompt changes to `[banner@stapp03 ~]$` before proceeding.

> **Operational Note:** Always verify the host fingerprint against the known infrastructure record on first connection. Blindly accepting fingerprints in production environments is a security risk.

**Screenshot: SSH session established from jump host to stapp03**

![SSH session to stapp03](https://github.com/user-attachments/assets/71d7736e-16a3-458f-8e6c-7c38f1b2e591)

---

### Step 2: Enable PHP 8.1 Repository

CentOS Stream 9 does not ship PHP 8.1 in its default AppStream module. The Remi repository provides a modular PHP 8.1 package set for EL9. The EL8 Remi RPM is incompatible with EL9 and will fail with dependency conflicts; the correct target is the EL9 release.

**2a. Attempt EL8 Remi release (produces a dependency conflict, resolved next):**

```bash
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm
```

This command fails due to `epel-release = 8` and `system-release(releasever) = 8` dependencies not being satisfiable on EL9. This error is expected and guides the correction.

**Screenshot: EL8 Remi RPM fails with conflicting requests on CentOS Stream 9**

![Remi EL8 conflict error](https://github.com/user-attachments/assets/0495cbb4-b369-4d5f-8212-2d6c5c692b27)

**2b. Install the correct EL9 Remi release:**

```bash
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
```

The `remi-release-9.7-4.el9.remi.noarch` package installs successfully, enabling both the Remi modular and safe repositories for EL9.

**Screenshot: Remi EL9 RPM installs successfully**

![Remi EL9 install success](https://github.com/user-attachments/assets/06564189-3b35-40ec-90b3-32918179de29)

**2c. Reset the default PHP module stream:**

```bash
sudo dnf module reset php -y
```

This clears any active PHP module stream from AppStream, ensuring there are no conflicts when enabling the Remi stream. Output confirms `Nothing to do` because no default PHP stream was previously active.

**Screenshot: PHP module stream reset completes**

![PHP module reset](https://github.com/user-attachments/assets/cd7d3418-c152-47ae-8335-6ef00d4a4cf3)

**2d. Enable the PHP 8.1 Remi module stream:**

```bash
sudo dnf module enable php:remi-8.1 -y
```

The transaction enables the `php` module stream pinned to `remi-8.1`. Subsequent installs of `php-fpm` will resolve from this stream.

**Screenshot: php:remi-8.1 module stream enabled**

![PHP remi-8.1 module enabled](https://github.com/user-attachments/assets/fcb7c2a5-f12d-47c2-a68d-659ac7250c94)

---

### Step 3: Install Required Packages

Install Nginx and PHP-FPM in a single transaction. DNF resolves all required dependencies including `nginx-core`, `nginx-filesystem`, `php-common`, `httpd-filesystem`, and `centos-logos-httpd`.

```bash
sudo dnf install -y nginx php-fpm
```

The transaction installs 7 packages totaling approximately 28 MB. PHP-FPM resolves to version `8.1.34-1.module_php.8.1.el9.remi` from the `remi-modular` repository, confirming the correct stream is active.

**Screenshot: Package resolution showing nginx and php-fpm from correct repositories**

![DNF install nginx php-fpm resolution](https://github.com/user-attachments/assets/7d4f2469-fe97-429b-8cc3-01fdbf5a2fce)

**Screenshot: Full install transaction completes with all 7 packages verified**

![DNF install transaction complete](https://github.com/user-attachments/assets/147ace65-0132-463f-99f0-1a144af024d8)

---

### Step 4: Configure PHP-FPM Unix Socket

The default PHP-FPM pool configuration at `/etc/php-fpm.d/www.conf` uses a TCP socket (`127.0.0.1:9000`). This must be changed to a Unix socket and the socket ownership must be set to the `nginx` user so Nginx can write to it.

Use `sed` to apply all required changes in a single, atomic operation:

```bash
sudo sed -i \
  -e 's|^listen =.*|listen = /var/run/php-fpm/default.sock|' \
  -e 's|^;listen.owner =.*|listen.owner = nginx|' \
  -e 's|^;listen.group =.*|listen.group = nginx|' \
  -e 's|^;listen.mode =.*|listen.mode = 0660|' \
  /etc/php-fpm.d/www.conf
```

> **Why `sed` over a text editor?** In automated and reproducible deployments, `sed` is preferred for targeted in-place substitutions. It is idempotent when combined with anchored patterns and avoids editor-specific behaviors.

**Verify the applied configuration:**

```bash
grep -E "^listen =|^listen.owner|^listen.group|^listen.mode" /etc/php-fpm.d/www.conf
```

Expected output confirms all four directives are correctly set:

```
listen = /var/run/php-fpm/default.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

**Screenshot: sed command applied and configuration verified via grep**

![PHP-FPM sed configuration](https://github.com/user-attachments/assets/375cc71c-3bb4-4a59-823b-56caa33eb8c9)

**Screenshot: PHP-FPM pool configuration verification output**

![PHP-FPM config grep verification](https://github.com/user-attachments/assets/8f25e5f8-8c67-4135-8c02-bce20d808320)

---

### Step 5: Prepare Socket Directory

The directory `/var/run/php-fpm/` must exist before PHP-FPM starts, and it must be owned by the `nginx` user so that the socket file created at runtime is accessible to Nginx.

```bash
sudo mkdir -p /var/run/php-fpm/
sudo chown nginx:nginx /var/run/php-fpm/
```

> **Operational Note:** On systems using `tmpfs` for `/var/run`, this directory will not persist across reboots. For production environments, add a `tmpfiles.d` configuration entry or a systemd `RuntimeDirectory` directive to recreate it automatically on boot.

**Screenshot: Socket directory created and ownership assigned to nginx**

![Socket directory creation](https://github.com/user-attachments/assets/a8aa66db-d62e-4543-9222-1a08f841a421)

---

### Step 6: Configure Nginx for PHP Application

Create a dedicated Nginx server block configuration for the PHP application. This is written to `/etc/nginx/conf.d/php_app.conf` to keep it separate from the default Nginx configuration.

```bash
sudo tee /etc/nginx/conf.d/php_app.conf <<EOF
server {
    listen 8096;
    server_name _;
    root /var/www/html;
    index index.php index.html;

    location / {
        try_files \$uri \$uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/var/run/php-fpm/default.sock;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
    }
}
EOF
```

**Configuration breakdown:**

- **`listen 8096`** serves on the required non-standard port
- **`server_name _`** acts as a catch-all, accepting requests regardless of the Host header
- **`root /var/www/html`** points to the pre-existing application files
- **`try_files $uri $uri/ =404`** handles static file resolution with a clean 404 fallback
- **`location ~ \.php$`** matches all `.php` request URIs and forwards them to PHP-FPM
- **`fastcgi_pass unix:/var/run/php-fpm/default.sock`** uses the Unix socket instead of TCP
- **`fastcgi_param SCRIPT_FILENAME`** instructs PHP-FPM on the full filesystem path to execute

**Screenshot: Nginx server block written and echoed to terminal via tee**

![Nginx configuration written](https://github.com/user-attachments/assets/71c1c0bd-70d6-4165-a19e-d8a12ba73635)

---

### Step 7: Enable and Start Services

Enable both services for automatic startup at boot and start them immediately using a single `systemctl` command.

```bash
sudo systemctl enable --now nginx php-fpm
```

This creates the required systemd symlinks in `multi-user.target.wants/` for both services and starts them. Both services must reach the `active (running)` state before validation.

**Screenshot: systemctl enables nginx and php-fpm with symlinks created**

![systemctl enable --now nginx php-fpm](https://github.com/user-attachments/assets/efee23e6-2526-4af2-b0b1-a39ef054c667)

---

### Step 8: Local Application Validation

From within `stapp03`, send an HTTP HEAD request to confirm the application is reachable on port 8096 and that PHP-FPM is processing requests correctly.

```bash
curl -I http://localhost:8096/index.php
```

**Expected response:**

```
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Thu, 26 Feb 2026 03:48:52 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/8.1.34
```

The `HTTP/1.1 200 OK` status confirms Nginx received the request. The `X-Powered-By: PHP/8.1.34` header confirms PHP-FPM 8.1 processed it. The `Server: nginx/1.20.1` header confirms the correct web server is handling the request.

**Screenshot: curl -I confirms HTTP 200 with PHP/8.1.34 header from localhost**

![Local curl validation](https://github.com/user-attachments/assets/c8c661f5-7291-4069-a704-d7e6030261c8)

---

### Step 9: Remote Validation from Jump Host

Exit the `stapp03` session and confirm end-to-end accessibility from the jump host, which represents the client perspective.

```bash
exit
curl http://stapp03:8096/index.php
```

**Expected output:**

```
Welcome to xFusionCorp Industries!
```

A successful response confirms that network routing from the jump host to stapp03 on port 8096 is functioning, Nginx is accepting external connections, and the PHP application is returning the expected output.

**Screenshot: Remote curl from jump host returns application output**

![Remote curl from jump host](https://github.com/user-attachments/assets/b24fe68e-29a9-482d-a53a-bbe5c23d76a2)

---

## Key Decisions

| Decision | Rationale |
|---|---|
| **Unix socket over TCP** | Eliminates TCP stack overhead for same-host IPC; lower latency and CPU cost |
| **Non-default port 8096** | Avoids collision with standard HTTP (80) and any existing services on the host |
| **Remi EL9 repository** | CentOS Stream 9 AppStream does not provide PHP 8.1; Remi is the authoritative modular source |
| **`dnf module reset` before enable** | Prevents stream conflict errors when switching from any previously active PHP module |
| **`sed` for config edits** | Reproducible, scriptable, and avoids interactive editor inconsistencies in automation pipelines |
| **Dedicated conf.d file** | Keeps the virtual host isolated from the default Nginx config; simplifies rollback and auditing |
| **`systemctl enable --now`** | Combines enable and start in a single operation; ensures the service survives reboots |

---

## Errors and Resolutions

### Error: Remi EL8 RPM fails on CentOS Stream 9

**Symptom:**

```
Problem: conflicting requests
  - nothing provides epel-release = 8 needed by remi-release-8.10-2.el8.remi.noarch
  - nothing provides system-release(releasever) = 8 needed by remi-release-8.10-2.el8.remi.noarch
```

**Cause:** The EL8 Remi release RPM requires `epel-release = 8` and a system release version of 8. CentOS Stream 9 satisfies neither.

**Resolution:** Install the correct EL9-targeted Remi release RPM:

```bash
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
```

**Lesson:** Always match the Remi release RPM to the major EL version of the target OS. Confirm with `cat /etc/os-release` before downloading the Remi package.

---

## Operational Considerations

**Socket directory persistence on tmpfs:**
`/var/run` is commonly mounted as `tmpfs` and cleared on reboot. The manually created `/var/run/php-fpm/` directory will not survive a reboot without an explicit systemd `RuntimeDirectory` directive or a `tmpfiles.d` entry. For persistent production deployments, add the following:

```
# /etc/tmpfiles.d/php-fpm.conf
d /var/run/php-fpm 0755 nginx nginx -
```

**SELinux considerations:**
If SELinux is enforcing, Nginx may be denied access to the Unix socket. Verify with `getenforce`. If enforcing, apply the appropriate boolean:

```bash
setsebool -P httpd_can_network_connect 1
```

Or generate a custom policy from `audit2allow` output.

**Firewall:**
If `firewalld` is active, port 8096 must be explicitly permitted:

```bash
sudo firewall-cmd --add-port=8096/tcp --permanent
sudo firewall-cmd --reload
```

**PHP-FPM pool tuning:**
The default `www` pool uses dynamic process management. For production, tune `pm.max_children`, `pm.start_servers`, `pm.min_spare_servers`, and `pm.max_spare_servers` based on expected concurrency and available memory.

**Nginx default server conflict:**
If the default Nginx configuration also listens on a conflicting port or uses `default_server`, the new virtual host may not be reached as expected. Disable or remove conflicting server blocks in `/etc/nginx/nginx.conf`.

---

## Lessons Learned

- **Version-specific repositories require OS version alignment.** Installing a repository RPM without verifying the EL version match leads to immediate dependency conflicts. Always check the OS major version first.
- **`dnf module reset` is a prerequisite to stream switching.** Even when no PHP module is currently active, running `reset` first is a safe and idempotent guard against hidden stream locks.
- **Unix socket ownership must match the Nginx worker user.** If `listen.owner` and `listen.group` do not match the Nginx process user, PHP-FPM will create the socket but Nginx will receive a permission denied error at request time.
- **`systemctl enable --now` is more reliable than separate enable and start calls.** It is atomic and avoids the common error of enabling a service but forgetting to start it during deployment.
- **Pre-flight validation with `curl -I` is faster than checking logs.** A single HTTP header response reveals both the web server version and the PHP backend version simultaneously, confirming the full stack in one command.

---

## Completion Checklist

| Status | Item |
|---|---|
| **✅** | Remi EL9 repository installed and PHP 8.1 module stream enabled |
| **✅** | Nginx and PHP-FPM 8.1 packages installed |
| **✅** | PHP-FPM pool configured with Unix socket at `/var/run/php-fpm/default.sock` |
| **✅** | Socket directory created with correct `nginx:nginx` ownership |
| **✅** | Nginx virtual host configured on port 8096 with FastCGI integration |
| **✅** | Both services enabled and started via systemd |
| **✅** | Local validation confirms HTTP 200 with PHP/8.1.34 header |
| **✅** | Remote validation from jump host returns expected application output |
