# Nginx HTTPS Deployment with Self-Signed TLS on CentOS Stream 9

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure Details](#infrastructure-details)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Step 1: SSH to App Server 1](#step-1-ssh-to-app-server-1)
- [Step 2: Install Nginx](#step-2-install-nginx)
- [Step 3: Enable and Start Nginx](#step-3-enable-and-start-nginx)
- [Step 4: Deploy SSL Certificate and Private Key](#step-4-deploy-ssl-certificate-and-private-key)
- [Step 5: Configure Nginx for HTTPS](#step-5-configure-nginx-for-https)
- [Step 6: Deploy Landing Page](#step-6-deploy-landing-page)
- [Step 7: Validate Configuration and Restart Nginx](#step-7-validate-configuration-and-restart-nginx)
- [Step 8: Validate HTTPS Access from Jump Host](#step-8-validate-https-access-from-jump-host)
- [Key Decisions](#key-decisions)
- [Lessons Learned](#lessons-learned)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This project covers the end-to-end provisioning of a production-ready HTTPS web service on a CentOS Stream 9 application server. The implementation installs and configures **Nginx 1.20.1**, deploys a **self-signed TLS certificate**, serves a static landing page from the Nginx document root, and validates encrypted access from a remote jump host using `curl`.

This is representative of the server hardening and web service bootstrapping work performed during infrastructure onboarding, environment preparation, and secure service deployment workflows.

---

## Problem Statement

Unencrypted HTTP services expose transmitted data to interception and are non-compliant with most production security baselines. The requirement is to:

- Install and configure a production web server on App Server 1
- Enable TLS termination using a pre-issued self-signed certificate
- Serve a verified HTML response over HTTPS port 443
- Confirm HTTPS reachability from a remote host without requiring certificate trust (self-signed context)

---

## Infrastructure Details

| Component | Details |
|---|---|
| Server Name | App Server 1 |
| Hostname | stapp01.stratos.xfusioncorp.com |
| IP Address | 172.16.238.10 |
| OS User | tony |
| Operating System | CentOS Stream 9 |
| Web Server | Nginx 2:1.20.1-26.el9 |
| TLS Certificate | Self-Signed (nautilus.crt / nautilus.key) |
| Certificate Path | /etc/pki/tls/certs/nautilus.crt |
| Private Key Path | /etc/pki/tls/private/nautilus.key |
| Document Root | /usr/share/nginx/html |
| Access Validation Host | jump host (thor@jumphost) |

---

## Architecture Summary

```
[thor@jumphost]
      |
      |  curl -Ik https://172.16.238.10/
      |
      v
[stapp01 - Nginx :443]
      |
      |-- TLS: /etc/pki/tls/certs/nautilus.crt
      |-- Key: /etc/pki/tls/private/nautilus.key
      |-- Root: /usr/share/nginx/html/index.html
```

---

## Prerequisites

- SSH access to `172.16.238.10` as user `tony` with sudo privileges
- Pre-staged certificate file at `/tmp/nautilus.crt` on the app server
- Pre-staged private key file at `/tmp/nautilus.key` on the app server
- Outbound internet access on the app server for package installation (yum repositories)
- `curl` available on the jump host for remote validation

---

## Step 1: SSH to App Server 1

From the jump host, initiate an SSH session to the application server. On first connection, accept the host key fingerprint to add the server to the known hosts list.

```bash
ssh tony@172.16.238.10
```

On first connection, confirm the host key prompt:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

Authenticate with the user password when prompted.

**Operational note:** In production environments, host key verification should be enforced rather than bypassed. For automated pipelines, pre-populate `~/.ssh/known_hosts` using `ssh-keyscan` prior to connection.

> **Screenshot:** SSH session established from jump host to stapp01. The ED25519 fingerprint is accepted and the host is added to the known hosts file.

![SSH login from jump host to stapp01](https://github.com/user-attachments/assets/304d9926-ff38-4586-b15d-21effecc1561)

---

## Step 2: Install Nginx

Install Nginx from the CentOS AppStream repository using the system package manager. The `-y` flag auto-confirms the transaction.

```bash
sudo yum install -y nginx
```

The package manager resolves and installs the following packages:

- `nginx-2:1.20.1-26.el9.x86_64` (main binary)
- `nginx-core-2:1.20.1-26.el9.x86_64` (core modules)
- `nginx-filesystem-2:1.20.1-26.el9.noarch` (directory structure)
- `centos-logos-httpd-90.9-1.el9.noarch` (branding assets)

**Total install size:** 4.3 MB

**Operational note:** In locked-down environments, verify that the AppStream and EPEL repositories are reachable before running the install. Use `yum repolist` to confirm repository availability.

> **Screenshot:** yum resolves dependencies and begins downloading the Nginx packages from AppStream.

![yum install nginx - dependency resolution and download](https://github.com/user-attachments/assets/87826513-204d-415b-80d5-af62bedc0c57)

> **Screenshot:** Transaction completes successfully. All four packages are installed and verified.

![yum install nginx - transaction complete](https://github.com/user-attachments/assets/452cdda5-67ca-4a9c-9510-fbb7f3423977)

---

## Step 3: Enable and Start Nginx

Enable Nginx to start automatically at boot using systemd, then bring the service up immediately.

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Enabling Nginx creates a systemd symlink:

```
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service
  -> /usr/lib/systemd/system/nginx.service
```

**Operational note:** Always enable the service alongside starting it. A service that is started but not enabled will not survive a system reboot, which is a common source of post-maintenance outages.

> **Screenshot:** Nginx is enabled for boot persistence and started. The systemd symlink is created confirming the enable operation.

![Nginx enabled and started via systemctl](https://github.com/user-attachments/assets/0238cb83-3818-4b48-b8fa-d2ffccc5d8d7)

---

## Step 4: Deploy SSL Certificate and Private Key

Create the standard PKI directory structure for TLS assets, then move the pre-staged certificate and key from the staging directory to their production paths.

```bash
sudo mkdir -p /etc/pki/tls/certs /etc/pki/tls/private
sudo mv /tmp/nautilus.crt /etc/pki/tls/certs/
sudo mv /tmp/nautilus.key /etc/pki/tls/private/
```

**Directory structure rationale:**

| Path | Purpose |
|---|---|
| `/etc/pki/tls/certs/` | Standard location for public certificates on RHEL-family systems |
| `/etc/pki/tls/private/` | Restricted directory for private keys |

**Security considerations:**
- The private key directory (`/etc/pki/tls/private/`) is mode `700` by default on CentOS, restricting access to root only
- In production, private key files should be mode `600` and owned by the service account
- Self-signed certificates are appropriate for internal or test services; public-facing production services should use certificates issued by a trusted CA (e.g., Let's Encrypt, DigiCert)
- Avoid staging sensitive key material in world-readable directories like `/tmp` in production workflows; use secure transfer mechanisms instead

> **Screenshot:** Certificate directory structure created. The certificate and key are moved from `/tmp` to their respective PKI paths.

![SSL certificate and key deployed to /etc/pki/tls](https://github.com/user-attachments/assets/d4860fee-388e-4049-82ae-f74ab827139f)

---

## Step 5: Configure Nginx for HTTPS

Edit the main Nginx configuration file to add the TLS-enabled server block.

```bash
sudo vi /etc/nginx/nginx.conf
```

Locate the commented TLS server block section and configure it with the correct certificate paths, hostname, and document root. The resulting server block:

```nginx
# Settings for a TLS enabled server.
server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  stapp01.stratos.xfusioncorp.com;
    root         /usr/share/nginx/html;

    ssl_certificate     "/etc/pki/tls/certs/nautilus.crt";
    ssl_certificate_key "/etc/pki/tls/private/nautilus.key";
    ssl_session_cache   shared:SSL:1m;
    ssl_session_timeout 10m;
    ssl_ciphers         PROFILE=SYSTEM;
    ssl_prefer_server_ciphers on;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```

**Configuration notes:**
- `http2` is enabled alongside SSL for improved performance; HTTP/2 requires TLS
- `ssl_ciphers PROFILE=SYSTEM` defers to the system-wide crypto policy on CentOS Stream 9, which is the recommended approach for maintaining consistent organizational cipher standards
- `ssl_session_cache` reduces TLS handshake overhead for repeat connections
- `ssl_prefer_server_ciphers on` ensures the server's cipher order takes precedence, providing control over negotiated cipher strength

> **Screenshot:** The TLS server block in nginx.conf showing the certificate paths, HTTP/2 configuration, and cipher settings.

![nginx.conf TLS server block configuration](https://github.com/user-attachments/assets/7f7a0a97-6f19-4e3d-8b4c-25775c212748)

---

## Step 6: Deploy Landing Page

Write a minimal HTML landing page to the Nginx document root. The `tee` command writes the file while also printing its contents to stdout for inline confirmation.

```bash
echo "Welcome!" | sudo tee /usr/share/nginx/html/index.html
```

**Expected output:**

```
Welcome!
```

**Operational note:** In production, the document root would be populated by a deployment pipeline (CI/CD) rather than inline shell commands. This step establishes a known-good content baseline for HTTPS validation purposes.

> **Screenshot:** Landing page written to `/usr/share/nginx/html/index.html`. The `tee` output confirms the write succeeded.

![index.html deployed to Nginx document root](https://github.com/user-attachments/assets/20cd2bd9-d532-4aff-9cee-d142503add67)

---

## Step 7: Validate Configuration and Restart Nginx

Before restarting the service, validate the Nginx configuration syntax to catch any errors without causing a service disruption.

```bash
sudo nginx -t
```

**Expected output:**

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Once validation passes, restart Nginx to apply the new TLS configuration:

```bash
sudo systemctl restart nginx
```

**Best practice:** Always run `nginx -t` before restarting in production environments. A failed configuration test on a running service means the restart would fail, causing a service outage. The `-t` flag tests the configuration without touching the running process.

> **Screenshot:** `nginx -t` returns syntax OK and test successful, confirming the configuration is valid before the restart is applied.

![nginx -t configuration test output](https://github.com/user-attachments/assets/8260e090-27be-4e7d-a344-18258052881b)

> **Screenshot:** `sudo systemctl restart nginx` completes without error. The service is now serving on port 443 with TLS.

![Nginx restart successful](https://github.com/user-attachments/assets/c22cedd0-3851-40c8-afd4-989b424278e5)

---

## Step 8: Validate HTTPS Access from Jump Host

Exit the app server session and return to the jump host to perform remote HTTPS validation.

```bash
exit
```

From the jump host, issue a `curl` request to the app server IP over HTTPS. The `-I` flag fetches headers only, and `-k` bypasses certificate trust validation (required for self-signed certificates).

```bash
curl -Ik https://172.16.238.10/
```

**Expected response headers:**

```
HTTP/2 200
server: nginx/1.20.1
date: Sat, 21 Feb 2026 03:53:44 GMT
content-type: text/html
content-length: 9
last-modified: Sat, 21 Feb 2026 03:48:11 GMT
etag: "69992afb-9"
accept-ranges: bytes
```

**Validation checklist:**

- `HTTP/2 200` confirms: TLS handshake succeeded, HTTP/2 is negotiated, and the server returned a successful response
- `server: nginx/1.20.1` confirms Nginx is serving the request
- `content-length: 9` matches the 9-byte payload of `Welcome!\n`
- The `date` and `last-modified` timestamps confirm the landing page was freshly deployed

**Operational note:** The `-k` flag is acceptable for self-signed certificate validation in controlled test environments. In production with a CA-signed certificate, omit `-k` to enforce full certificate chain validation.

> **Screenshot:** `curl -Ik` from the jump host returns HTTP/2 200 with Nginx response headers. The TLS handshake completes successfully against the self-signed certificate and the server returns the expected response.

![curl HTTPS validation from jump host showing HTTP/2 200](https://github.com/user-attachments/assets/172cf54f-d96a-4767-9a85-8e8fd0ab8a30)

---

## Key Decisions

**Self-signed certificate placement in `/etc/pki/tls/`:** The standard PKI directory hierarchy on RHEL-family systems separates public certificates from private keys with appropriate filesystem permissions. Following this convention ensures compatibility with system-level crypto tooling and auditing expectations.

**Enabling HTTP/2 (`ssl http2`):** HTTP/2 multiplexing reduces connection overhead compared to HTTP/1.1 and is enabled by default in modern Nginx TLS configurations. It requires TLS and is activated with the `http2` parameter on the listen directive.

**Using `PROFILE=SYSTEM` for cipher configuration:** Deferring to the system-wide crypto policy rather than hardcoding a custom cipher suite ensures that the server's TLS posture stays consistent with the organization's policy and benefits from OS-level policy updates without requiring Nginx configuration changes.

**Config validation before restart (`nginx -t`):** Running the configuration test before applying a restart is a mandatory operational gate. A misconfigured Nginx that fails to start would take the service offline with no automatic rollback mechanism.

---

## Lessons Learned

- **Staging sensitive assets in `/tmp` is a risk in shared environments.** `/tmp` is world-readable by default on Linux. In production workflows, private key material should be transferred directly to its target path using secure channels (e.g., secrets management systems, ansible vault, or SFTP with restricted permissions).

- **`nginx -t` is a zero-risk pre-flight check.** It parses and validates the full configuration without touching the running process. It should be a mandatory step in any Nginx configuration change workflow, including automated deployment pipelines.

- **HTTP/2 requires TLS.** Attempting to enable HTTP/2 over plain HTTP will fail. HTTP/2 is only negotiated during the TLS handshake via ALPN. This is why HTTP/2 configuration is co-located with SSL directives in Nginx.

- **Self-signed certificates require `-k` with curl.** The curl client enforces certificate chain validation by default. Self-signed certificates have no chain to a trusted CA, so validation fails unless bypassed with `-k`. This is expected and correct behavior for test environments.

- **Content-length confirms end-to-end correctness.** A `content-length: 9` response for the string `Welcome!\n` (8 characters + newline = 9 bytes) confirms that the file was written correctly, the document root is correctly mapped, and the full request/response cycle is functioning as expected.

---

## Skills Demonstrated

- **Linux Systems Administration:** SSH, sudo privilege escalation, `yum` package management, filesystem operations
- **Nginx Web Server:** Installation, TLS server block configuration, HTTP/2 enablement, configuration validation
- **TLS/SSL Operations:** Certificate and private key deployment, PKI directory structure, self-signed certificate usage
- **Systemd Service Management:** Enable for boot persistence, start, restart, service lifecycle control
- **Remote Infrastructure Validation:** `curl` header inspection, HTTPS connectivity testing from a remote host
- **Operational Discipline:** Pre-restart configuration testing, structured step-by-step execution, inline validation at each phase
