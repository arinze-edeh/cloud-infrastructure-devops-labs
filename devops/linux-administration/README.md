# Linux Administration

Production-focused Linux administration work covering service management, web server deployment, security hardening, database operations, automation, and infrastructure troubleshooting. All tasks were executed against live multi-server environments within the Stratos Datacenter, accessed via a bastion jump host architecture.

Each project reflects patterns and constraints found in real operations: non-standard ports, no-restart requirements, multi-node consistency enforcement, and audit-ready configuration changes.

---

## Directory Structure

```
linux-administration/
├── access-and-authentication/
├── apache-deployment-and-custom-port-binding/
├── apache-multi-site-deployment/
├── apache-service-troubleshooting/
├── bash-scripting-automation/
├── cron-service-validation/
├── java-webapp-tomcat-systemd-deployment/
├── mariadb-service-recovery/
├── multi-node-apache-service-troubleshooting/
├── nginx-load-balancer-setup/
├── php81-nginx-unixsocket-deployment/
├── postgresql-infrastructure/
├── script-execution-permissions/
├── secure-nginx-https-deployment/
├── selinux-configuration/
├── ssh-hardening/
├── user-management/
└── README.md
```

---

## Project Summaries

### [Access and Authentication](./access-and-authentication)

**Quick Summary:** Configured passwordless SSH key-based authentication from a centralized jump host to all three Nautilus application servers, enabling unattended automation without credential exposure.

**Purpose:** Password-based SSH in automated workflows breaks CI/CD pipelines and exposes credentials in logs. RSA key-based auth eliminates both risks while maintaining a single auditable trust anchor.

**Approach:** Generated a 3072-bit RSA key pair on the jump host, distributed the public key via `ssh-copy-id` to each app server, and validated with non-interactive `hostname` execution across all nodes.

**Outcome:** All three servers accept passwordless connections from the jump host. Infrastructure is ready for unattended automation and scheduled maintenance scripts.

---

### [Apache Deployment and Custom Port Binding](./apache-deployment-and-custom-port-binding)

**Quick Summary:** Installed Apache2 inside a running Docker container on a non-default port (8083), working around container-specific constraints such as `policy-rc.d` blocking auto-start.

**Purpose:** Completing an incomplete handover where Apache needed to be installed, configured on port 8083, and confirmed running without stopping or restarting the container.

**Approach:** Entered the running `kkloud` container via `docker exec`, installed Apache2 via `apt`, modified both `ports.conf` and `000-default.conf` using `sed`, and validated with `curl` before exiting to confirm container uptime.

**Key decision:** Updated both config files atomically. A mismatch between `ports.conf` and the VirtualHost directive is the leading cause of Apache startup failures in this environment.

**Outcome:** Apache2 serving on port 8083 inside the container; container remained in `Up` state throughout.

---

### [Apache Multi-Site Deployment](./apache-multi-site-deployment)

**Quick Summary:** Deployed two static websites (`ecommerce` and `demo`) to an Apache server on CentOS Stream 9 listening on port 8085, transferred via SCP from the jump host.

**Purpose:** Migrating pre-built site assets from a jump host to a production app server without disrupting existing services, served on a non-standard port.

**Approach:** Staged assets to `/tmp/` via SCP before moving to `/var/www/html/`, modified the `Listen` directive via `sed`, set recursive `apache:apache` ownership, enabled the service at boot, and confirmed both sites with `curl`.

**Outcome:** Both sites accessible at `http://localhost:8085/ecommerce/` and `/demo/`. Service persists across reboots.

---

### [Apache Service Troubleshooting](./apache-service-troubleshooting)

**Quick Summary:** Diagnosed and resolved a compound Apache failure on port 8087 involving a `sendmail` port conflict, a config syntax error on line 45, and a blocking `iptables` rule.

**Purpose:** Restore an Apache service reported unavailable by monitoring, without modifying the existing `index.html`.

**Approach:** Used `netstat` to identify the conflicting process (PID 491), killed it, corrected the `httpd.conf` syntax error, validated with `httpd -t`, started the service, then inserted an `iptables` ACCEPT rule at INPUT position 1 before confirming end-to-end access via `curl` from the jump host.

