# Apache HTTP Service Recovery on Non-Default Port 8087

**Project Type:** Linux Administration
**Category:** Service Troubleshooting and Incident Resolution
**Service:** Apache HTTP Server (`httpd`)
**Target Port:** `8087`
**Environment:** CentOS / RHEL-based Application Server

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment Details](#environment-details)
- [Objectives](#objectives)
- [Tools Used](#tools-used)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: SSH Into the Application Server](#step-1-ssh-into-the-application-server)
  - [Step 2: Initial Diagnosis and Port Conflict Identification](#step-2-initial-diagnosis-and-port-conflict-identification)
  - [Step 3: Check Apache Service Status and Error Logs](#step-3-check-apache-service-status-and-error-logs)
  - [Step 4: Terminate the Conflicting Process](#step-4-terminate-the-conflicting-process)
  - [Step 5: Confirm Port Clearance](#step-5-confirm-port-clearance)
  - [Step 6: Repair Apache Configuration and Validate Syntax](#step-6-repair-apache-configuration-and-validate-syntax)
  - [Step 7: Start and Verify the Apache Service](#step-7-start-and-verify-the-apache-service)
  - [Step 8: Update Firewall Rules for Inbound Access](#step-8-update-firewall-rules-for-inbound-access)
  - [Step 9: Final Validation from the Jump Host](#step-9-final-validation-from-the-jump-host)
- [Validation Checklist](#validation-checklist)
- [Outcome](#outcome)
- [Operational Considerations and Lessons Learned](#operational-considerations-and-lessons-learned)

---

## Overview

This document details the end-to-end incident resolution process for an Apache HTTP Server that became unreachable on a non-default port (`8087`). The monitoring system triggered an alert indicating service unavailability. Root cause analysis revealed a compound failure involving three distinct issues:

1. **Port conflict** - A `sendmail` process (PID 491) was binding to `127.0.0.1:8087`, preventing Apache from starting.
2. **Configuration syntax error** - An invalid directive on line 45 of `/etc/httpd/conf/httpd.conf` caused Apache to exit on startup.
3. **Firewall restriction** - `iptables` rules blocked inbound TCP traffic on port 8087, rendering the service unreachable from the network even after it started.

The resolution followed a structured diagnostic and remediation process: identify the conflict, eliminate it, repair the configuration, restart the service, and verify end-to-end connectivity. The `index.html` file was preserved without modification throughout.

---

## Problem Statement

- Apache HTTP Server is **not reachable on port 8087** as reported by the monitoring system.
- Port `8087` is occupied by a `sendmail` process with **PID 491**, preventing Apache from binding to the port.
- **Syntax error on line 45** of `/etc/httpd/conf/httpd.conf` causes Apache to fail during configuration parsing.
- Access attempts from the jump host fail due to **iptables firewall rules** blocking port 8087.
- **Constraint:** The `index.html` file must remain unmodified at all times.

---

## Environment Details

| Parameter | Value |
|---|---|
| Jump Host | `jumphost` |
| Application Server | `stapp01` |
| Application Server IP | `172.16.238.10` |
| Operating User | `tony` |
| Target Service | Apache HTTP Server (`httpd`) |
| Target Port | `8087` |

---

## Objectives

- Verify Apache service status and inspect error logs for root cause indicators.
- Identify all processes holding port 8087 and resolve the conflict.
- Repair the Apache configuration file and confirm syntax validity.
- Ensure Apache is configured to listen on port 8087.
- Update firewall rules to permit inbound TCP traffic on port 8087.
- Confirm full service restoration using `curl` from the jump host.
- Preserve `index.html` without modification.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `ssh` | Remote access to the application server |
| `systemctl` | Service lifecycle management |
| `netstat` | Network port and process binding inspection |
| `kill` | Process termination |
| `vi` / `sed` | Configuration file editing |
| `httpd -t` | Apache configuration syntax validation |
| `iptables` | Firewall rule management |
| `curl` | End-to-end HTTP connectivity verification |

---

## Step-by-Step Implementation

---

### Step 1: SSH Into the Application Server

Establish a secure shell session from the jump host to the application server (`stapp01`) as user `tony`.

```bash
ssh tony@172.16.238.10
```

**Intent:** All subsequent diagnostic and remediation commands are executed on the application server. SSH is the standard remote access method in this environment.

> **Screenshot:** Initial SSH connection to stapp01 from the jump host, with host key verification and password authentication.

![Step 1 - SSH into App Server](https://github.com/user-attachments/assets/e879d55b-6c3a-473e-beba-5e6e941bd74f)

---

### Step 2: Initial Diagnosis and Port Conflict Identification

Before touching the service or configuration, inspect which process is currently holding port 8087. This is the most critical first step in diagnosing any "address already in use" failure.

```bash
sudo netstat -tunlp | grep 8087
```

**Result:** `sendmail` process with **PID 491** is listening on `127.0.0.1:8087`, directly blocking Apache from binding to the same port.

```
tcp   0   0 127.0.0.1:8087   0.0.0.0:*   LISTEN   491/sendmail: accep
```

> **Operational Note:** Always verify which process owns a conflicting port before killing it. Terminating the wrong process in a production environment can cause cascading failures. In this case, `sendmail` on port 8087 is clearly an anomaly and safe to terminate.

> **Screenshot:** `netstat` output confirming sendmail (PID 491) is binding to port 8087.

![Step 2 - Port Conflict Identified](https://github.com/user-attachments/assets/bab783de-b251-46c6-a048-cf0d3f5459ca)

---

### Step 3: Check Apache Service Status and Error Logs

Inspect the Apache service state via `systemctl` to understand the full scope of the failure before making any changes.

```bash
sudo systemctl status httpd
```

**Result:** The service is in a **failed** state. The status output reveals two distinct failure causes:

- `AH00526: Syntax error on line 45 of /etc/httpd/conf/httpd.conf: Invalid address or port` - Apache is rejecting its own configuration.
- `httpd.service: Failed with result 'exit-code'` - The service exited abnormally.

> **Best Practice:** Always review service logs before modifying configuration files. The `systemctl status` output and journal logs provide targeted diagnostic information that narrows the remediation scope, preventing unnecessary changes.

> **Screenshot:** Apache service status showing the failed state and associated error log output, including the configuration syntax error on line 45.

![Step 3 - Apache Service Status](https://github.com/user-attachments/assets/305eec9d-4f95-4254-a2f1-9db83d63404f)

---

### Step 4: Terminate the Conflicting Process

With the conflict identified (PID 491, `sendmail`), forcefully terminate the process to release port 8087.

```bash
sudo kill -9 491
```

**Result:** The `sendmail` process is terminated and port 8087 is freed for Apache to bind.

> **Risk Note:** Using `kill -9` sends `SIGKILL` and does not allow the process to clean up gracefully. This is appropriate here because the `sendmail` process is occupying a port it should not be using. In environments where `sendmail` is a legitimate mail relay, coordinate with the responsible team before terminating it. Consider also investigating why `sendmail` was misconfigured to use port 8087.

> **Screenshot:** Execution of `sudo kill -9 491` to terminate the sendmail process holding port 8087.

![Step 4 - Kill Conflicting Process](https://github.com/user-attachments/assets/27bd64ab-ec14-46b4-80ad-c986ccd8223f)

---

### Step 5: Confirm Port Clearance

After terminating the conflicting process, verify that port 8087 is no longer occupied before attempting to start Apache.

```bash
sudo netstat -tunlp | grep 8087
sudo kill -9 491
```

**Result:** The initial `netstat` run confirms the port was still occupied, prompting the kill. A repeat `netstat` check after termination confirms the port is clear.

> **Best Practice:** Always confirm port clearance after killing a process. Race conditions or process respawning (e.g., from a watchdog or init system) can re-occupy the port immediately.

> **Screenshot:** Two consecutive `netstat` checks bracketing the `kill` command, confirming the port was held and then released.

![Step 5 - Confirm Port Clearance](https://github.com/user-attachments/assets/f25774e2-ad1e-4a07-b77d-f71121ca4a7d)

---

### Step 6: Repair Apache Configuration and Validate Syntax

Address the second root cause: the syntax error in the Apache configuration file. Edit the file to correct the invalid directive, then validate the full configuration before attempting to restart the service.

```bash
sudo vi /etc/httpd/conf/httpd.conf
```

**Change Applied:** Corrected the invalid address or port directive on line 45. Ensured the `Listen` directive is properly set to port `8087`:

```apache
Listen 8087
```

After saving the file, validate the configuration syntax:

```bash
sudo httpd -t
```

**Result:** `Syntax OK`

> **Best Practice:** Never restart a service after a configuration change without first running a syntax validation. For Apache, `httpd -t` parses the full configuration tree and reports all errors without starting the daemon. This prevents a broken configuration from taking a previously running service offline.

> **Edge Case:** If `httpd.conf` includes other configuration files (via `Include` or `IncludeOptional`), the syntax check validates all of them as well. Ensure all included files are accessible and syntactically correct.

> **Screenshot:** Apache configuration syntax validation returning `Syntax OK` after the error on line 45 was corrected.

![Step 6 - Syntax Validation](https://github.com/user-attachments/assets/55832b85-84bb-44bc-af62-5e5a03c13776)

---

### Step 7: Start and Verify the Apache Service

With the port conflict resolved and the configuration validated, start the Apache service and confirm it reaches a healthy running state.

```bash
sudo systemctl start httpd
sudo systemctl status httpd
```

**Result:**

- Service status: **active (running)**
- Apache is now listening on port 8087 as confirmed in the status output: `Status: "Running, listening on: port 8087"`
- Main PID: `1200 (httpd)`

> **Operational Note:** In this environment, the `httpd` service is not enabled to start on boot (`disabled; vendor preset: disabled`). If persistent availability across reboots is required, run `sudo systemctl enable httpd` to add it to the system startup sequence.

> **Screenshot:** Apache service status confirming active (running) state with port 8087 listening, along with the full systemd journal log showing a clean startup sequence.

![Step 7 - Apache Running](https://github.com/user-attachments/assets/e5dfc3f4-a745-47d0-9e60-20af9e0f84ec)

---

### Step 8: Update Firewall Rules for Inbound Access

Even with the service running, network traffic on port 8087 is blocked by the host firewall (`iptables`). Insert a rule at the top of the INPUT chain to permit inbound TCP connections on port 8087.

```bash
sudo iptables -I INPUT 1 -p tcp --dport 8087 -j ACCEPT
```

**Breakdown of the command:**

- `-I INPUT 1` - Inserts the rule at position 1 (highest priority) in the INPUT chain.
- `-p tcp` - Matches TCP protocol traffic.
- `--dport 8087` - Matches destination port 8087.
- `-j ACCEPT` - Accepts matching packets.

**Result:** Inbound TCP traffic on port 8087 is now permitted, resolving the "not reachable" condition reported by the monitoring system.

> **Security Consideration:** This rule permits traffic from all source addresses. In a production hardened environment, restrict the source IP range using `-s <source_cidr>` to limit exposure. Additionally, these `iptables` rules are not persistent by default across reboots. Use `iptables-save` and `iptables-restore` or a service like `iptables-persistent` to persist rules.

> **Screenshot:** `iptables` command execution to insert the ACCEPT rule for port 8087 at INPUT chain position 1.

![Step 8 - Firewall Rule Added](https://github.com/user-attachments/assets/d88aa054-3921-47f3-ac82-052e4b8387a2)

---

### Step 9: Final Validation from the Jump Host

Perform end-to-end validation by issuing an HTTP request from the jump host to the application server on port 8087. This confirms that the service is reachable over the network and serving content correctly.

```bash
curl http://stapp01:8087
```

**Result:** The CentOS HTTP Server Test Page HTML is returned successfully, confirming:

- Apache is running and bound to port 8087.
- The firewall is permitting inbound traffic on port 8087.
- DNS or hostname resolution for `stapp01` is functioning from the jump host.
- The default `index.html` is being served and has not been modified.

> **Screenshot:** `curl` response from the jump host returning the full CentOS HTTP Server Test Page HTML, confirming complete end-to-end service restoration.

![Step 9 - Final Validation via curl](https://github.com/user-attachments/assets/3efad4a6-b97c-4916-b246-2d64e32c2501)

---

## Validation Checklist

| Check | Status |
|---|---|
| Apache service is active (running) | PASS |
| Apache is listening on port 8087 | PASS |
| Conflicting `sendmail` process (PID 491) terminated | PASS |
| Configuration syntax error on line 45 resolved | PASS |
| `httpd -t` returns `Syntax OK` | PASS |
| `iptables` INPUT rule permits TCP port 8087 | PASS |
| `curl http://stapp01:8087` returns HTTP 200 from jump host | PASS |
| `index.html` remains unmodified | PASS |

---

## Outcome

The Apache HTTP Server was fully restored and made accessible on port 8087. The incident was caused by a combination of three independent failures: a rogue `sendmail` process occupying the target port, an invalid directive in the Apache configuration file, and a missing firewall ACCEPT rule. All three root causes were individually diagnosed, addressed, and validated. The service is now active, reachable from the network, and serving content as expected.

---

## Operational Considerations and Lessons Learned

**Port Conflicts in Multi-Service Environments**
Non-default ports increase the risk of unintentional conflicts when services are misconfigured. Implement port reservation conventions and document all non-standard port assignments in a service registry. Consider using firewall rules or SELinux policy to restrict which processes can bind to specific ports.

**Configuration Validation as a Gate**
Running `httpd -t` (or the equivalent for any service) before every restart should be a mandatory operational practice, especially in CI/CD pipelines and during on-call incident response. A failing validation surfaces problems without disrupting availability of an already-running instance.

**Firewall Rule Persistence**
`iptables` rules applied at runtime are lost on reboot. In production, always persist firewall changes using `iptables-save > /etc/sysconfig/iptables` or equivalent, and verify the rules are loaded on boot.

**Service Boot Enablement**
Services resolved during an incident should be verified for boot-time enablement. A service manually started but not enabled will not survive a system reboot, creating a recurring incident.

**Incident Documentation**
Every production incident should be followed by a concise post-incident review documenting the root causes, timeline, remediation steps, and preventive measures. This document serves as that reference for future on-call engineers.
