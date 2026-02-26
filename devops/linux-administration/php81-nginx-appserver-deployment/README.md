# PHP 8.1 Application Deployment with Nginx (Unix Socket)

## Project Overview

This project documents the deployment of a `PHP-based application` on app server 3 within the Nautilus infrastructure at Stratos Datacenter, focusing on a production-ready setup. The deployment involves installing and configuring nginx to serve content on port 8096 with the document root `/var/www/html`, setting up `PHP-FPM 8.1` to use the Unix socket `/var/run/php-fpm/default.sock`, and integrating `nginx with PHP-FPM for optimized PHP processing`. The pre-provided application files `(index.php and info.php)` are preserved without modification, and the configuration is verified from the jump host using `curl http://stapp03:8096/index.php` to ensure full functionality and accessibility.

---

## Infrastructure Context

| Component | Value |
|--------|------|
| Server | stapp03 |
| OS | CentOS Stream 9 |
| Web Server | Nginx |
| PHP Runtime | PHP-FPM 8.1 |
| Port | 8096 |
| Document Root | /var/www/html |
| PHP Socket | /var/run/php-fpm/default.sock |

---

## Objectives

- Install and configure Nginx on a non-default port
- Install PHP-FPM version 8.1
- Configure PHP-FPM to use a Unix socket
- Integrate Nginx with PHP-FPM
- Validate application accessibility from jump host

---

## Step 1: Access Application Server

```
CONNECT to jump_host
SSH into stapp03
ACCEPT SSH fingerprint if prompted
VERIFY successful login
```

📸 Screenshot:
<img width="1035" height="551" alt="image" src="https://github.com/user-attachments/assets/71d7736e-16a3-458f-8e6c-7c38f1b2e591" />

## Step 2: Enable PHP 8.1 Repository
```
INSTALL Remi repository for EL9
RESET default PHP module stream
ENABLE php:remi-8.1 module
VERIFY module activation
```
📸 Screenshots:
<img width="1040" height="860" alt="image" src="https://github.com/user-attachments/assets/0495cbb4-b369-4d5f-8212-2d6c5c692b27" />
<img width="1038" height="707" alt="image" src="https://github.com/user-attachments/assets/06564189-3b35-40ec-90b3-32918179de29" />
<img width="1030" height="490" alt="image" src="https://github.com/user-attachments/assets/cd7d3418-c152-47ae-8335-6ef00d4a4cf3" />
<img width="1035" height="302" alt="image" src="https://github.com/user-attachments/assets/fcb7c2a5-f12d-47c2-a68d-659ac7250c94" />

## Step 3: Install Required Packages
```
INSTALL nginx package
INSTALL php-fpm package
VERIFY installation success
```
📸 Screenshots:
<img width="1034" height="841" alt="image" src="https://github.com/user-attachments/assets/7d4f2469-fe97-429b-8cc3-01fdbf5a2fce" />
<img width="1033" height="686" alt="image" src="https://github.com/user-attachments/assets/147ace65-0132-463f-99f0-1a144af024d8" />


## Step 4: Configure PHP-FPM Unix Socket
```
EDIT PHP-FPM pool configuration
SET listen path to /var/run/php-fpm/default.sock
SET socket owner to nginx
SET socket group to nginx
SET socket permissions to 0660
SAVE configuration
```
📸 Screenshot:
<img width="1037" height="303" alt="image" src="https://github.com/user-attachments/assets/375cc71c-3bb4-4a59-823b-56caa33eb8c9" />

## Step 5: Prepare Socket Directory
```
CREATE /var/run/php-fpm directory if missing
SET ownership to nginx:nginx
VERIFY permissions
```
📸 Screenshot:



## Step 6: Configure Nginx for PHP Application
```
CREATE nginx server configuration
SET listen port to 8096
SET document root to /var/www/html
DEFINE index files
CONFIGURE PHP request handling
PASS PHP requests to Unix socket
SET SCRIPT_FILENAME parameter
SAVE configuration
```
📸 Screenshot:


## Step 7: Enable and Start Services
```
ENABLE nginx service at boot
ENABLE php-fpm service at boot
START nginx
START php-fpm
VERIFY services are active
```
📸 Screenshot:



## Step 8: Local Application Validation
```
SEND HTTP request to localhost:8096
CONFIRM HTTP 200 response
VERIFY PHP version in headers
```
📸 Screenshot:

## Step 9: Remote Validation from Jump Host
```
EXIT stapp03 session
FROM jump_host
SEND HTTP request to stapp03:8096
CONFIRM application output
```
📸 Screenshot:
<img width="1036" height="584" alt="image" src="https://github.com/user-attachments/assets/b24fe68e-29a9-482d-a53a-bbe5c23d76a2" />

## System Design Notes

### Architecture Overview
```
CLIENT REQUEST
  → Nginx (Port 8096)
    → PHP-FPM (Unix Socket)
      → PHP Runtime Execution
        → HTML Response
```

#### Design Decisions
```
USE Unix socket instead of TCP
REASON: Lower latency and reduced overhead
USE non-default port (8096)
REASON: Avoid port conflicts and improve service isolation
USE php-fpm pool separation
REASON: Better process control and scalability
```
#### Performance Considerations
```
Unix sockets reduce network stack overhead
Faster IPC between Nginx and PHP-FPM
Lower CPU utilization under load
```
#### Security Considerations
```
LIMIT socket access via owner/group permissions
AVOID exposing PHP-FPM over TCP
RUN services with least privilege
```
#### Scalability Notes
```
CAN add multiple PHP-FPM pools per application
CAN front Nginx with load balancer
CAN integrate with CI/CD for config automation
```
#### Failure Scenarios & Mitigation
```
IF socket missing → php-fpm service down
MITIGATION: systemd monitoring and restart
IF port unavailable → nginx startup failure
MITIGATION: pre-flight port validation
```
## Result

- Nginx serves PHP application on port 8096

- PHP-FPM executes requests via Unix socket

- Application accessible from jump host

- Design aligns with production best practices

## Completion Checklist

| Status | Item |
|------|------|
| ✅ | PHP 8.1 Installed |
| ✅ | Nginx Configured |
| ✅ | Unix Socket Enabled |
| ✅ | Services Running |
| ✅ | System Design Documented |
| ✅ | Application Verified |








<img width="1036" height="828" alt="image" src="https://github.com/user-attachments/assets/71c1c0bd-70d6-4165-a19e-d8a12ba73635" />
<img width="1034" height="584" alt="image" src="https://github.com/user-attachments/assets/8f25e5f8-8c67-4135-8c02-bce20d808320" />
<img width="1032" height="307" alt="image" src="https://github.com/user-attachments/assets/a8aa66db-d62e-4543-9222-1a08f841a421" />
<img width="1034" height="357" alt="image" src="https://github.com/user-attachments/assets/efee23e6-2526-4af2-b0b1-a39ef054c667" />
<img width="1037" height="507" alt="image" src="https://github.com/user-attachments/assets/c8c661f5-7291-4069-a704-d7e6030261c8" />