**Key decisions:** Validated config before every restart; used `iptables -I INPUT 1` to guarantee rule priority without disrupting existing chain order.

**Outcome:** All three root causes resolved; `curl http://stapp01:8087` returns the expected page from the jump host.

---

### [Bash Scripting and Automation](./bash-scripting-automation)

**Quick Summary:** Built a production-style website backup script that archives a static site to a ZIP file and securely transfers it to a remote backup server using passwordless SCP.

**Purpose:** Replace unreliable manual backups with a privilege-free, cron-compatible automation script that transfers backups without storing credentials.

**Approach:** Configured RSA key-based auth from the app server to the backup server, created `/scripts/official_backup.sh` using `zip -r` and `scp`, scoped directory ownership to the running user, and verified both local and remote archive integrity after execution.

**Outcome:** Script executes without `sudo`, is cron-ready, and transfers successfully to `/backup` on the backup server. Includes documented patterns for timestamped retention and error handling.

---

### [Cron Service Validation](./cron-service-validation)

**Quick Summary:** Deployed and verified `cronie` across all three application servers, adding a root-level cron job to validate scheduler function before committing production workloads.

**Purpose:** Establish a verified cron baseline across the fleet before relying on scheduled automation in production.

**Approach:** Installed `cronie` via `yum`, started and enabled `crond` on each server, added `*/5 * * * * echo hello > /tmp/cron_text` to the root crontab via `crontab -e -u root`, and confirmed registration with `crontab -l -u root`.

**Outcome:** Consistent cron configuration confirmed on stapp01, stapp02, and stapp03. Service enabled at boot on all nodes.

---

### [Java Web App Tomcat Deployment](./java-webapp-tomcat-systemd-deployment)

**Quick Summary:** Deployed a Java WAR application (`ROOT.war`) on Apache Tomcat 9 on CentOS Stream 9, reconfigured to serve on port 3000 at the base URL.

**Purpose:** Stand up a fresh Tomcat environment and deploy an application to validate the environment before pipeline automation.

**Approach:** Installed OpenJDK 11 and Tomcat 9.0.87 via `yum`, edited `server.xml` to change the HTTP connector from 8080 to 3000, transferred `ROOT.war` via SCP, removed the default ROOT directory to prevent deployment conflicts, set `tomcat:tomcat` ownership, and restarted the service.

**Outcome:** Application responding at `http://stapp02:3000/` with "Welcome to xFusionCorp Industries!" confirmed via `curl`.

---

### [MariaDB Service Recovery](./mariadb-service-recovery)

**Quick Summary:** Restored a failed MariaDB instance by correcting `/var/lib/mysql` directory ownership, which had drifted from `mysql:mysql` and blocked daemon startup.

**Purpose:** Recover database availability after a full application-layer connectivity loss caused by a permission issue on the data directory.

**Approach:** SSHed to the database server via the jump host, ran `ls -ld /var/lib/mysql` to confirm ownership drift, applied `sudo chown -R mysql:mysql /var/lib/mysql`, started and enabled `mariadb` via `systemctl`, and confirmed the `"Taking your SQL requests now..."` readiness status.

**Outcome:** MariaDB running, enabled at boot, and accepting connections. Root cause documented for upstream remediation.

---

### [Multi-Node Apache Service Troubleshooting](./multi-node-apache-service-troubleshooting)

**Quick Summary:** Resolved an `httpd` port conflict caused by a `sendmail` process occupying port 8086 on stapp01, then enforced service consistency and boot persistence across all three application nodes.

**Purpose:** Restore a monitoring-flagged Apache outage and eliminate configuration drift across the application tier.

**Approach:** Inspected the failed service with `systemctl status`, identified the conflicting process via `ss -tulnp`, stopped `sendmail`, started `httpd`, verified the port binding, ran `systemctl enable` on all three nodes, and confirmed a uniform post-incident state.

**Outcome:** All nodes running `httpd` on port 8086, enabled at boot, with port binding confirmed via `ss`.

---

### [Nginx Load Balancer Setup](./nginx-load-balancer-setup)

**Quick Summary:** Configured Nginx on a dedicated load balancer node to distribute HTTP traffic across three Apache backend servers running on port 5004, without modifying any backend configuration.

**Purpose:** Address production traffic degradation by introducing a layer-7 load balancer in front of the existing application tier.

