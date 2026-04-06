# Apache HTTP Server Outage Resolution: Port Conflict Remediation Across Multi-Node Application Tier

## Table of Contents

- [Incident Summary](#incident-summary)
- [Environment](#environment)
- [Impact Assessment](#impact-assessment)
- [Root Cause Analysis](#root-cause-analysis)
- [Detection and Diagnosis](#detection-and-diagnosis)
- [Remediation Steps](#remediation-steps)
  - [1. Stop the Conflicting Service](#1-stop-the-conflicting-service)
  - [2. Start the Apache Service](#2-start-the-apache-service)
  - [3. Validate Port Binding](#3-validate-port-binding)
  - [4. Enforce Persistence Across Reboots](#4-enforce-persistence-across-reboots)
  - [5. Cross-Node Validation](#5-cross-node-validation)
- [Post-Incident State](#post-incident-state)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Lessons Learned](#lessons-learned)
- [SRE Competencies Demonstrated](#sre-competencies-demonstrated)

---

## Incident Summary

Production monitoring flagged the Apache HTTP Server (`httpd`) as unavailable on the application tier of Stratos Datacenter. The service had entered a failed state at startup due to a port-binding conflict on port `8086`, caused by a competing `sendmail` process that had claimed the socket before `httpd` could bind to it. This document covers the full incident lifecycle: detection, root cause analysis, remediation, validation, and cross-node consistency enforcement across all application servers.

---

## Environment

| Parameter | Value |
|---|---|
| Platform | Linux (systemd-managed) |
| Service | Apache HTTP Server (`httpd`) |
| Incident Scope | Multi-node application tier |
| Affected Port | `8086` |
| Node: stapp01 | `172.16.238.10` (user: tony) |
| Node: stapp02 | `172.16.238.11` (user: steve) |
| Node: stapp03 | `172.16.238.12` (user: banner) |
| Jump Host | `thor@jumphost` |

---

## Impact Assessment

- **Primary impact:** `httpd` service failed to start on stapp01, resulting in service downtime on port `8086`
- **Secondary risk:** Inconsistent service state across the application tier if the issue was node-specific
- **Detection vector:** Monitoring alert triggered on service health check failure
- **Blast radius:** Application traffic to the affected node was unavailable for the duration of the outage

---

## Root Cause Analysis

Apache failed to bind to port `8086` because `sendmail` had already claimed the socket. This caused `httpd` to exit immediately with `status=1/FAILURE` during the `ExecStart` phase. The service was configured as `disabled` in systemd, meaning it would not recover automatically across reboots, compounding the reliability risk.

**Key error from `systemctl status httpd`:**

```
(98)Address already in use: AH00072: make_sock: could not bind to address 0.0.0.0:8086
no listening sockets available, shutting down
```

**Contributing factors:**

- `sendmail` was running on a non-standard port (`8086`) instead of its conventional SMTP port (`25`), suggesting a misconfiguration in the environment
- `httpd` was not enabled at boot (`disabled; vendor preset: disabled`), leaving it vulnerable to a manual start failure with no automatic recovery path
- No port reservation or pre-flight socket checks were in place to prevent conflicting services from claiming the application port

---

## Detection and Diagnosis

### Step 1: SSH to the Affected Node and Check Service Status

Access was established from the jump host to stapp01, and service status was inspected immediately:

```bash
thor@jumphost:~$ ssh tony@172.16.238.10
tony@172.16.238.10's password: <password>

[tony@stapp01 ~]$ sudo systemctl status httpd
```

The output confirmed that `httpd` was in a **failed** state. The `Active` field showed `failed (Result: exit-code)`, and both the `ExecStart` and `ExecStop` directives had exited with `status=1/FAILURE`. The journal logs embedded in the status output clearly indicated the root cause: `AH00072: make_sock: could not bind to address 0.0.0.0:8086`.

> **Screenshot: `httpd` service in failed state on stapp01**
>
> ![httpd failed service status on stapp01](https://github.com/user-attachments/assets/464c0840-06ba-4115-ba60-66af0bf9bfa2)
>
> *`systemctl status httpd` showing the service in `failed` state with exit-code `status=1/FAILURE`. The embedded journal reveals socket binding failure on port `8086` as the root cause.*

---

### Step 2: Identify the Process Holding Port 8086

Port occupancy was confirmed using `ss`, filtered to the specific port:

```bash
[tony@stapp01 ~]$ sudo ss -tulnp | grep :8086
```

Output revealed that `sendmail` (PID 747) had an active `LISTEN` socket bound to `127.0.0.1:8086`, directly blocking Apache from binding.

> **Screenshot: `sendmail` occupying port 8086**
>
> ![sendmail listening on port 8086](https://github.com/user-attachments/assets/d47fea25-7236-4572-931a-2634fc474d60)
>
> *`ss -tulnp` output confirms `sendmail` (PID 747) is listening on `127.0.0.1:8086`, preventing `httpd` from claiming the socket. Note the restricted bind address (`127.0.0.1`), indicating sendmail was misconfigured to hold this port locally.*

---

## Remediation Steps

### 1. Stop the Conflicting Service

`sendmail` was stopped to release its hold on port `8086`:

```bash
[tony@stapp01 ~]$ sudo systemctl stop sendmail
```

No additional action (such as disabling `sendmail`) was required beyond stopping the service, as the operational objective was to restore `httpd` immediately. Disabling `sendmail` permanently would require a separate change management decision.

> **Screenshot: `sendmail` stopped on stapp01**
>
> ![sendmail stopped on stapp01](https://github.com/user-attachments/assets/0c023d6f-1e3c-44c3-9089-f3afe1f2c3a0)
>
> *`sudo systemctl stop sendmail` executed successfully with no output, indicating a clean service stop. The port `8086` is now free for `httpd` to bind.*

---

### 2. Start the Apache Service

With the port conflict resolved, `httpd` was started:

```bash
[tony@stapp01 ~]$ sudo systemctl start httpd
```

The absence of an error message confirmed that `httpd` started without issue. This was immediately followed by a port-binding validation to confirm active listener state.

> **Screenshot: `httpd` started successfully on stapp01**
>
> ![httpd started on stapp01](https://github.com/user-attachments/assets/1d05e776-0e5a-4e46-a7ba-90005b14bd6d)
>
> *`sudo systemctl start httpd` returns cleanly, indicating a successful service start after the port conflict was cleared.*

---

### 3. Validate Port Binding

Listener state was re-verified to confirm `httpd` had successfully claimed port `8086`:

```bash
[tony@stapp01 ~]$ sudo ss -tulnp | grep :8086
```

The output showed `httpd` (multiple worker PIDs) now listening on `*:8086`, confirming the service was active and accepting connections.

> **Screenshot: `httpd` bound to port 8086 on stapp01**
>
> ![httpd listening on port 8086](https://github.com/user-attachments/assets/fe25907f-faa1-4a43-b83b-6669ee6a4246)
>
> *`ss -tulnp` output shows `httpd` worker processes (PIDs 848-853) now holding the LISTEN socket on port `8086`. The send-queue value of `511` confirms the backlog is healthy and the service is accepting connections.*

---

### 4. Enforce Persistence Across Reboots

The service was enabled in systemd to ensure automatic startup on all future reboots, preventing a recurrence of the outage in the event of an unplanned restart:

```bash
[tony@stapp01 ~]$ sudo systemctl enable httpd
```

The command confirmed successful enablement by outputting the symlink creation path:

```
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service
  to /usr/lib/systemd/system/httpd.service.
```

> **Screenshot: `httpd` enabled at boot on stapp01**
>
> ![httpd enabled at boot on stapp01](https://github.com/user-attachments/assets/fe25907f-faa1-4a43-b83b-6669ee6a4246)
>
> *`sudo systemctl enable httpd` creates the systemd symlink under `multi-user.target.wants`, ensuring `httpd` starts automatically at boot. This eliminates the manual recovery requirement that contributed to the initial outage.*

---

### 5. Cross-Node Validation

To enforce consistency across the entire application tier, the same verification and enablement procedures were applied to stapp02 and stapp03.

#### stapp02 (172.16.238.11, user: steve)

```bash
thor@jumphost:~$ ssh steve@172.16.238.11

[steve@stapp02 ~]$ sudo ss -tulnp | grep :8086
[steve@stapp02 ~]$ sudo systemctl enable httpd
```

Port `8086` was already occupied by `httpd` worker processes on stapp02, confirming no remediation was needed. The `enable` command was applied to enforce boot persistence.

> **Screenshot: stapp02 validation showing `httpd` running and enabled**
>
> ![stapp02 httpd running and enabled](https://github.com/user-attachments/assets/2bda11df-b624-4009-84af-2276752951b0)
>
> *stapp02 shows `httpd` already listening on port `8086` (PIDs 1662-1672). `systemctl enable httpd` confirms the symlink was created, aligning stapp02 with the desired boot configuration.*

> **Screenshot: stapp02 enablement confirmation**
>
> ![stapp02 httpd enabled confirmation](https://github.com/user-attachments/assets/ae7cbf40-79a3-4d75-98fe-59a9efe2fb96)
>
> *Full terminal view showing the SSH session to stapp02, the port verification confirming `httpd` is already bound, and the successful `systemctl enable httpd` symlink creation.*

---

#### stapp03 (172.16.238.12, user: banner)

```bash
thor@jumphost:~$ ssh banner@172.16.238.12

[banner@stapp03 ~]$ sudo ss -tulnp | grep :8086
[banner@stapp03 ~]$ sudo systemctl stop sendmail
[banner@stapp03 ~]$ sudo vi /etc/httpd/conf/httpd.conf
[banner@stapp03 ~]$ sudo systemctl restart httpd
[banner@stapp03 ~]$ sudo systemctl enable httpd
[banner@stapp03 ~]$ sudo ss -tulnp | grep :8086
```

On stapp03, `httpd` was also already running on port `8086`. The `stop sendmail` attempt returned an error indicating `sendmail` was not loaded on this node, confirming it posed no conflict. A config review and `httpd` restart were performed as a precaution, followed by enablement and final port validation.

> **Screenshot: stapp03 initial port check showing `httpd` already active**
>
> ![stapp03 httpd already running](https://github.com/user-attachments/assets/6a7f53c9-36f9-4796-85f6-0258cc754285)
>
> *stapp03 port scan confirms `httpd` is already listening on `0.0.0.0:8086`. `sendmail` is not present on this node (`Unit sendmail.service not loaded`), eliminating any port conflict risk.*

> **Screenshot: stapp03 final state showing `httpd` running, enabled, and bound**
>
> ![stapp03 final state confirmation](https://github.com/user-attachments/assets/ee6783e7-1902-485e-925d-17f1a1626530)
>
> *Final validation on stapp03 after `httpd` restart and `systemctl enable`. Port `8086` is actively held by `httpd` worker processes (PIDs 2575-2585), and the service is configured for automatic startup.*

> **Screenshot: stapp03 `systemctl enable` symlink confirmation**
>
> ![stapp03 enable symlink created](https://github.com/user-attachments/assets/c40573c4-db2c-4afc-81a1-f022ef4e76c0)
>
> *`systemctl enable httpd` on stapp03 creates the required symlink under `multi-user.target.wants`, completing the persistence configuration across all three application nodes.*

---

## Post-Incident State

| Node | Service | Port 8086 | Boot Persistence |
|---|---|---|---|
| stapp01 | Running | Bound (httpd) | Enabled |
| stapp02 | Running | Bound (httpd) | Enabled |
| stapp03 | Running | Bound (httpd) | Enabled |

All three application nodes are now in a consistent, healthy state. The port conflict has been resolved, `httpd` is actively serving on port `8086`, and systemd persistence is enforced across the tier.

---

## Errors and Resolutions

| Error | Node | Cause | Resolution |
|---|---|---|---|
| `AH00072: make_sock: could not bind to address 0.0.0.0:8086` | stapp01 | `sendmail` holding port `8086` before `httpd` start | Stopped `sendmail`, freeing the socket for `httpd` |
| `httpd.service: main process exited, code=exited, status=1/FAILURE` | stapp01 | Consequence of socket binding failure | Resolved by clearing the port conflict |
| `Failed to stop sendmail.service: Unit sendmail.service not loaded.` | stapp03 | `sendmail` not installed or loaded on stapp03 | No action required; confirmed non-issue, `httpd` already running |
| `httpd.service` in `disabled` state at first inspection | stapp01 | `httpd` not configured for automatic startup | Resolved with `systemctl enable httpd` |

---

## Key Decisions

- **Stopped `sendmail` rather than reconfiguring its port:** The immediate operational priority was restoring `httpd`. Reconfiguring `sendmail` to use a different port would have been a longer-path change requiring validation of downstream mail delivery. Stopping the service was the lowest-risk, fastest path to resolution.
- **Applied `systemctl enable` to all nodes regardless of prior state:** Even on nodes where `httpd` was already running, enabling the service ensures consistent recovery behavior after any future reboot. Partial enablement across a tier is an operational risk that this step eliminates.
- **Verified port binding post-start using `ss` rather than relying on service status alone:** `systemctl status` can report `active (running)` while the process has not yet completed socket binding. Using `ss -tulnp` provides ground-truth confirmation that the listener is established.
- **Inspected `httpd.conf` on stapp03 as a precautionary measure:** Given that stapp03 showed `httpd` already running but with a slightly different environment (no `sendmail`), a config review was performed to rule out any silent misconfiguration before calling the node clean.

---

## Lessons Learned

- **Port allocation must be managed at the infrastructure level.** Services claiming the same port across different service types (web server vs. mail relay) indicates a gap in port governance. A port inventory or reservation policy would prevent this class of conflict.
- **All long-running services must be enabled at boot.** A service that is running but not enabled is a reliability liability. Any reboot, planned or unplanned, will silently drop the service unless enabled. Boot persistence should be enforced as part of the initial provisioning checklist.
- **Monitoring should include port-level health checks, not just process-level checks.** A process check on `httpd` would not have caught the scenario where `httpd` is stopped and `sendmail` holds its port. Port-level probes (e.g., a TCP check to `8086`) provide earlier and more accurate signal.
- **Cross-node consistency validation should be a mandatory post-remediation step.** A fix applied to a single node in a multi-node tier should always trigger a sweep of peer nodes to confirm uniform state. Node-specific fixes that are not propagated create silent divergence that complicates future incident response.
- **The `ss` utility is preferable to `netstat` for socket inspection** on modern Linux systems. It provides faster output, is available by default on most distributions, and supports rich filtering options.

---

## SRE Competencies Demonstrated

- Structured incident response with clear problem-solution-validation flow
- Root cause analysis from systemd journal output without reliance on external tooling
- Linux service lifecycle management (`start`, `stop`, `enable`, `status`)
- Socket-level diagnostics using `ss -tulnp`
- Port conflict identification and targeted remediation
- Multi-node consistency validation across a distributed application tier
- Preventive configuration hardening (boot persistence enforcement)
- Production reliability and operational readiness mindset






























# Multi-Node Apache Service Outage Resolution (Stratos DC)

## Incident Summary
- Production monitoring detected Apache (`httpd`) service unavailability on one
Nautilus application server within Stratos DC.

- The incident was caused by a port-binding conflict that prevented Apache from
starting successfully. The issue was diagnosed, remediated, and validated
across all application nodes to restore service consistency.

---

## Environment
- Platform: Linux (systemd)
- Service: Apache HTTP Server (httpd)
- Scope: Multi-node application tier
- Nodes:
  - stapp01 (172.16.238.10)
  - stapp02 (172.16.238.11)
  - stapp03 (172.16.238.12)

---

## Impact
- Apache service failed to start on one application node
- Monitoring alert triggered due to service downtime
- Risk of inconsistent application-tier behavior

---

## Root Cause Analysis
- Apache failed to start due to an **address already in use** error.

- Investigation revealed that the required application port was already bound
by another system service (`sendmail`), causing Apache to exit with a failure
status during startup.

---

## Detection & Diagnosis

`sudo systemctl status httpd`

- Observed

`httpd.service` in failed state

Exit code due to socket binding failure

📸 Screenshot: `httpd failed service status`
<img width="1033" height="824" alt="image" src="https://github.com/user-attachments/assets/464c0840-06ba-4115-ba60-66af0bf9bfa2" />

`sudo ss -tulnp | grep :8086`

- Finding

`Port 8086 occupied by sendmail`

📸 Screenshot: `sendmail listening on port 8086`
<img width="1027" height="579" alt="image" src="https://github.com/user-attachments/assets/d47fea25-7236-4572-931a-2634fc474d60" />

## Remediation Steps

## 1. Stop Conflicting Service
`sudo systemctl stop sendmail`

📸 Screenshot: `sendmail stopped`
<img width="1033" height="412" alt="image" src="https://github.com/user-attachments/assets/0c023d6f-1e3c-44c3-9089-f3afe1f2c3a0" />

## 2. Restore Apache Service
`sudo systemctl start httpd`

📸 Screenshot: `httpd started successfully`
<img width="1035" height="485" alt="image" src="https://github.com/user-attachments/assets/1d05e776-0e5a-4e46-a7ba-90005b14bd6d" />

## 3. Validate Port Binding
`sudo ss -tulnp | grep :8086`

- Result

Apache successfully listening on required port

📸 Screenshot: `httpd bound to port 8086`
<img width="1035" height="528" alt="image" src="https://github.com/user-attachments/assets/fe25907f-faa1-4a43-b83b-6669ee6a4246" />

## 4. Enforce Persistence
`sudo systemctl enable httpd`

📸 Screenshot: `httpd enabled at boot`
<img width="1035" height="528" alt="image" src="https://github.com/user-attachments/assets/fe25907f-faa1-4a43-b83b-6669ee6a4246" />

## 5. Cross-Node Validation

- The following checks were performed on all application servers to ensure
configuration consistency:

- `sudo ss -tulnp | grep :8086`
- `sudo systemctl status httpd`
- `sudo systemctl enable httpd`

📸 Screenshots: `Apache running and enabled on all nodes`
<img width="1034" height="403" alt="image" src="https://github.com/user-attachments/assets/2bda11df-b624-4009-84af-2276752951b0" />
<img width="1036" height="659" alt="image" src="https://github.com/user-attachments/assets/ae7cbf40-79a3-4d75-98fe-59a9efe2fb96" />
<img width="1030" height="404" alt="image" src="https://github.com/user-attachments/assets/6a7f53c9-36f9-4796-85f6-0258cc754285" />
<img width="1030" height="383" alt="image" src="https://github.com/user-attachments/assets/ee6783e7-1902-485e-925d-17f1a1626530" />

<img width="1038" height="267" alt="image" src="https://github.com/user-attachments/assets/c40573c4-db2c-4afc-81a1-f022ef4e76c0" />


## Post-Incident State

- Apache service running on all application servers

- Port conflict resolved

- Service enabled to persist across reboots

- Monitoring alert cleared



## SRE Competencies Demonstrated

- Incident response and service restoration

- Root cause analysis

- Linux service lifecycle management

- Port conflict diagnosis

- Multi-node consistency validation

- Production reliability mindset
