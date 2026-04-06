# Hardening SSH Access: Disabling Root Login Across Production App Servers

> **Security Control:** SSH Root Login Restriction
> **Scope:** Nautilus Project, Stratos Datacenter
> **Servers:** stapp01, stapp02, stapp03
> **Service:** OpenSSH (`sshd`)
> **Privilege Model:** Non-root SSH + `sudo` escalation

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Environment](#environment)
- [Implementation](#implementation)
  - [Step 1: Connect to the Jump Host](#step-1-connect-to-the-jump-host)
  - [Step 2: SSH into the Target App Server](#step-2-ssh-into-the-target-app-server)
  - [Step 3: Escalate Privileges via sudo](#step-3-escalate-privileges-via-sudo)
  - [Step 4: Edit the SSH Daemon Configuration](#step-4-edit-the-ssh-daemon-configuration)
  - [Step 5: Restart the SSH Service](#step-5-restart-the-ssh-service)
  - [Step 6: Verify the Configuration](#step-6-verify-the-configuration)
  - [Step 7: Repeat Across All App Servers](#step-7-repeat-across-all-app-servers)
- [Validation Summary](#validation-summary)
- [Security Best Practices](#security-best-practices)
- [Operational Considerations and Edge Cases](#operational-considerations-and-edge-cases)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This document details the end-to-end process for disabling direct SSH root login across all Nautilus application servers in the Stratos Datacenter. This hardening measure enforces the principle of least privilege by requiring all administrative access to pass through a named, auditable user account before escalating via `sudo`. The change is applied identically to **stapp01**, **stapp02**, and **stapp03** and verified at the daemon level on each host.

---

## Problem Statement

By default or through misconfiguration, SSH daemons may permit direct root login. This creates significant security exposure:

- **Brute-force risk:** Root is a known target. A compromised root credential gives unrestricted system access immediately.
- **Audit gap:** Direct root sessions cannot be attributed to an individual operator. Actions taken as root leave no named user trail.
- **Compliance failure:** Most enterprise security frameworks (CIS Benchmarks, NIST SP 800-53, PCI DSS) explicitly require `PermitRootLogin no` or equivalent controls.

**Solution:** Set `PermitRootLogin no` in `/etc/ssh/sshd_config` on every app server, restart the SSH daemon, and verify the active runtime configuration.

---

## Prerequisites

- SSH access to the jump host (`jump_host.stratos.xfusioncorp.com`)
- Named user credentials for each app server (`tony` on stapp01, `steve` on stapp02, `banner` on stapp03)
- `sudo` privileges configured for each named user on their respective host
- Network connectivity from the jump host to the internal app server subnet (`172.17.0.0/24`)

---

## Environment

| Component | Value |
|---|---|
| Jump Host | `jump_host.stratos.xfusioncorp.com` (172.16.238.3) |
| App Server 01 | `stapp01.stratos.xfusioncorp.com` (172.17.0.7) |
| App Server 02 | `stapp02.stratos.xfusioncorp.com` (172.17.0.8) |
| App Server 03 | `stapp03.stratos.xfusioncorp.com` (172.17.0.6) |
| SSH Config File | `/etc/ssh/sshd_config` |
| SSH Service | `sshd` (managed via `systemctl`) |
| Verification Command | `sshd -T \| grep permitrootlogin` |

---

## Implementation

### Step 1: Connect to the Jump Host

From the local workstation, initiate an SSH session to the jump host. Accept the host key fingerprint on first connection and authenticate with the provided credentials.

```bash
ssh thor@jump_host.stratos.xfusioncorp.com
```

**What to expect:** On first connection, SSH will present the ED25519 host key fingerprint for verification. Confirm with `yes` to add the host to `~/.ssh/known_hosts`. Subsequent connections will not prompt again.

> **Screenshot: Jump host login and initial SSH connection**

![Jump host login](https://github.com/user-attachments/assets/2762f39f-7c18-4d8c-aa31-6240f988eaed)

---

### Step 2: SSH into the Target App Server

From the jump host, SSH into the target application server using its non-root user account. The first connection will again prompt for host key verification.

```bash
ssh tony@stapp01.stratos.xfusioncorp.com
```

**Why a non-root user:** The entire purpose of this exercise is to demonstrate that administrative access can and should be performed through named users. SSHing directly as root would contradict the security control being implemented.

> **Screenshot: SSH session established to stapp01 as user tony**

![SSH into stapp01](https://github.com/user-attachments/assets/c81c1b0e-aa68-43b2-972e-1cfe9ae6bb5c)

---

### Step 3: Escalate Privileges via sudo

Once logged in as the named user, escalate to a root shell using `sudo -i`. This simulates the correct privileged access pattern: authenticate as a named user, then elevate only when required.

```bash
sudo -i
```

**What to expect:** `sudo` will present the standard privilege warning on first use in the session and prompt for the user's own password (not the root password). A successful escalation drops you into a root shell (`[root@stapp01 ~]#`).

> **Screenshot: Privilege escalation to root shell on stapp01**

![sudo escalation on stapp01](https://github.com/user-attachments/assets/7303bde5-2fe5-4d20-98f1-f669aeb14da2)

![Root shell confirmed on stapp01](https://github.com/user-attachments/assets/706dd8cd-f0cc-4743-9c5b-9e83153a7439)

---

### Step 4: Edit the SSH Daemon Configuration

Open the main SSH daemon configuration file using `vi` (or your preferred editor). Locate the `PermitRootLogin` directive and set its value to `no`.

```bash
vi /etc/ssh/sshd_config
```

**Change to make:**

Locate the line (it may be commented by default):

```
#PermitRootLogin yes
```

Uncomment it and change the value:

```
PermitRootLogin no
```

Save and exit the file (`:wq` in `vi`).

**Important notes:**
- The directive is **case-insensitive** at the key level, but use the conventional casing for clarity.
- The file also contains an `Include /etc/ssh/sshd_config.d/*.conf` directive. If drop-in config files in that directory override `PermitRootLogin`, they will take precedence. Verify no conflicting directives exist in that directory.
- Do **not** modify any other directives unless explicitly required. Unintended changes to `sshd_config` can lock out all SSH access.

> **Screenshot: sshd_config open in vi with PermitRootLogin set to no (INSERT mode)**

![sshd_config edited - insert mode](https://github.com/user-attachments/assets/5f94446f-2de4-4bc0-84d1-45773eaced14)

> **Screenshot: sshd_config showing PermitRootLogin no saved**

![sshd_config with PermitRootLogin no](https://github.com/user-attachments/assets/986f2b6c-4efd-4166-a38c-35f09d0fc897)

![Full sshd_config context view](https://github.com/user-attachments/assets/1746211a-dc77-4765-bed9-6a023843431f)

---

### Step 5: Restart the SSH Service

Apply the configuration change by restarting the `sshd` service. `systemctl restart` performs a full stop-start cycle, ensuring the daemon reloads the configuration from disk.

```bash
systemctl restart sshd
```

**Risk awareness:**
- `systemctl restart sshd` will **not** disconnect active SSH sessions on modern systemd-managed systems. Existing connections are maintained by the socket.
- However, if the configuration contains a syntax error, the daemon may fail to start, potentially locking out new connections. Always validate the config before restarting in production (`sshd -t` performs a dry-run syntax check).
- As a precaution, maintain your current session open and test a new connection in a separate terminal before closing your working session.

> **Screenshot: sshd service restarted and PermitRootLogin verified via sshd -T**

![systemctl restart sshd and verification on stapp01](https://github.com/user-attachments/assets/81fab739-cbc9-4b76-b70b-ec7bb10acd72)

---

### Step 6: Verify the Configuration

Query the SSH daemon's active runtime configuration to confirm the change is in effect. `sshd -T` dumps the full effective configuration as the daemon sees it, including all defaults and overrides.

```bash
sshd -T | grep permitrootlogin
```

**Expected output:**

```
permitrootlogin no
```

This confirms the daemon is actively enforcing the restriction, not just that the file was edited. The `sshd -T` approach is more reliable than reading the config file directly because it accounts for `Include` overrides and compiled-in defaults.

> **Screenshot: Runtime verification confirming permitrootlogin no on stapp01**

![Verification output stapp01](https://github.com/user-attachments/assets/28adb7cc-f0d6-4c62-9d85-38082c86fa7a)

> **Screenshot: Full session view showing all steps completed on stapp01**

![Full stapp01 session](https://github.com/user-attachments/assets/b9d12071-30ba-4288-acba-31ff55f8b844)

---

### Step 7: Repeat Across All App Servers

The identical procedure is applied to **stapp02** and **stapp03**. Each server has its own named user account. SSH into each server from either the jump host or from stapp01, escalate via `sudo`, edit `sshd_config`, restart the service, and verify.

**stapp02 (user: steve):**

```bash
ssh steve@stapp02.stratos.xfusioncorp.com
sudo -i
vi /etc/ssh/sshd_config        # Set PermitRootLogin no
systemctl restart sshd
sshd -T | grep permitrootlogin
```

> **Screenshot: stapp02 session: sudo escalation and sshd_config edit**

![stapp02 session and sshd_config edit](https://github.com/user-attachments/assets/5a1f872b-7675-4043-b59a-420ceed13f12)

> **Screenshot: stapp02 verification confirming permitrootlogin no**

![stapp02 full session and verification](https://github.com/user-attachments/assets/28adb7cc-f0d6-4c62-9d85-38082c86fa7a)

**stapp03 (user: banner):**

```bash
ssh banner@stapp03.stratos.xfusioncorp.com
sudo -i
vi /etc/ssh/sshd_config        # Set PermitRootLogin no
systemctl restart sshd
sshd -T | grep permitrootlogin
```

> **Screenshot: stapp02 and stapp03 session continuity: both servers hardened and verified**

![stapp02 to stapp03 session flow](https://github.com/user-attachments/assets/b9d12071-30ba-4288-acba-31ff55f8b844)

> **Note on stapp02 password issue:** During the stapp02 session, an initial authentication failure was observed (`Permission denied, please try again`). This was resolved by verifying the user's password status with `sudo passwd -S steve`, which confirmed the account was active (`PS` status, SHA512 crypt). The second authentication attempt succeeded. This is a known edge case when user passwords have not been set or synced in a new environment.

---

## Validation Summary

| Server | User | `PermitRootLogin` Value | Service Restarted | Verified via `sshd -T` |
|---|---|---|---|---|
| stapp01 | tony | `no` | Yes | Yes |
| stapp02 | steve | `no` | Yes | Yes |
| stapp03 | banner | `no` | Yes | Yes |

All three app servers have been successfully hardened. Direct SSH root login is disabled at the daemon level across the entire Nautilus application tier.

---

## Security Best Practices

- **Disable direct root SSH access** on all internet-facing and internal servers. `PermitRootLogin no` is a baseline requirement in CIS Level 1, DISA STIG, and most enterprise security policies.
- **Enforce named user authentication.** Every SSH session should be attributable to an individual. This supports incident investigation, change tracking, and compliance audits.
- **Use `sudo` for administrative elevation.** Sudo logs all commands to syslog (`/var/log/secure` or `/var/log/auth.log`), providing a full audit trail of privileged actions.
- **Validate config before restarting SSH.** Use `sshd -t` to perform a dry-run syntax check and catch errors before they can cause a service disruption.
- **Verify with `sshd -T`, not just the config file.** The runtime dump reflects all effective settings including `Include` overrides, compiled defaults, and drop-in configs.
- **Regularly audit SSH configurations.** Use automation (Ansible, Chef, Puppet, or a compliance scanner) to detect configuration drift and ensure `PermitRootLogin no` remains enforced after system updates or manual changes.
- **Restrict SSH access further where possible.** Consider combining this control with `AllowUsers`, `AllowGroups`, or `Match` blocks to limit SSH access to only the users and hosts that require it.

---

## Operational Considerations and Edge Cases

- **Include directory overrides:** The default `sshd_config` on many modern Linux distributions includes `Include /etc/ssh/sshd_config.d/*.conf`. A drop-in file in that directory can override `PermitRootLogin`. Always check this directory and use `sshd -T` (not just `grep` on the file) to confirm the effective value.
- **Active session safety:** Restarting `sshd` via `systemctl` on systemd-based systems preserves active sessions through socket activation. However, do not close your current session before verifying a new connection can be established.
- **Authentication failures:** If a named user cannot authenticate, check the account's password status with `passwd -S <username>` before assuming a network or SSH configuration issue. Locked or expired accounts (`L` or `E` status) will fail regardless of SSH configuration.
- **SELinux and firewall considerations:** On SELinux-enabled systems, changing the SSH port requires an additional `semanage port` step (documented in the default `sshd_config` comments). This does not apply to `PermitRootLogin` changes.
- **Rollback procedure:** To re-enable root login (for example, in a break-glass emergency scenario), set `PermitRootLogin yes` (or `PermitRootLogin prohibit-password` to allow key-based root auth only), then run `systemctl restart sshd`.

---

## Skills Demonstrated

- Linux server hardening and SSH security configuration
- Multi-hop SSH access via jump host (bastion host pattern)
- Privilege escalation using `sudo` with named user accountability
- Editing system-level daemon configuration files
- Service lifecycle management with `systemctl`
- Runtime configuration verification using `sshd -T`
- Consistent application of a security control across a multi-server environment
- Troubleshooting authentication failures using `passwd -S`




























# Disable Root SSH Login (Linux Hardening)

## Overview
This lab demonstrates how to harden Linux servers by disabling direct SSH access for the root user.  
This is a standard security control to enforce least privilege and reduce brute-force attack risks.

---

## Lab Objective
- Prevent direct SSH login as root
- Enforce administrative access via sudo
- Apply changes across all Nautilus app servers

---

## Environment
- Project: Nautilus (Stratos Datacenter)
- Servers:
  - stapp01
  - stapp02
  - stapp03
- OS: Linux
- Service: OpenSSH

---

## Step 1: Access Jump Host

- CONNECT to jump host using SSH
- AUTHENTICATE with provided credentials

📸 Screenshot: Jump host login
<img width="1030" height="788" alt="image" src="https://github.com/user-attachments/assets/2762f39f-7c18-4d8c-aa31-6240f988eaed" />

## Step 2: Connect to App Server
- SSH into target app server (non-root user)

📸 Screenshot: SSH session to app server
<img width="1027" height="825" alt="image" src="https://github.com/user-attachments/assets/c81c1b0e-aa68-43b2-972e-1cfe9ae6bb5c" />

## Step 3: Elevate Privileges
- SWITCH to root using sudo

📸 Screenshot: Root shell confirmation
<img width="1032" height="764" alt="image" src="https://github.com/user-attachments/assets/7303bde5-2fe5-4d20-98f1-f669aeb14da2" />
<img width="1037" height="866" alt="image" src="https://github.com/user-attachments/assets/706dd8cd-f0cc-4743-9c5b-9e83153a7439" />

## Step 4: Edit SSH Configuration
- OPEN /etc/ssh/sshd_config
- LOCATE PermitRootLogin directive
- CHANGE value from "yes" to "no"
- SAVE configuration

📸 Screenshot: sshd_config edited
<img width="1038" height="863" alt="image" src="https://github.com/user-attachments/assets/5f94446f-2de4-4bc0-84d1-45773eaced14" />
<img width="1030" height="842" alt="image" src="https://github.com/user-attachments/assets/986f2b6c-4efd-4166-a38c-35f09d0fc897" />
<img width="1037" height="866" alt="image" src="https://github.com/user-attachments/assets/1746211a-dc77-4765-bed9-6a023843431f" />

## Step 5: Restart SSH Service
- RESTART sshd service to apply changes

📸 Screenshot: SSH service restart
<img width="1040" height="862" alt="image" src="https://github.com/user-attachments/assets/81fab739-cbc9-4b76-b70b-ec7bb10acd72" />
<img width="1037" height="864" alt="image" src="https://github.com/user-attachments/assets/28adb7cc-f0d6-4c62-9d85-38082c86fa7a" />

## Step 6: Verify Configuration
- QUERY SSH daemon runtime config
- CONFIRM PermitRootLogin is set to "no"

📸 Screenshot: Verification output
<img width="1032" height="866" alt="image" src="https://github.com/user-attachments/assets/b9d12071-30ba-4288-acba-31ff55f8b844" />
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/5a1f872b-7675-4043-b59a-420ceed13f12" />

## Result
- Root SSH login successfully disabled

- Servers hardened against direct root access

- SSH access preserved for non-root users

## Security Best Practices
- Disable direct root SSH access

- Use sudo for administrative actions

- Enforce least-privilege access

- Regularly audit SSH configurations

## Real-World Relevance
- This configuration mirrors real production security baselines used in enterprise Linux and cloud environments to prevent unauthorized access.

## Skills Demonstrated
- Linux server hardening

- SSH security configuration

- Privilege management

- Operational security best practices
