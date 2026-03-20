# Docker Macvlan Network Provisioning on App Server 2 (Stratos DC)

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment](#environment)
- [Prerequisites](#prerequisites)
- [Resolution](#resolution)
  - [Step 1: SSH into App Server 2](#step-1-ssh-into-app-server-2)
  - [Step 2: Verify Identity and Hostname](#step-2-verify-identity-and-hostname)
  - [Step 3: Confirm Docker Service Status](#step-3-confirm-docker-service-status)
  - [Step 4: List Existing Docker Networks](#step-4-list-existing-docker-networks)
  - [Step 5: Create the Macvlan Network](#step-5-create-the-macvlan-network)
  - [Step 6: Verify Network Creation](#step-6-verify-network-creation)
  - [Step 7: Inspect the Network Configuration](#step-7-inspect-the-network-configuration)
- [Validation](#validation)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This document details the provisioning of a Docker network named **`ecommerce`** on **App Server 2** within the **Stratos DC** environment. The network is configured with the `macvlan` driver, subnet `10.10.1.0/24`, and IP range `10.10.1.0/24` to support upcoming containerized application deployments by the Nautilus DevOps team.

---

## Problem Statement

The Nautilus DevOps team requires pre-configured Docker networks to support multi-application container environments. A team member was tasked with creating a dedicated Docker network on App Server 2 with the following specifications:

| Requirement | Value |
|---|---|
| **Network Name** | `ecommerce` |
| **Target Host** | App Server 2 (`stapp02`) |
| **Data Center** | Stratos DC |
| **Driver** | `macvlan` |
| **Subnet** | `10.10.1.0/24` |
| **IP Range** | `10.10.1.0/24` |

**Impact:** Without this pre-provisioned network, containerized services requiring macvlan-based isolation and custom IP addressing cannot be deployed on App Server 2.

---

## Environment

| Component | Details |
|---|---|
| **Jump Host** | `jump-host` (user: `thor`) |
| **Target Server** | `stapp02` (IP: `10.244.29.224`) |
| **Target User** | `steve` |
| **OS** | Linux (systemd-based) |
| **Docker Version** | Active via `dockerd` with containerd runtime |
| **Auth Method** | SSH with ED25519 key fingerprint verification |

---

## Prerequisites

- SSH access from the jump host to `stapp02`
- Sudo privileges for user `steve` on `stapp02`
- Docker service installed and running on `stapp02`
- Network CIDR `10.10.1.0/24` confirmed available and not conflicting with existing infrastructure

---

## Resolution

### Step 1: SSH into App Server 2

From the jump host, initiate an SSH session to `stapp02` as user `steve`. Accept the host fingerprint upon first connection.

```bash
thor@jump-host ~$ ssh steve@stapp02
```

> **Screenshot**

<img width="1019" height="437" alt="image" src="https://github.com/user-attachments/assets/12ba8a74-c2ce-44c9-97eb-b2b6d3d2b31b" />

> `SSH connection prompt and ED25519 fingerprint verification for stapp02`

**Expected Output:**

```
The authenticity of host 'stapp02 (10.244.29.224)' can't be established.
ED25519 key fingerprint is SHA256:UJmNVd8dDOS44dhTfvk48nqIx9wKocuwUggAJxPF3qI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
```

---

### Step 2: Verify Identity and Hostname

After logging in, confirm you are operating as the correct user on the correct host before making any privileged changes.

```bash
[steve@stapp02 ~]$ whoami
steve

[steve@stapp02 ~]$ hostname
stapp02
```

> **Screenshot**

<img width="1020" height="520" alt="image" src="https://github.com/user-attachments/assets/bebbd4a9-5bf9-4933-9124-e3cc62c2fbbe" />

> `whoami and hostname output confirming correct user and host`

**Why this matters:** Confirming identity and hostname before executing privileged operations is a critical safety gate, especially in multi-server environments where a misidentified target can cause unintended infrastructure changes.

---

### Step 3: Confirm Docker Service Status

Verify that the Docker daemon is active and running before proceeding with network operations.

```bash
[steve@stapp02 ~]$ sudo systemctl status docker
```

> **Screenshot**

<img width="1021" height="858" alt="image" src="https://github.com/user-attachments/assets/e2f792f6-6d31-48b8-a7a3-e423df618a9f" />

> `sudo systemctl status docker showing active (running) state]`

**Expected Output (key lines):**

```
docker.service - Docker Application Container Engine
   Active: active (running) since Fri 2026-03-20 02:27:22 UTC; 33min ago
   Main PID: 1393 (dockerd)
```

**Note:** The Docker daemon (`dockerd`) must be in an `active (running)` state. If Docker is not running, start it with `sudo systemctl start docker` before proceeding.

---

### Step 4: List Existing Docker Networks

Audit the existing Docker networks to confirm the baseline state and ensure no naming conflicts exist.

```bash
[steve@stapp02 ~]$ sudo docker network ls
```

> **Screenshot Placeholder**
> `[SCREENSHOT_04 - docker network ls showing default bridge, host, and none networks before creation]`

**Expected Output:**

```
NETWORK ID     NAME      DRIVER    SCOPE
6c7cf8008bc9   bridge    bridge    local
08f30ba7ca25   host      host      local
6e3c553800b5   none      null      local
```

**Observation:** Only the three default Docker networks are present. No `ecommerce` network exists yet, confirming the environment is clean and ready for provisioning.

---

### Step 5: Create the Macvlan Network

Create the `ecommerce` Docker network using the `macvlan` driver with the specified subnet and IP range.

```bash
[steve@stapp02 ~]$ sudo docker network create \
  --driver macvlan \
  --subnet=10.10.1.0/24 \
  --ip-range=10.10.1.0/24 \
  ecommerce
```

> **Screenshot Placeholder**
> `[SCREENSHOT_05 - docker network create command with macvlan driver, subnet, ip-range, and the returned network ID hash]`

**Expected Output:**

```
6049c10a52cb0172975e6f51abbfcdd8e324af37929c792e75f68aa0de51d4ac
```

A 64-character SHA256 hash is returned, confirming the network object was successfully created and assigned a unique network ID within Docker's internal store.

**Command Breakdown:**

| Flag | Value | Purpose |
|---|---|---|
| `--driver` | `macvlan` | Assigns each container a unique MAC address, making it appear as a physical device on the network |
| `--subnet` | `10.10.1.0/24` | Defines the IP address space for the network |
| `--ip-range` | `10.10.1.0/24` | Restricts the pool of IPs Docker allocates to containers |
| (positional) | `ecommerce` | The human-readable name for the network |

---

### Step 6: Verify Network Creation

Re-list all Docker networks to confirm the `ecommerce` network is now present with the correct driver.

```bash
[steve@stapp02 ~]$ sudo docker network ls
```

> **Screenshot Placeholder**
> `[SCREENSHOT_06 - docker network ls output showing ecommerce network with macvlan driver listed alongside default networks]`

**Expected Output:**

```
NETWORK ID     NAME        DRIVER    SCOPE
6c7cf8008bc9   bridge      bridge    local
6049c10a52cb   ecommerce   macvlan   local
08f30ba7ca25   host        host      local
6e3c553800b5   none        null      local
```

The `ecommerce` network now appears with `DRIVER: macvlan` and `SCOPE: local`, confirming successful creation.

---

### Step 7: Inspect the Network Configuration

Perform a full inspection of the `ecommerce` network to validate all configuration parameters match the ticket requirements exactly.

```bash
[steve@stapp02 ~]$ sudo docker network inspect ecommerce
```

> **Screenshot Placeholder**
> `[SCREENSHOT_07 - Full docker network inspect ecommerce JSON output showing Driver, Subnet, IPRange, and all IPAM configuration fields]`

**Expected Output (abridged):**

```json
[
    {
        "Name": "ecommerce",
        "Id": "6049c10a52cb0172975e6f51abbfcdd8e324af37929c792e75f68aa0de51d4ac",
        "Scope": "local",
        "Driver": "macvlan",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "10.10.1.0/24",
                    "IPRange": "10.10.1.0/24"
                }
            ]
        },
        "Internal": false,
        "Containers": {}
    }
]
```

---

## Validation

All ticket requirements were verified against the final inspected state:

| Requirement | Expected | Actual | Status |
|---|---|---|---|
| Network Name | `ecommerce` | `ecommerce` | PASS |
| Driver | `macvlan` | `macvlan` | PASS |
| Subnet | `10.10.1.0/24` | `10.10.1.0/24` | PASS |
| IP Range | `10.10.1.0/24` | `10.10.1.0/24` | PASS |
| Scope | `local` | `local` | PASS |
| IPv6 Enabled | `false` | `false` | PASS |

> **Screenshot Placeholder**
> `[SCREENSHOT_08 - Side-by-side or annotated final terminal state showing ticket requirements mapped to inspect output]`

---

## Best Practices

### Infrastructure and Security

* **Always SSH from a designated jump host.** Direct access to application servers from personal workstations bypasses audit logging and violates least-privilege principles in production environments.
* **Verify host identity before accepting SSH keys.** Validate the ED25519 fingerprint out-of-band (e.g., via your internal CMDB or infrastructure-as-code repository) before typing `yes` on first connection to prevent man-in-the-middle attacks.
* **Confirm identity and hostname before running privileged commands.** A two-second `whoami` and `hostname` check can prevent costly mistakes in environments with many similarly named servers.

### Docker Network Management

* **Audit existing networks before provisioning new ones.** Running `docker network ls` before creating a network prevents silent name collisions and documents the baseline state.
* **Use the most restrictive IP range that satisfies the requirement.** Setting `--ip-range` equal to or smaller than `--subnet` limits Docker's automatic IP allocation pool, reducing the risk of address conflicts with statically assigned hosts on the same physical segment.
* **Macvlan networks require a parent interface in production.** In a real hardware environment, always specify `--opt parent=eth0` (or the appropriate NIC) so Docker knows which physical interface to attach the macvlan to. Omitting this in lab environments is acceptable but will cause connectivity failures in production.
* **Always run `docker network inspect` after creation.** Do not rely solely on `docker network ls` for validation. Inspect provides the full IPAM configuration and confirms all parameters were applied correctly.
* **Document the network ID.** Record the full 64-character network ID in your ticketing system or CMDB. It is required for troubleshooting and for scripted lookups in automation pipelines.

### Operational Excellence

* **Treat `docker network create` as infrastructure provisioning, not configuration.** Apply the same change management controls you would for any infrastructure change: ticket, peer review, and post-change verification.
* **Use multiline command formatting for readability.** Breaking a `docker network create` command into multiple lines with `\` continuations makes it easier to review in pull requests and audit logs.
* **Store network provisioning commands in version-controlled runbooks or IaC.** Manually typed commands leave no reproducible artifact. Codify this as a Terraform resource, Ansible task, or shell script committed to your infrastructure repository.

---

## Lessons Learned

1. **Docker service state must be confirmed, not assumed.** A running Docker socket does not guarantee the daemon is healthy. `systemctl status docker` provides authoritative confirmation of process state, uptime, and any recent errors before any network operations begin.

2. **Macvlan is not the same as bridge.** The `macvlan` driver assigns a unique MAC address to each container, making it visible directly on the physical network. This is fundamentally different from the default `bridge` driver, which uses NAT. Selecting the wrong driver silently creates a network that will fail at runtime when containers attempt to communicate.

3. **Subnet and IP range are not interchangeable.** `--subnet` defines the full network address space. `--ip-range` controls the portion of that space Docker uses for automatic DHCP-style assignment. Setting them equal (as done here) is a valid and deliberate choice; setting `--ip-range` larger than `--subnet` will cause an error.

4. **The returned hash is your audit trail.** The 64-character ID returned by `docker network create` is the canonical identifier for that network object. It is more reliable than the human-readable name for scripting and troubleshooting, because names can be reused after deletion while IDs are unique.

5. **Jump host workflows enforce accountability.** Routing all server access through `jump-host` via `thor` and then to `stapp02` as `steve` creates a clear, auditable access chain. This pattern is the foundation of privileged access management (PAM) in enterprise environments.

6. **Pre-provisioning networks before container deployment reduces deployment friction.** Creating networks ahead of time, as instructed in this ticket, decouples infrastructure provisioning from application deployment. It allows developers to reference well-known network names in their `docker run` or `docker-compose.yml` configurations without needing infrastructure access at deploy time.

---

## References

* [Docker Documentation: Networking Overview](https://docs.docker.com/network/)
* [Docker Documentation: Use macvlan networks](https://docs.docker.com/network/drivers/macvlan/)
* [Docker CLI Reference: docker network create](https://docs.docker.com/reference/cli/docker/network/create/)
* [systemd Documentation: systemctl](https://www.freedesktop.org/software/systemd/man/systemctl.html)

---






<img width="1035" height="860" alt="image" src="https://github.com/user-attachments/assets/989faa19-d6f0-4320-bf81-e0d2b966f34a" />
<img width="1028" height="765" alt="image" src="https://github.com/user-attachments/assets/e52ddb34-8d81-4833-9600-fe37b4838b32" />
<img width="1030" height="468" alt="image" src="https://github.com/user-attachments/assets/dfea542a-2a9e-49c9-977c-80bf3ddc0cbe" />
<img width="1037" height="507" alt="image" src="https://github.com/user-attachments/assets/527fa9d1-d079-49fd-99df-cf41c07a59c2" />
<img width="1033" height="858" alt="image" src="https://github.com/user-attachments/assets/9853df7a-4cca-40ff-aba3-adcd8832f79c" />