**Approach:** Installed Nginx on `stlb01`, verified Apache health on all backends, added an `upstream` block and `server` block directly to `/etc/nginx/nginx.conf` (as required by the constraint), ran `nginx -t` before reloading, and validated via the StaticApp URL.

**Key decisions:** Used `proxy_set_header` directives to forward client IP and protocol context to backends; chose `nginx -t` as a mandatory gate before every reload to catch the brace imbalance error encountered during configuration.

**Outcome:** Traffic successfully routed to all three backends. Application loads correctly through the load balancer.

---

### [PHP 8.1 Nginx Unix Socket Deployment](./php81-nginx-unixsocket-deployment)

**Quick Summary:** Deployed a PHP 8.1 application on Nginx using a Unix domain socket for PHP-FPM communication, served on port 8096 on CentOS Stream 9 with Remi repository sourcing.

**Purpose:** Install a non-default PHP version not available in AppStream and integrate it with Nginx via a Unix socket for reduced latency compared to TCP.

**Approach:** Added the EL9 Remi repository, reset and enabled the `php:remi-8.1` module stream, installed Nginx and PHP-FPM, configured the pool to use `/var/run/php-fpm/default.sock` with `nginx` ownership, created the socket directory, wrote the Nginx virtual host config, enabled both services with `--now`, and validated with `curl -I` confirming `X-Powered-By: PHP/8.1.34`.

**Outcome:** Application serving at `http://stapp03:8096/index.php`, confirmed from the jump host.

---

### [PostgreSQL Infrastructure](./postgresql-infrastructure)

**Quick Summary:** Provisioned a PostgreSQL role and dedicated database with full privileges on a live database server, satisfying a no-restart constraint throughout.

**Purpose:** Onboard a new application with isolated database credentials and scoped access without disrupting existing database connections.

**Approach:** SSHed to `stdb01` via the jump host, escalated to the `postgres` OS user via `sudo -i`, ran all DDL non-interactively via `psql -c`, verified role and database creation with `\du` and `\l`, and corrected a password case mismatch via `ALTER USER` before handoff.

**Key decision:** Used `psql -c` throughout rather than entering the interactive shell for auditability and reduced error surface.

**Outcome:** Role `kodekloud_joy` and database `kodekloud_db4` provisioned with correct privileges. Service never restarted.

---

### [Script Execution Permissions](./script-execution-permissions)

**Quick Summary:** Remediated a fully permission-stripped shell script (`----------`) on a production app server, restoring execute and read access for all users via `chmod a+rx`.

**Purpose:** A pre-deployed script was non-executable by any user or process, blocking dependent workflows.

**Approach:** SSHed to stapp01 via jump host, confirmed the baseline with `ls -l`, applied `sudo chmod a+rx /tmp/xfusioncorp.sh`, and verified the resulting `-r-xr-xr-x` permission string.

**Key decision:** Used `a+rx` (symbolic mode) rather than an octal equivalent to avoid overwriting any bits not under review.

**Outcome:** Script universally readable and executable, root ownership preserved.

---

### [Secure Nginx HTTPS Deployment](./secure-nginx-https-deployment)

**Quick Summary:** Provisioned an Nginx HTTPS server on CentOS Stream 9 using a self-signed TLS certificate, with HTTP/2 enabled and validated remotely via `curl -Ik`.

**Purpose:** Replace an unencrypted HTTP baseline with a TLS-terminating web server meeting production security posture requirements.

**Approach:** Installed Nginx, deployed the certificate and key to standard PKI paths (`/etc/pki/tls/`), configured the TLS server block in `nginx.conf` with `ssl http2`, ran `nginx -t` before restarting, and confirmed `HTTP/2 200` with `X-Powered-By` headers from the jump host.

**Outcome:** HTTPS serving on port 443 with HTTP/2 and system-managed cipher policy (`PROFILE=SYSTEM`).

---

### [SELinux Configuration](./selinux-configuration)

**Quick Summary:** Permanently disabled SELinux on a CentOS Stream 9 app server for a security audit baseline, without rebooting the server.

**Purpose:** Establish a clean policy-free baseline for the audit's initial testing phase, deferring the reboot to the next maintenance window.

