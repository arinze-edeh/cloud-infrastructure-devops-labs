# Host-Based Firewall Hardening with iptables Across Multi-Server Application Tier

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Security Requirements](#security-requirements)
- [Architecture](#architecture)
- [Tools and Environment](#tools-and-environment)
- [Implementation](#implementation)
  - [Step 1: Access Application Server via Jump Host](#step-1-access-application-server-via-jump-host)
  - [Step 2: Escalate to Root Privileges](#step-2-escalate-to-root-privileges)
  - [Step 3: Install iptables Persistence Service](#step-3-install-iptables-persistence-service)
  - [Step 4: Enable and Start the iptables Service](#step-4-enable-and-start-the-iptables-service)
  - [Step 5: Flush Existing Firewall Rules](#step-5-flush-existing-firewall-rules)
  - [Step 6: Allow SSH Access](#step-6-allow-ssh-access)
  - [Step 7: Permit Load Balancer Traffic on Application Port](#step-7-permit-load-balancer-traffic-on-application-port)
  - [Step 8: Block All Other Inbound Traffic on Application Port](#step-8-block-all-other-inbound-traffic-on-application-port)
  - [Step 9: Persist Firewall Rules Across Reboots](#step-9-persist-firewall-rules-across-reboots)
  - [Step 10: Replicate Configuration Across All Application Servers](#step-10-replicate-configuration-across-all-application-servers)
- [Key Decisions](#key-decisions)
- [Best Practices](#best-practices)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)
- [Final Outcome](#final-outcome)

---

## Overview

This implementation documents the configuration of host-based firewall rules using **iptables** across three CentOS Stream 9 application servers. The goal was to enforce network-level access control at the host layer, restricting inbound traffic on the application port to a single trusted source: the Load Balancer (LBR) host.

This is a foundational Linux security hardening task relevant to any multi-tier production architecture where defense-in-depth requires both perimeter and host-level controls.

---

## Problem Statement

**Current State:**

- Application port `5001` was publicly accessible with no host-based firewall enforcement
- Any host on the network could reach the application layer directly, bypassing the load balancer
- No firewall rules were in place on `stapp01`, `stapp02`, or `stapp03`
- Rules would not survive a reboot without a persistence mechanism

**Target State:**

- Port `5001` accessible **only** from the LBR host at `172.16.238.14`
- All other inbound connections to port `5001` are silently dropped
- SSH (port `22`) remains fully accessible for administrative operations
- Firewall rules are persisted to survive service restarts and system reboots

---

## Security Requirements

| Requirement | Detail |
|---|---|
| Restrict application port | Port `5001` accessible from LBR only |
| Trusted source IP | `172.16.238.14` (Load Balancer host) |
| Preserve administrative access | SSH port `22` must remain open |
| Rule persistence | Rules must survive reboots via `/etc/sysconfig/iptables` |
| Scope | Applied uniformly to `stapp01`, `stapp02`, `stapp03` |

---

## Architecture

```
                         +-----------------------+
                         |     Jump Host         |
                         |   (thor@jumphost)     |
                         +----------+------------+
                                    |
                          SSH into each stapp server
                                    |
          +-------------------------+-------------------------+
          |                         |                         |
  +-------+-------+         +-------+-------+         +-------+-------+
  |   stapp01     |         |   stapp02     |         |   stapp03     |
  | 172.16.238.10 |         | 172.16.238.11 |         | 172.16.238.12 |
  +-------+-------+         +-------+-------+         +-------+-------+
          |                         |                         |
          +-------------------------+-------------------------+
                                    |
                             Port 5001 (TCP)
                        ACCEPT from 172.16.238.14 only
                        DROP all other sources
                                    |
                         +----------+----------+
                         |   Load Balancer     |
                         |   172.16.238.14     |
                         +---------------------+
```

---

## Tools and Environment

| Component | Detail |
|---|---|
| Operating System | CentOS Stream 9 |
| Firewall Engine | `iptables` |
| Persistence Package | `iptables-services` (v1.8.10-11.1.el9) |
| Service Manager | `systemctl` |
| Package Manager | `yum` |
| Access Method | SSH via jump host |
| Servers Configured | `stapp01`, `stapp02`, `stapp03` |

---

## Implementation

### Step 1: Access Application Server via Jump Host

Connect to the target application server from the designated jump host. The jump host serves as the single entry point for administrative access to the internal network.

**Command:**
```bash
ssh tony@stapp01
```

**Expected behavior:** On first connection, SSH presents the host key fingerprint for verification. Accepting the fingerprint permanently adds the server to `~/.ssh/known_hosts`.

> **Operational note:** In production environments, SSH host key verification should be enforced via a pre-populated `known_hosts` file or a certificate authority rather than interactive acceptance. Interactive acceptance is acceptable in controlled lab environments but represents a TOFU (Trust On First Use) risk in uncontrolled networks.

![SSH login to stapp01 from jump host](https://github.com/user-attachments/assets/a2e9bf81-345b-4b32-8ca5-66055266d94a)
*Successful SSH connection from `thor@jumphost` to `tony@stapp01` at `172.16.238.10`. The host key fingerprint is verified and permanently recorded on first connection.*

---

### Step 2: Escalate to Root Privileges

Firewall management requires root-level access. Escalate using `sudo -i` to obtain a full root shell, which sets the working environment and PATH appropriate for administrative tasks.

**Command:**
```bash
sudo -i
```

> **Security note:** Use `sudo -i` rather than `su -` wherever sudo is available. This preserves audit trail logging through the sudoers facility, associating privileged actions with the originating user account.

![Root privilege escalation via sudo -i](https://github.com/user-attachments/assets/49f1d81f-da73-418e-a909-8b5974f3bae6)
*`sudo -i` grants a root shell on `stapp01`. The sudo lecture confirms the privilege escalation policy enforced on this system.*

---

### Step 3: Install iptables Persistence Service

CentOS Stream 9 does not ship with `iptables-services` by default. This package provides the `iptables.service` systemd unit required to persist and restore firewall rules across reboots. Without it, rules applied at runtime are lost on the next boot.

**Command:**
```bash
yum install iptables-services -y
```

**Expected result:** Package `iptables-services-1.8.10-11.1.el9.noarch` installed from the EPEL repository.

> **Dependency note:** Ensure the EPEL repository is enabled before running this command. If it is not available, enable it with `yum install epel-release -y` first.

![iptables-services package resolution](https://github.com/user-attachments/assets/3d59cac2-ea71-4e19-8d47-f0b79e2f45ed)
*`yum` resolves the `iptables-services` package from the EPEL repository on `stapp01`. Package metadata is refreshed from all configured repositories including CentOS BaseOS, AppStream, Extras, Docker CE, and EPEL.*

![iptables-services installation complete](https://github.com/user-attachments/assets/b60f3bae-d04d-4563-b003-697c4f88cd08)
*Transaction completes successfully. `iptables-services-1.8.10-11.1.el9.noarch` is installed and verified. The `Complete!` output confirms the package is ready for use.*

---

### Step 4: Enable and Start the iptables Service

Enable the `iptables` service to start automatically at boot, then start it immediately for the current session. This is a two-step operation: enabling configures the systemd symlink for automatic startup, while starting activates the service now.

**Commands:**
```bash
systemctl enable iptables
systemctl start iptables
```

**Expected result:** A systemd symlink is created at `/etc/systemd/system/multi-user.target.wants/iptables.service`, confirming the service is set to start on boot.

> **Operational consideration:** Do not skip `systemctl start iptables`. Enabling a service alone does not start it in the current session. Any rules applied before starting the service may behave unexpectedly if the service is initialized afterward without a restart.

![iptables service enabled and started](https://github.com/user-attachments/assets/d3f0cfd9-cf2d-458e-8a77-6710f5fb58cf)
*`systemctl enable iptables` creates the systemd symlink for boot persistence. `systemctl start iptables` activates the service immediately. No errors indicate clean initialization.*

---

### Step 5: Flush Existing Firewall Rules

Before applying a controlled rule set, flush all existing rules in the `filter` table to start from a clean state. This prevents conflicts or unexpected behavior caused by inherited or residual rules from prior sessions.

**Command:**
```bash
iptables -F
```

> **Risk note:** `iptables -F` flushes all chains in the default `filter` table. If you are operating on a live server with active production traffic, flushing rules can temporarily expose the host. Apply the critical ACCEPT rules immediately after flushing, or use a rule-replacement approach instead. In this controlled environment, the flush is safe to perform prior to rule application.

![iptables rules flushed](https://github.com/user-attachments/assets/c04dff37-6e1d-4dd3-9a80-ee6f1a0ba3f7)
*`iptables -F` clears all existing rules from the filter chains on `stapp01`. A clean rule set is confirmed, ready for the target policy to be applied.*

---

### Step 6: Allow SSH Access

Before restricting any traffic, explicitly allow inbound SSH connections on port `22`. This rule must be added **before** any DROP rules to prevent locking yourself out of the server.

**Command:**
```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

**Rule breakdown:**

| Flag | Value | Meaning |
|---|---|---|
| `-A INPUT` | | Append to the INPUT chain |
| `-p tcp` | | Match TCP protocol |
| `--dport 22` | | Match destination port 22 (SSH) |
| `-j ACCEPT` | | Accept matching packets |

> **Critical ordering note:** iptables evaluates rules top-to-bottom in each chain. Always define ACCEPT rules for management access (SSH) before adding DROP rules. Reversing this order can result in immediate loss of administrative access.

![SSH allow rule added to iptables](https://github.com/user-attachments/assets/7aa8dcad-4807-46f9-b1bc-c2b6b480133a)
*The SSH ACCEPT rule is appended to the INPUT chain. Administrative access to the server is secured before any restrictive DROP rules are applied.*

---

### Step 7: Permit Load Balancer Traffic on Application Port

Add an explicit ACCEPT rule that permits inbound TCP traffic on port `5001` **exclusively from the Load Balancer host** at `172.16.238.14`. This is the only source authorized to reach the application tier directly.

**Command:**
```bash
iptables -A INPUT -p tcp -s 172.16.238.14 --dport 5001 -j ACCEPT
```

**Rule breakdown:**

| Flag | Value | Meaning |
|---|---|---|
| `-A INPUT` | | Append to the INPUT chain |
| `-p tcp` | | Match TCP protocol |
| `-s 172.16.238.14` | | Match source IP of the LBR host |
| `--dport 5001` | | Match destination port 5001 |
| `-j ACCEPT` | | Accept matching packets |

> **Design intent:** By specifying the source IP, this rule enforces the single-path ingress model. Application servers must only receive traffic from the load balancer, not directly from clients. This constraint protects the application tier from direct exposure and ensures all traffic passes through the LBR for health checks, connection management, and logging.

![LBR allow rule for port 5001 added](https://github.com/user-attachments/assets/42f8cad9-870d-4dd6-b811-2c063b8f1ebd)
*The source-restricted ACCEPT rule for `172.16.238.14` on port `5001` is appended to the INPUT chain. Only the LBR host is permitted to initiate connections to the application port.*

---

### Step 8: Block All Other Inbound Traffic on Application Port

Add a DROP rule that silently discards all other inbound TCP traffic destined for port `5001`. This rule is evaluated only for traffic that did not match the preceding LBR-specific ACCEPT rule.

**Command:**
```bash
iptables -A INPUT -p tcp --dport 5001 -j DROP
```

**Rule breakdown:**

| Flag | Value | Meaning |
|---|---|---|
| `-A INPUT` | | Append to the INPUT chain |
| `-p tcp` | | Match TCP protocol |
| `--dport 5001` | | Match destination port 5001 |
| `-j DROP` | | Silently discard the packet |

> **DROP vs REJECT:** `DROP` discards packets silently without sending a response to the sender. `REJECT` sends an ICMP or TCP RST response, which is more informative for debugging but also exposes more information to potential attackers. In a hardened production environment, `DROP` is the preferred choice for denied external access.

> **Rule ordering is critical:** The ACCEPT rule for `172.16.238.14` (Step 7) must precede this DROP rule. iptables evaluates rules sequentially, and the first matching rule wins. Reversing the order would drop all traffic to port `5001`, including legitimate LBR traffic.

![DROP rule for port 5001 added](https://github.com/user-attachments/assets/e3120de9-c9c1-4dc6-85ed-f7614948fe03)
*The DROP rule for all other sources on port `5001` is appended after the LBR-specific ACCEPT rule. Traffic from any IP other than `172.16.238.14` to port `5001` is silently discarded.*

---

### Step 9: Persist Firewall Rules Across Reboots

Save the current in-memory iptables rules to the persistent configuration file at `/etc/sysconfig/iptables`. The `iptables-services` package reads this file at boot to restore the rule set automatically.

**Command:**
```bash
service iptables save
```

**Expected result:**
```
iptables: Saving firewall rules to /etc/sysconfig/iptables: [ OK ]
```

> **Persistence mechanism:** The `iptables-services` systemd unit reads from `/etc/sysconfig/iptables` during the `start` operation. Running `service iptables save` writes the current rule set to this file. Any rules not saved before a reboot are lost. Always verify the save confirmation before exiting the session.

![iptables rules persisted to disk](https://github.com/user-attachments/assets/d4a5357e-210f-4a9c-82a7-03bd7880f305)
*`service iptables save` writes all active rules to `/etc/sysconfig/iptables`. The `[ OK ]` status confirms the rules are durably saved and will be restored automatically on the next system boot.*

---

### Step 10: Replicate Configuration Across All Application Servers

Repeat Steps 1 through 9 on all remaining application servers: `stapp02` (user: `steve`) and `stapp03` (user: `banner`). Each server requires an independent installation, configuration, and save operation.

**Server inventory:**

| Server | IP Address | Admin User |
|---|---|---|
| stapp01 | 172.16.238.10 | tony |
| stapp02 | 172.16.238.11 | steve |
| stapp03 | 172.16.238.12 | banner |

**Access commands:**
```bash
ssh banner@stapp03   # Configure stapp03
ssh steve@stapp02    # Configure stapp02
```

**Complete command sequence per server:**
```bash
sudo -i
yum install iptables-services -y
systemctl enable iptables
systemctl start iptables
iptables -F
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp -s 172.16.238.14 --dport 5001 -j ACCEPT
iptables -A INPUT -p tcp --dport 5001 -j DROP
service iptables save
```

**stapp01 full rule application and save:**

![stapp01 complete rule set applied and saved](https://github.com/user-attachments/assets/766ddd96-ad0d-41bc-9145-1f548bb9893e)
*Full command sequence executed on `stapp01`. The rule set shows the SSH ACCEPT rule, the LBR-specific ACCEPT rule for port `5001`, and the catch-all DROP rule applied in correct order.*

**Transitioning to stapp03:**

![Exit stapp01 and connect to stapp03](https://github.com/user-attachments/assets/7a4c773f-2432-41af-99f6-9ce24f687176)
*After completing `stapp01`, the root and user sessions are exited cleanly. SSH is initiated from the jump host to `banner@stapp03` at `172.16.238.12`. Root access is obtained via `sudo -i`.*

**stapp03 iptables-services installation:**

![iptables-services installation on stapp03](https://github.com/user-attachments/assets/9f21b848-5041-4056-bb9a-704147583482)
*`yum install iptables-services -y` resolves and installs the package from the EPEL repository on `stapp03`. Metadata cache is used from a prior refresh.*

**stapp03 service enable, start, and flush:**

![stapp03 service enabled, started, and rules flushed](https://github.com/user-attachments/assets/aad83a94-248a-49af-bc06-f80af275989d)
*`systemctl enable iptables` creates the boot-time symlink on `stapp03`. `systemctl start iptables` activates the service. `iptables -F` clears any pre-existing rules.*

**stapp03 full rule application and save:**

![stapp03 complete rule set applied and saved](https://github.com/user-attachments/assets/46d656cf-8dcb-421d-9c27-3a567b02575d)
*All four iptables rules applied in correct sequence on `stapp03`. `service iptables save` confirms the rule set is written to `/etc/sysconfig/iptables` with `[ OK ]`.*

**Transitioning to stapp02:**

![Exit stapp03 and connect to stapp02](https://github.com/user-attachments/assets/3467c84b-84c3-4616-90bb-17331a81459c)
*`stapp03` sessions are exited cleanly. SSH from the jump host to `steve@stapp02` at `172.16.238.11` is initiated. Host key fingerprint is verified and recorded.*

![Root access obtained on stapp02](https://github.com/user-attachments/assets/7b99f563-46a3-463f-aa1a-5fb11fe3ba78)
*`sudo -i` grants root access on `stapp02` after `steve` authenticates with the sudo password.*

**stapp02 iptables-services installation:**

![iptables-services installation on stapp02](https://github.com/user-attachments/assets/dacde58b-4d4d-415d-b636-307965b81811)
*`yum install iptables-services -y` resolves and installs the package on `stapp02`. Repository metadata is refreshed from all sources. Package is resolved from the EPEL repository.*

![stapp02 installation transaction complete](https://github.com/user-attachments/assets/665b84ee-a815-4eea-b4a8-50b7de1f885e)
*`iptables-services-1.8.10-11.1.el9.noarch` is installed successfully on `stapp02`. `Complete!` confirms the transaction. The service is then enabled and started via `systemctl`.*

**stapp02 service enable and start:**

![stapp02 service enabled and started](https://github.com/user-attachments/assets/5f3baf8d-20cc-4893-abc0-6f178f6386cb)
*`systemctl enable iptables` creates the boot symlink on `stapp02`. `systemctl start iptables` activates the service for the current session.*

**stapp02 full rule application and save:**

![stapp02 complete rule set applied and saved](https://github.com/user-attachments/assets/c746837e-10cc-44d9-9c05-f4614bb58ba0)
*All four firewall rules applied in correct order on `stapp02`. `service iptables save` confirms persistence with `[ OK ]`.*

**stapp02 session exit and jump host return:**

![stapp02 session exit, return to jump host](https://github.com/user-attachments/assets/d5123376-a85c-4d7f-b938-81888e2539b9)
*Root and user sessions on `stapp02` are exited cleanly. The terminal returns to `thor@jumphost`, confirming all three servers have been fully configured and sessions closed.*

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Use `iptables` over `firewalld` | Explicit control over rule ordering and chain structure; more predictable in scripted environments |
| `DROP` instead of `REJECT` for port 5001 | Avoids leaking information about closed ports to unauthorized sources |
| Source IP restriction (`-s 172.16.238.14`) | Enforces single-path ingress model; application tier should only receive traffic from the load balancer |
| SSH rule before DROP rules | Prevents administrative lockout during rule application |
| `iptables-services` for persistence | CentOS Stream 9 does not persist iptables rules natively; this package provides `systemd`-integrated persistence via `/etc/sysconfig/iptables` |
| Manual replication across servers | Ensures each server is independently hardened; in production, Ansible would be used for idempotent fleet-wide enforcement |

---

## Best Practices

- **Always secure SSH before applying DROP rules.** Loss of SSH access on a remote server requires out-of-band console access to recover.
- **Flush before applying a new rule set** to avoid rule accumulation across sessions, which can cause unpredictable behavior.
- **Verify rule order with `iptables -L -n -v --line-numbers`** after applying rules to confirm the chain reads as intended.
- **Save rules immediately after applying them.** Do not rely on memory or assume rules will be saved later.
- **Prefer Ansible for fleet-wide firewall management** in production. Manual SSH-based replication is error-prone and does not scale. An Ansible `iptables` module task can enforce the exact rule set idempotently across all servers.
- **Use `/etc/sysconfig/iptables` for audits.** This file represents the authoritative saved state of the rule set and can be version-controlled in a configuration management repository.
- **Test rule effectiveness** by attempting a connection to port `5001` from a non-LBR host after applying rules, and confirming that the connection is dropped.

---

## Errors and Resolutions

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `yum install iptables-services` fails | EPEL repository not enabled | Run `yum install epel-release -y` first, then retry |
| SSH session drops after flushing rules | DROP rules applied before SSH ACCEPT rule | Re-establish access via console; reapply rules in correct order (SSH ACCEPT first) |
| Rules lost after reboot | `service iptables save` not run, or `iptables-services` not enabled | Run `service iptables save` and verify with `systemctl is-enabled iptables` |
| LBR traffic still blocked after applying rules | ACCEPT and DROP rules applied in wrong order | Verify rule order with `iptables -L -n --line-numbers`; flush and reapply in correct sequence |
| `iptables: command not found` | `iptables` binary not in PATH | Ensure you are in a root shell (`sudo -i`); install `iptables` package if missing |

---

## Lessons Learned

- **Rule ordering is deterministic and unforgiving.** iptables processes rules sequentially and applies the first match. A DROP rule placed before the LBR ACCEPT rule will block all traffic, including legitimate load balancer requests. Always review rule ordering before saving.
- **Persistence is not automatic.** Many engineers assume that rules applied at runtime will survive a reboot. On CentOS Stream 9 without `iptables-services`, they will not. The installation of the persistence package and the explicit save step are both required.
- **Scope creep in manual processes is a real risk.** Repeating the same nine-step process across three servers manually introduces the possibility of drift between servers. In production, this workflow should be codified as an Ansible playbook to guarantee consistency and enable repeatable enforcement.
- **Jump host architecture adds a layer of access control.** All administrative operations flow through a single bastion host, which provides a centralized point for session logging, access control, and audit trail generation.

---

## Final Outcome

All three application servers were successfully hardened with the following security posture:

| Server | iptables Installed | Service Enabled | SSH Open | Port 5001 (LBR) | Port 5001 (Other) | Rules Persisted |
|---|---|---|---|---|---|---|
| stapp01 | Yes | Yes | ACCEPT | ACCEPT | DROP | Yes |
| stapp02 | Yes | Yes | ACCEPT | ACCEPT | DROP | Yes |
| stapp03 | Yes | Yes | ACCEPT | ACCEPT | DROP | Yes |

The application tier now enforces the single-path ingress model. Port `5001` is accessible exclusively from the Load Balancer host (`172.16.238.14`), all other inbound connections to the application port are silently dropped, SSH administrative access is preserved, and the complete rule set is durably persisted across reboots.























# iptables Firewall Configuration for Application Servers

---

## Project Overview

- This project secures application servers by configuring host-based firewall rules using iptables.

- The objective was to:
  -  Install and enable iptables on all application servers
  -  Restrict access to application port 5001
  -  Allow traffic only from the LBR host
  -  Persist firewall rules across reboots

- Servers configured:
  -  stapp01
  -  stapp02
  -  stapp03

---

## Security Requirement

- CURRENT STATE:
    - Application port 5001 was publicly accessible
    - No firewall rules were enforced

- TARGET STATE:
    - Port 5001 accessible ONLY from LBR host (172.16.238.14)
    - All other incoming traffic to port 5001 blocked
    - SSH access (port 22) preserved
    - Rules persistent after reboot

---

## Tools Used

- iptables
- iptables-services
- systemctl
- SSH
- CentOS Stream 9

---

## Implementation Workflow

---

### Step 1: Access Application Server

- ACTION:
  -  Connect to application server via jump host

- COMMAND:
  -  `ssh <user>@<stapp-server>`

SCREENSHOT: `SSH login to application server`
<img width="1027" height="622" alt="image" src="https://github.com/user-attachments/assets/a2e9bf81-345b-4b32-8ca5-66055266d94a" />

---

### Step 2: Switch to Root User

- ACTION:
    Gain administrative privileges

- COMMAND:
   - `sudo -i`

SCREENSHOT: `sudo root access`
<img width="1028" height="858" alt="image" src="https://github.com/user-attachments/assets/49f1d81f-da73-418e-a909-8b5974f3bae6" />

---

### Step 3: Install iptables Services

- ACTION:
  -  Install iptables persistence service

- COMMAND:
  -  `yum install iptables-services -y`

- EXPECTED RESULT:
  -  iptables-services package installed successfully

SCREENSHOT: `iptables-services installation`
<img width="1028" height="858" alt="image" src="https://github.com/user-attachments/assets/3d59cac2-ea71-4e19-8d47-f0b79e2f45ed" />
<img width="1033" height="605" alt="image" src="https://github.com/user-attachments/assets/b60f3bae-d04d-4563-b003-697c4f88cd08" />

---

### Step 4: Enable and Start iptables Service

- ACTION:
  -  Ensure iptables starts automatically on boot

- COMMAND:
  -  `systemctl enable iptables`
  -  `systemctl start iptables`

SCREENSHOT: `iptables service enabled and started`
<img width="1032" height="486" alt="image" src="https://github.com/user-attachments/assets/d3f0cfd9-cf2d-458e-8a77-6710f5fb58cf" />

---

### Step 5: Flush Existing Firewall Rules

- ACTION:
  -  Clear any pre-existing firewall rules

- COMMAND:
  -  `iptables -F`

SCREENSHOT: `iptables rules flushed`
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/c04dff37-6e1d-4dd3-9a80-ee6f1a0ba3f7" />

---

### Step 6: Allow SSH Access

- ACTION:
    `Ensure SSH access is not interrupted`

- COMMAND:
  -  `iptables -A INPUT -p tcp --dport 22 -j ACCEPT`

SCREENSHOT: `SSH allow rule added`
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/7aa8dcad-4807-46f9-b1bc-c2b6b480133a" />

---

### Step 7: Allow LBR Host to Access Application Port

- ACTION:
  -  Permit application traffic from Load Balancer host only

- LBR IP:
    `172.16.238.14`

- COMMAND:
  -  `iptables -A INPUT -p tcp -s 172.16.238.14 --dport 5001 -j ACCEPT`

SCREENSHOT: `LBR allow rule for port 5001`
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/42f8cad9-870d-4dd6-b811-2c063b8f1ebd" />

---

### Step 8: Block All Other Access to Application Port

- ACTION:
  -  Deny application port access from all other sources

- COMMAND:
  -  `iptables -A INPUT -p tcp --dport 5001 -j DROP`

SCREENSHOT: `DROP rule for port 5001`
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/e3120de9-c9c1-4dc6-85ed-f7614948fe03" />

---

### Step 9: Save Firewall Rules

- ACTION:
  -  Persist firewall rules across system reboots

- COMMAND:
  -  `service iptables save`

- EXPECTED RESULT:
  -  Rules saved to `/etc/sysconfig/iptables`

SCREENSHOT: `iptables rules saved`
<img width="1029" height="597" alt="image" src="https://github.com/user-attachments/assets/d4a5357e-210f-4a9c-82a7-03bd7880f305" />

---

### Step 10: Repeat on All Application Servers

- ACTION:
  -  Repeat Steps 1–9 on:
      -  stapp01
      -  stapp02
      -  stapp03

SCREENSHOTS: `firewall applied on all app servers`
<img width="1037" height="716" alt="image" src="https://github.com/user-attachments/assets/766ddd96-ad0d-41bc-9145-1f548bb9893e" />
<img width="1033" height="862" alt="image" src="https://github.com/user-attachments/assets/7a4c773f-2432-41af-99f6-9ce24f687176" />
<img width="1027" height="513" alt="image" src="https://github.com/user-attachments/assets/9f21b848-5041-4056-bb9a-704147583482" />
<img width="1036" height="566" alt="image" src="https://github.com/user-attachments/assets/aad83a94-248a-49af-bc06-f80af275989d" />
<img width="1038" height="607" alt="image" src="https://github.com/user-attachments/assets/46d656cf-8dcb-421d-9c27-3a567b02575d" />
<img width="1034" height="465" alt="image" src="https://github.com/user-attachments/assets/3467c84b-84c3-4616-90bb-17331a81459c" />
<img width="1029" height="659" alt="image" src="https://github.com/user-attachments/assets/7b99f563-46a3-463f-aa1a-5fb11fe3ba78" />
<img width="1026" height="863" alt="image" src="https://github.com/user-attachments/assets/dacde58b-4d4d-415d-b636-307965b81811" />
<img width="1033" height="616" alt="image" src="https://github.com/user-attachments/assets/665b84ee-a815-4eea-b4a8-50b7de1f885e" />
<img width="1031" height="483" alt="image" src="https://github.com/user-attachments/assets/5f3baf8d-20cc-4893-abc0-6f178f6386cb" />
<img width="1033" height="600" alt="image" src="https://github.com/user-attachments/assets/c746837e-10cc-44d9-9c05-f4614bb58ba0" />
<img width="1034" height="690" alt="image" src="https://github.com/user-attachments/assets/d5123376-a85c-4d7f-b938-81888e2539b9" />

---

## Final Outcome

- iptables installed and enabled on all app servers
- Port 5001 secured and restricted to LBR host
- SSH access maintained
- Firewall rules persisted across reboots
- Infrastructure security posture improved
