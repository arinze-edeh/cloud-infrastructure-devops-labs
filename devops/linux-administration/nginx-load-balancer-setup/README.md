# Nginx HTTP Load Balancer: High Availability Traffic Distribution Across Apache Backend Servers

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Infrastructure Details](#infrastructure-details)
- [Objectives](#objectives)
- [Implementation](#implementation)
  - [Step 1: Access the Load Balancer](#step-1-access-the-load-balancer)
  - [Step 2: Install and Enable Nginx](#step-2-install-and-enable-nginx)
  - [Step 3: Verify Apache on Backend App Servers](#step-3-verify-apache-on-backend-app-servers)
  - [Step 4: Configure Nginx Load Balancing](#step-4-configure-nginx-load-balancing)
  - [Step 5: Validate and Reload Nginx](#step-5-validate-and-reload-nginx)
- [Application Validation](#application-validation)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Lessons Learned](#lessons-learned)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

The Nautilus Production Support Team experienced increased traffic and progressive performance degradation on a production website hosted in the Stratos Data Center. To address this, the application was migrated to a high-availability architecture distributing load across multiple Apache HTTP servers.

This document covers the final and most critical phase of that migration: provisioning and configuring an Nginx-based HTTP load balancer (`stlb01`) to distribute incoming traffic across three Apache application servers (`stapp01`, `stapp02`, `stapp03`), each running on a non-standard port (`5004`). A core constraint was that backend Apache ports must not be modified under any circumstance, preserving the integrity of the existing application layer.

---

## Problem Statement

**Context:** A single-server deployment was unable to sustain growing production traffic, resulting in latency spikes and service degradation.

**Constraint:** Backend Apache servers were already configured and running on port `5004`. Modifying their configuration was out of scope and not permitted.

**Solution:** Deploy Nginx as a reverse proxy and load balancer in front of three backend Apache instances. The Nginx configuration must route all inbound HTTP traffic on port `80` to the upstream pool on port `5004`, with no changes to the backend service configuration.

---

## Architecture

```text
              Client (HTTP Request)
                      |
                      v
         Nginx Load Balancer (stlb01)
               172.16.238.14:80
                      |
         +------------+------------+
         |            |            |
         v            v            v
   stapp01:5004  stapp02:5004  stapp03:5004
  172.16.238.10 172.16.238.11 172.16.238.12
    (Apache)      (Apache)      (Apache)
```

Traffic enters through the Nginx load balancer on port `80`. Nginx distributes requests in round-robin fashion across the three Apache upstream servers, each listening on port `5004`. No backend server configuration is altered.

---

## Infrastructure Details

| Role          | Hostname | IP Address    | Service            | Port |
| ------------- | -------- | ------------- | ------------------ | ---- |
| Load Balancer | stlb01   | 172.16.238.14 | Nginx              | 80   |
| App Server 1  | stapp01  | 172.16.238.10 | Apache (httpd)     | 5004 |
| App Server 2  | stapp02  | 172.16.238.11 | Apache (httpd)     | 5004 |
| App Server 3  | stapp03  | 172.16.238.12 | Apache (httpd)     | 5004 |
| Jump Host     | jumphost | Dynamic       | SSH Bastion Access | 22   |

---

## Objectives

- Install and configure Nginx on `stlb01` as an HTTP load balancer
- Configure upstream load balancing exclusively in `/etc/nginx/nginx.conf`
- Preserve existing Apache port (`5004`) on all backend servers without modification
- Validate Nginx configuration syntax before applying changes
- Confirm end-to-end traffic routing via the StaticApp URL

---

## Implementation

### Step 1: Access the Load Balancer

All operations begin from the jump host. SSH into the load balancer node (`stlb01`) using the `loki` user account.

```bash
ssh loki@172.16.238.14
```

> **Operational Note:** On first connection, the host key fingerprint is verified and permanently added to `~/.ssh/known_hosts`. This is standard SSH host key exchange behavior and does not indicate a security issue.

📸 **SSH login to stlb01 from the jump host, confirming connectivity and initiating Nginx installation**

![SSH login to Load Balancer stlb01](https://github.com/user-attachments/assets/31ddbf55-3e92-4f88-b5af-f1544a73e02c)

---

### Step 2: Install and Enable Nginx

Install the Nginx package from the CentOS Stream 9 AppStream repository, enable it for automatic startup on reboot, then start and verify the service.

```bash
sudo yum install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

> **Best Practice:** Always enable a service alongside starting it. Without `systemctl enable`, the service will not persist across system reboots, which is a common operational oversight in production environments.

📸 **`yum install` resolving Nginx and its dependencies from the AppStream and EPEL repositories**

![Nginx yum install package resolution](https://github.com/user-attachments/assets/2f23828e-6ee0-4c17-b988-9de4310d0e9a)

📸 **Download and transaction completion: 5 packages installed including `nginx`, `nginx-core`, `nginx-filesystem`, `centos-logos-httpd`, and `logrotate`**

![Nginx installation transaction complete](https://github.com/user-attachments/assets/568108e7-fd09-481d-87e5-a0ae00c39a33)

📸 **`systemctl enable` creating the systemd symlink for persistent startup**

![Nginx enabled and started via systemctl](https://github.com/user-attachments/assets/95b84ad9-7d70-464d-86b1-e342fbd8584e)

📸 **Nginx service status showing `active (running)` with master and worker processes confirmed via systemd**

![Nginx service active and running](https://github.com/user-attachments/assets/a7678bd7-bc2d-4896-81e9-93756376f015)

---

### Step 3: Verify Apache on Backend App Servers

Before configuring load balancing, verify that Apache (`httpd`) is running and listening on port `5004` across all three backend servers. This step is critical: it confirms the upstream target port before writing the Nginx upstream block.

> **Constraint:** Backend Apache servers must not be modified. Verification is read-only.

**stapp01 (172.16.238.10)**

```bash
ssh tony@172.16.238.10
sudo systemctl status httpd
```

📸 **Apache `httpd` active and running on `stapp01` with 5 worker processes**

![Apache status on stapp01](https://github.com/user-attachments/assets/355640a1-9ba2-4886-b73e-daf5055cc02c)

---

**stapp02 (172.16.238.11)**

```bash
ssh steve@172.16.238.11
sudo systemctl status httpd
```

📸 **Apache `httpd` active and running on `stapp02`, confirming the same process structure**

![Apache status on stapp02](https://github.com/user-attachments/assets/ffdbf250-7ddf-45ca-8223-3ebce461e9fa)

---

**stapp03 (172.16.238.12)**

```bash
ssh banner@172.16.238.12
sudo systemctl status httpd
```

📸 **Apache `httpd` active and running on `stapp03`, all three backend nodes confirmed healthy**

![Apache status on stapp03](https://github.com/user-attachments/assets/f815f010-ff60-435e-bb25-7691b40eae34)

---

### Step 4: Configure Nginx Load Balancing

Return to `stlb01` and modify the main Nginx configuration file. Per the task constraint, **only `/etc/nginx/nginx.conf` is modified.** No files under `/etc/nginx/conf.d/` are used.

```bash
ssh loki@172.16.238.14
sudo vi /etc/nginx/nginx.conf
```

The `upstream` block and `server` block are added within the existing `http {}` context. The `upstream` directive defines the pool of backend servers. The `server` block configures Nginx to listen on port `80` and proxy all requests to the upstream group.

**Final `http {}` block additions:**

```nginx
upstream app_servers {
    server 172.16.238.10:5004;  # stapp01
    server 172.16.238.11:5004;  # stapp02
    server 172.16.238.12:5004;  # stapp03
}

server {
    listen 80;

    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> **Design Note:** The `proxy_set_header` directives ensure that backend Apache servers receive the original client IP and protocol information. This is required for accurate access logging, security auditing, and any application logic dependent on the originating client context.

> **Operational Note:** By default, Nginx uses round-robin load balancing when no algorithm is explicitly specified. For stateless applications, this is sufficient. For session-persistent workloads, `ip_hash` or `least_conn` would be more appropriate.

📸 **`nginx.conf` in edit mode showing the `upstream app_servers` block and `server` block correctly configured within the `http {}` context**

![Nginx upstream and server block configuration](https://github.com/user-attachments/assets/bc473fd7-2a06-4687-8c7e-e997243e7684)

---

### Step 5: Validate and Reload Nginx

Before reloading the service, validate the configuration syntax. This prevents service disruption from misconfigurations.

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Expected output:**

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

> **Best Practice:** Always run `nginx -t` before applying any configuration change. A failed test leaves the running configuration intact. Reloading without testing risks a full service outage if the configuration contains syntax errors.

> **`reload` vs `restart`:** `systemctl reload nginx` sends a `HUP` signal, causing Nginx to reload configuration without dropping active connections. This is the preferred zero-downtime approach in production environments.

📸 **Initial `nginx -t` failure due to a syntax error (`unexpected "}"` at line 83), then the corrected configuration passing both syntax and file checks**

![Nginx config test failure then success](https://github.com/user-attachments/assets/feee1d48-774c-46ed-ae04-bf92109c97c5)

📸 **Port verification on `stapp01` using `ss -lntp | grep httpd` confirming Apache listening on `0.0.0.0:5004`, followed by successful `nginx -t` and `systemctl reload nginx` on `stlb01`**

![Port verification and Nginx reload confirmation](https://github.com/user-attachments/assets/0965d1d1-b11c-4cd9-8e62-b02cf011cce8)

---

## Application Validation

With Nginx reloaded and the upstream pool configured, the application was accessed via the StaticApp URL provided by the KodeKloud environment:

```
http://80-port-<environment-id>.labs.kodekloud.com
```

**Validation results:**

- Load balancer responded on port `80`
- Requests were proxied to backend Apache servers on port `5004`
- No `502 Bad Gateway` or `504 Gateway Timeout` errors observed
- Application content rendered successfully: `Welcome to xFusionCorp Industries!`

📸 **Application loading successfully in the browser via the StaticApp URL, confirming end-to-end traffic flow through the Nginx load balancer to the Apache backend**

![Application loading via StaticApp URL](https://github.com/user-attachments/assets/7ad29568-fd91-4d75-8b72-237f106617a0)

---

## Errors and Resolutions

### Error 1: Nginx Configuration Syntax Failure

**Symptom:**

```
nginx: [emerg] unexpected "}" in /etc/nginx/nginx.conf:83
nginx: configuration file /etc/nginx/nginx.conf test failed
```

**Root Cause:** A misplaced or unbalanced closing brace `}` in the `nginx.conf` file after adding the `upstream` and `server` blocks.

**Resolution:**

- Reopened the file with `sudo vi /etc/nginx/nginx.conf`
- Reviewed block nesting and corrected brace alignment
- Re-ran `sudo nginx -t` to confirm resolution

**Prevention:** When editing Nginx configuration manually with `vi`, validate immediately after saving. Structural errors in `nginx.conf` are common when inserting blocks into an existing context. Using a linter or reviewing indentation before saving reduces error frequency.

---

### Error 2: 502 Bad Gateway (Pre-Resolution)

**Symptom:** Initial Nginx configuration forwarded traffic to the default port `80` on backend servers, where no service was listening.

**Root Cause:** The upstream block was written without explicit port specification, causing Nginx to default to port `80`, while Apache was running on port `5004`.

**Resolution:**

- Verified Apache listening port on `stapp01` using `sudo ss -lntp | grep httpd`
- Confirmed `0.0.0.0:5004` as the active listener
- Updated upstream server entries to include `:5004`
- Reloaded Nginx

> **Key Insight:** Always verify the actual listening port of backend services before writing upstream configuration. Do not assume default ports. Port mismatches are among the most common causes of `502` errors in reverse proxy setups.

---

## Key Decisions

- **Single configuration file:** The task required all changes to be applied exclusively in `/etc/nginx/nginx.conf`. Drop-in files under `/etc/nginx/conf.d/` were deliberately not used to comply with this constraint and to keep the configuration auditable in a single location.
- **Round-robin load balancing:** No explicit algorithm was specified, defaulting to round-robin. This is appropriate for stateless HTTP workloads where all backends are of equal capacity.
- **Proxy headers preserved:** `X-Real-IP`, `X-Forwarded-For`, and `X-Forwarded-Proto` headers were forwarded to backend servers to maintain client context transparency across the proxy boundary.
- **Zero-downtime reload:** `systemctl reload nginx` was used instead of `restart` to avoid dropping in-flight connections during the configuration update.
- **No backend modification:** Apache configuration on all three app servers was left entirely intact, satisfying the core production constraint.

---

## Lessons Learned

- **Validate before reload:** Running `nginx -t` caught a syntax error before it could cause a service disruption. This step should be non-negotiable in any Nginx workflow.
- **Port verification is foundational:** Before writing any upstream block, always verify the actual port a backend service is bound to. The `ss -lntp` command is reliable for this on CentOS/RHEL systems where `netstat` may not be installed.
- **`netstat` may be absent on modern CentOS:** `netstat` was not available on `stapp01` (`sudo: netstat: command not found`). The `ss` command from the `iproute2` package is the modern replacement and should be the default tool for socket inspection.
- **Brace discipline in Nginx config:** When adding `upstream` and `server` blocks inside an existing `http {}` context, maintaining consistent indentation and immediately testing with `nginx -t` prevents hard-to-spot brace imbalance errors.
- **`reload` over `restart` in production:** Nginx's graceful reload preserves active connections. In production, `restart` should only be used when a full process replacement is required (e.g., binary upgrades).

---

## Skills Demonstrated

- Linux system administration on CentOS Stream 9
- Nginx installation, configuration, and service lifecycle management
- Reverse proxy and HTTP load balancer configuration
- Production-safe configuration validation (`nginx -t`) and zero-downtime reload
- Network socket inspection (`ss -lntp`) for port verification
- Multi-host SSH operations from a bastion jump host
- Upstream proxy header forwarding for client transparency
- Troubleshooting and resolution of `502 Bad Gateway` and Nginx syntax errors
- Constraint-aware implementation (no backend modification, single config file)
- High-availability infrastructure validation and end-to-end traffic verification