**Approach:** Installed `selinux-policy` and `policycoreutils`, set `SELINUX=disabled` in `/etc/selinux/config`, and confirmed `sestatus` output reflects the configuration change.

**Outcome:** SELinux permanently disabled at next boot. Change documented and ready for rollback via `SELINUX=enforcing` plus `.autorelabel`.

---

### [SSH Hardening](./ssh-hardening)

**Quick Summary:** Disabled direct SSH root login across all three application servers by setting `PermitRootLogin no` in `sshd_config` and verifying the runtime state with `sshd -T`.

**Purpose:** Enforce named-user accountability for all SSH sessions and eliminate a high-value brute-force target per CIS Benchmark and PCI-DSS requirements.

**Approach:** Connected to each server via the jump host using named user accounts, escalated via `sudo -i`, edited `sshd_config`, restarted `sshd`, and confirmed the effective value with `sshd -T | grep permitrootlogin` rather than relying on file inspection alone.

**Outcome:** `permitrootlogin no` confirmed at the daemon level on stapp01, stapp02, and stapp03.

---

### [User Management](./user-management)

**Quick Summary:** Covers two user lifecycle patterns: provisioning a non-interactive service account and creating a time-bound developer account with enforced expiry.

**Purpose:** Reduce attack surface from service identities and eliminate orphaned contractor accounts through native OS-level controls.

**Approach:**
- Service account: `useradd -s /sbin/nologin kareem` with `su` rejection confirmed post-creation.
- Temporary access: `useradd -e 2026-12-07 mariyam` with expiry verified via `chage -l`.

**Outcome:** Both accounts are provisioned with OS-enforced constraints requiring no external tooling or manual review cycles for enforcement.

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Web Servers | Apache2 (Ubuntu/Debian), httpd (CentOS/RHEL), Nginx, Apache Tomcat 9 |
| Languages and Runtimes | PHP-FPM 8.1 (Remi), OpenJDK 11, Bash |
| Databases | MariaDB 10.5, PostgreSQL |
| Containers | Docker (`exec`, `ps`, `start`) |
| Service Management | systemd, systemctl, cronie/crond |
| Networking and Security | SSH, SCP, `ssh-keygen`, `ssh-copy-id`, iptables, Nginx upstream/proxy |
| Diagnostics | `ss`, `netstat`, `curl`, `sshd -T`, `httpd -t`, `nginx -t`, `sestatus` |
| Configuration | `sed`, `vi`, `chmod`, `chown`, `useradd`, `chage`, `psql` |
| OS Platforms | CentOS Stream 9, Ubuntu 18.04 (containerized) |

---

## Key Skills Demonstrated

- **Incident Response:** Compound failure diagnosis and resolution across port conflicts, syntax errors, permission drift, and firewall rules.
- **Security Hardening:** SSH root login restriction, SELinux policy management, TLS deployment, and service account lockdown following CIS Benchmark patterns.
- **Service Lifecycle Management:** Correct use of `enable --now`, boot persistence enforcement, and zero-downtime config reloads.
- **Multi-Node Operations:** Enforcing consistent configuration state across three-node application tiers with post-remediation sweep validation.
- **Production Constraints:** Working within no-restart requirements, no-backend-modification constraints, and live container environments.
- **Automation Readiness:** Privilege-free scripts, passwordless SSH, and cron-compatible design patterns.

---

## How to Navigate

Each subdirectory is self-contained with its own `README.md` covering environment details, step-by-step implementation, key decisions, and validation output.

**Recommended reading paths:**

- **Security focus:** `ssh-hardening` → `selinux-configuration` → `secure-nginx-https-deployment` → `access-and-authentication`
- **Web server operations:** `apache-service-troubleshooting` → `multi-node-apache-service-troubleshooting` → `nginx-load-balancer-setup` → `php81-nginx-unixsocket-deployment`
- **Automation and scripting:** `bash-scripting-automation` → `cron-service-validation` → `access-and-authentication`
- **Database and infrastructure:** `mariadb-service-recovery` → `postgresql-infrastructure` → `java-webapp-tomcat-systemd-deployment`

All projects use the same Stratos Datacenter infrastructure (jump host + three application servers + dedicated DB and backup servers), so environment context carries across documents.




