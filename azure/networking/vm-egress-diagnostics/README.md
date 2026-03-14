# Azure VM Egress Diagnostics: NSG Outbound Block Remediation

![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_22.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=for-the-badge)
![Region](https://img.shields.io/badge/Region-East_US-blue?style=for-the-badge)

---

## Table of Contents

* [Problem Statement](#problem-statement)
* [Environment Overview](#environment-overview)
* [Root Cause](#root-cause)
* [Prerequisites](#prerequisites)
* [Resolution Workflow](#resolution-workflow)
  * [Phase 0: Subscription Context and Region Configuration](#phase-0-subscription-context-and-region-configuration)
  * [Phase 1: Resource Group Resolution](#phase-1-resource-group-resolution)
  * [Phase 2: Public IP Discovery](#phase-2-public-ip-discovery)
  * [Phase 3: NIC and NSG Resolution](#phase-3-nic-and-nsg-resolution)
  * [Phase 4: NSG Rule Enumeration and SSH Validation](#phase-4-nsg-rule-enumeration-and-ssh-validation)
  * [Phase 5: Pre-Fix Connectivity Confirmation](#phase-5-pre-fix-connectivity-confirmation)
  * [Phase 6: Remediation](#phase-6-remediation)
  * [Phase 7: Post-Fix Verification](#phase-7-post-fix-verification)
* [Verification Results](#verification-results)
* [Key Lessons](#key-lessons)

---

## Problem Statement

The Nautilus DevOps team encountered an issue with an Azure VM named `nautilus-vm`. The team was unable to install any packages on the VM due to connectivity issues. The team needed to identify the root cause of the problem and resolve it to restore normal operations.

**Objectives:**

1. Investigate the connectivity issue preventing package installation on the Azure VM `nautilus-vm`
2. Implement a solution to resolve the connectivity issue and restore package installation capabilities on the VM

> **Note:** The SSH key required to access the Azure VM is already created and added to the VM's authorized keys. The key is located at `/root/.ssh/id_rsa` on the `azure-client` host.

---

## Environment Overview

| Property | Value |
|---|---|
| **Cloud Provider** | Microsoft Azure |
| **Region** | East US (`eastus`) |
| **Subscription Name** | Azure Free Labs |
| **Subscription ID** | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| **Tenant ID** | `54c1a2d3-d100-453c-9636-3a109eb45552` |
| **Resource Group** | `KML_RG_MAIN-ABFB64E535324D83` |
| **Target VM** | `nautilus-vm` |
| **VM OS** | Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1044-azure x86_64) |
| **NIC** | `nautilus-vmVMNic` |
| **NSG** | `nautilus-nsg` |
| **VM Private IP** | `10.0.0.4` (eth0) |
| **VM Public IP** | `20.121.33.102` |
| **SSH Key Path** | `/root/.ssh/id_rsa` |
| **SSH User** | `azureuser` |

---

## Root Cause

A Network Security Group outbound rule named **`Block-All-Outbound`** was configured at **priority 200** on `nautilus-nsg`, explicitly denying all outbound traffic (`*`) before any allow rule could be evaluated.

Azure NSG rules are processed in strict ascending priority order. The lowest priority number always wins. At priority 200, `Block-All-Outbound` intercepted and dropped every outbound packet from `nautilus-vm` before the default `AllowInternetOutBound` rule at priority 65001 could permit it.

```
Priority    Name                   Access    DestPort    Outcome
--------    ----                   ------    --------    -------
200         Block-All-Outbound     Deny      *           FIRES -- drops all outbound traffic
65000       AllowVnetOutBound      Allow     *           Never evaluated
65001       AllowInternetOutBound  Allow     *           Never evaluated
65500       DenyAllOutBound        Deny      *           Never evaluated
```

**Effect:** All outbound connections from `nautilus-vm` were silently dropped. Package manager requests to `azure.archive.ubuntu.com` never left the VM. ICMP requests to `8.8.8.8` resulted in `100% packet loss`.

---

## Prerequisites

* Azure CLI (`az`) authenticated and accessible in the current shell session
* SSH private key present at `/root/.ssh/id_rsa`
* Azure RBAC role with `Network Contributor` permissions or higher on the target resource group

---

## Resolution Workflow

### Phase 0: Subscription Context and Region Configuration

Set the active subscription to the correct Azure Free Labs subscription and configure the default region to East US.

```bash
az account set --subscription $(az account list --query "[0].id" --output tsv)
az configure --defaults location=eastus
```

---

### Phase 1: Resource Group Resolution

Identify the resource group that contains `nautilus-vm` by querying the VM list and extracting the `resourceGroup` property of the first result.

```bash
RG=$(az vm list --query "[0].resourceGroup" --output tsv)
echo "Resource Group: $RG"
```

**Output:**

```
Resource Group: KML_RG_MAIN-ABFB64E535324D83
```

---

### Phase 2: Public IP Discovery

Retrieve the public IP address assigned to `nautilus-vm`. The VM's private IP (`10.0.0.4`) is only routable within its VNet and cannot be targeted from the `azure-client` shell. The public IP is required for all subsequent SSH operations.

```bash
NAUTILUS_PIP=$(az network public-ip list \
  --resource-group $RG \
  --query "[0].ipAddress" \
  --output tsv)
echo "nautilus-vm Public IP: $NAUTILUS_PIP"
```

**Output:**

```
nautilus-vm Public IP: 20.121.33.102
```

---

### Phase 3: NIC and NSG Resolution

Resolve the Network Interface Card (NIC) attached to `nautilus-vm`, then identify the Network Security Group (NSG) bound to that NIC. Both identifiers are required to inspect and modify the firewall rules governing the VM's traffic.

**Capture the NIC name:**

```bash
NIC_NAME=$(az vm show \
  --name nautilus-vm \
  --resource-group $RG \
  --query "networkProfile.networkInterfaces[0].id" \
  --output tsv | xargs basename)
echo "NIC: $NIC_NAME"
```

**Output:**

```
NIC: nautilus-vmVMNic
```

**Capture the NSG name:**

```bash
NSG_NAME=$(az network nic show \
  --name $NIC_NAME \
  --resource-group $RG \
  --query "networkSecurityGroup.id" \
  --output tsv | xargs basename)
echo "NSG: $NSG_NAME"
```

**Output:**

```
NSG: nautilus-nsg
```

> ***Screenshot:***

<img width="1031" height="711" alt="image" src="https://github.com/user-attachments/assets/9b83a617-b655-4551-addc-3302f89144f4" />

---

### Phase 4: NSG Rule Enumeration and SSH Validation

List all outbound and inbound NSG rules on `nautilus-nsg` to identify any rule blocking traffic. Simultaneously open an SSH session into `nautilus-vm` via its public IP to confirm remote access is available.

```bash
echo "=== OUTBOUND RULES ===" && \
az network nsg rule list \
  --nsg-name $NSG_NAME \
  --resource-group $RG \
  --include-default \
  --output table \
  --query "[?direction=='Outbound'].{Priority:priority, Name:name, Access:access, DestPort:destinationPortRange}" && \
echo "=== INBOUND RULES ===" && \
az network nsg rule list \
  --nsg-name $NSG_NAME \
  --resource-group $RG \
  --include-default \
  --output table \
  --query "[?direction=='Inbound'].{Priority:priority, Name:name, Access:access, DestPort:destinationPortRange}" && \
echo "=== TESTING SSH ===" && \
ssh -i /root/.ssh/id_rsa \
    -o StrictHostKeyChecking=no \
    -o ConnectTimeout=15 \
    azureuser@$NAUTILUS_PIP
```

**Outbound rules (culprit identified):**

```
=== OUTBOUND RULES ===
Priority    Name                   Access    DestPort
----------  ---------------------  --------  ----------
200         Block-All-Outbound     Deny      *
65000       AllowVnetOutBound      Allow     *
65001       AllowInternetOutBound  Allow     *
65500       DenyAllOutBound        Deny      *
```

**Inbound rules (SSH access confirmed open):**

```
=== INBOUND RULES ===
Priority    Name                           Access    DestPort
----------  -----------------------------  --------  ----------
100         Allow-SSH                      Allow     22
110         Allow-HTTP                     Allow     80
65000       AllowVnetInBound               Allow     *
65001       AllowAzureLoadBalancerInBound  Allow     *
65500       DenyAllInBound                 Deny      *
```

**SSH connection established:**

```
=== TESTING SSH ===
Warning: Permanently added '20.121.33.102' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1044-azure x86_64)
...
azureuser@nautilus-vm:~$
```

**Finding:** `Block-All-Outbound` at priority 200 is the offending rule. Inbound SSH on port 22 is permitted at priority 100, allowing the session to open via the public IP despite the outbound block.

> ***Screenshots:***

<img width="1035" height="756" alt="image" src="https://github.com/user-attachments/assets/d80b5406-79e3-4314-9636-2784f42bbb11" />
<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/b74b9fe2-bb83-4c61-b34e-c4bc328c5923" />

---

### Phase 5: Pre-Fix Connectivity Confirmation

From inside the active SSH session on `nautilus-vm`, confirm that outbound internet connectivity and package installation are broken. This establishes the documented baseline state before any remediation is applied.

```bash
ping -c 2 8.8.8.8 ; sudo apt-get update 2>&1 | head -5
```

**Output confirming the broken state:**

```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1046ms

Ign:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
Ign:2 http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease
Ign:3 http://azure.archive.ubuntu.com/ubuntu jammy-backports InRelease
Ign:4 http://azure.archive.ubuntu.com/ubuntu jammy-security InRelease
Ign:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
```

Exit the SSH session to return to the `azure-client` shell:

```bash
exit
```

> ***Screenshot:*** 

<img width="1030" height="671" alt="image" src="https://github.com/user-attachments/assets/2d60dab8-45e7-4b6d-8dca-b2862f409e3b" />

>Terminal output from inside `nautilus-vm` showing `100% packet loss` on ICMP and all apt sources returning `Ign` (ignored), confirming complete outbound failure

>Terminal showing `logout`, `Connection to 20.121.33.102 closed.`, and prompt returning to `~ ->`

---

### Phase 6: Remediation

Delete the `Block-All-Outbound` rule from `nautilus-nsg`. Chain the deletion with an immediate re-listing of outbound rules to confirm the rule is gone before proceeding.

```bash
az network nsg rule delete \
  --nsg-name $NSG_NAME \
  --resource-group $RG \
  --name "Block-All-Outbound" && \
echo "=== RULE DELETED -- VERIFYING ===" && \
az network nsg rule list \
  --nsg-name $NSG_NAME \
  --resource-group $RG \
  --include-default \
  --output table \
  --query "[?direction=='Outbound'].{Priority:priority, Name:name, Access:access, DestPort:destinationPortRange}"
```

**Output confirming successful deletion:**

```
=== RULE DELETED -- VERIFYING ===
Priority    Name                   Access    DestPort
----------  ---------------------  --------  ----------
65000       AllowVnetOutBound      Allow     *
65001       AllowInternetOutBound  Allow     *
65500       DenyAllOutBound        Deny      *
```

`Block-All-Outbound` is absent. Outbound traffic from `nautilus-vm` now reaches `AllowInternetOutBound` at priority 65001 and is permitted to all internet destinations.

> ***Screenshot:*** 

<img width="1030" height="671" alt="image" src="https://github.com/user-attachments/assets/2d60dab8-45e7-4b6d-8dca-b2862f409e3b" />

>Terminal output showing `RULE DELETED -- VERIFYING` followed by the three-rule outbound table with `Block-All-Outbound` completely absent

---

### Phase 7: Post-Fix Verification

Re-enter `nautilus-vm` via a single non-interactive SSH command that chains all verification steps: ICMP reachability, package list refresh, package installation, and binary execution confirmation.

```bash
ssh -i /root/.ssh/id_rsa \
    -o StrictHostKeyChecking=no \
    azureuser@$NAUTILUS_PIP \
    "ping -c 4 8.8.8.8 && sudo apt-get update && sudo apt-get install -y curl && curl --version"
```

**ICMP reachability restored:**

```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=114 time=1.74 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=114 time=2.25 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=114 time=1.78 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=114 time=1.65 ms

4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 1.647/1.855/2.251/0.233 ms
```

**Package list refresh restored:**

```
Hit:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]
...
Fetched 44.0 MB in 9s (4890 kB/s)
Reading package lists... Done
```

**Package installation restored:**

```
The following NEW packages will be installed:
  curl
The following packages will be upgraded:
  libcurl4
1 upgraded, 1 newly installed, 0 to remove and 31 not upgraded.
Fetched 484 kB in 0s (18.0 MB/s)
Setting up libcurl4:amd64 (7.81.0-1ubuntu1.23) ...
Setting up curl (7.81.0-1ubuntu1.23) ...
```

**Binary execution confirmed:**

```
curl 7.81.0 (x86_64-pc-linux-gnu) libcurl/7.81.0 OpenSSL/3.0.2 zlib/1.2.11 brotli/1.0.9 zstd/1.4.8
Release-Date: 2022-01-05
Protocols: dict file ftp ftps gopher gophers http https imap imaps ldap ldaps mqtt pop3 pop3s rtmp rtsp scp sftp smb smbs smtp smtps telnet tftp
Features: alt-svc AsynchDNS brotli GSS-API HSTS HTTP2 HTTPS-proxy IDN IPv6 Kerberos Largefile libz NTLM NTLM_WB PSL SPNEGO SSL TLS-SRP UnixSockets zstd
```

> ***Screenshots:***


<img width="1029" height="856" alt="image" src="https://github.com/user-attachments/assets/6931d647-5f26-4311-be53-2c7d323bdacb" />
<img width="1032" height="857" alt="image" src="https://github.com/user-attachments/assets/ea3b7a5b-46fe-454b-946f-b0e9273e5e5d" />
<img width="1031" height="869" alt="image" src="https://github.com/user-attachments/assets/c69d18bf-ad23-4847-889e-d3adbb41a431" />

>Terminal output showing `4 packets transmitted, 4 received, 0% packet loss` with consistent sub-2ms round-trip times

>Terminal output showing all 42 package sources returning `Hit` or `Get`, with `44.0 MB` fetched at `4890 kB/s` and `Reading package lists... Done`

>Terminal output showing `curl` and `libcurl4` fetched at `18.0 MB/s` and installed at version `7.81.0-1ubuntu1.23`

>Terminal output showing `curl 7.81.0 (x86_64-pc-linux-gnu)` version string with full protocol and feature list confirming successful binary execution

---

## Verification Results

| Test | Pre-Fix | Post-Fix |
|---|---|---|
| `ping 8.8.8.8` (4 packets) | `100% packet loss` | `0% packet loss`, avg `1.85ms` |
| `apt-get update` | All sources `Ign` (ignored) | `44.0 MB` fetched in `9s` at `4890 kB/s` |
| `apt-get install -y curl` | No outbound path | `1 upgraded, 1 newly installed` |
| `curl --version` | Not installed | `curl 7.81.0` confirmed |

---

## Key Lessons

**1. NSG rule priority is absolute and linear.**
Azure evaluates NSG rules in strict ascending priority order. A `Deny *` rule at priority 200 overrides every `Allow` rule at priority 65000 or higher without exception. When diagnosing connectivity failures, always list and sort NSG rules by priority before drawing any conclusions.

**2. Always target the public IP for SSH from outside the VNet.**
The VM's private IP (`10.0.0.4`) is only routable within its own VNet. SSH sessions from the `azure-client` shell must use the public IP. Retrieving it via `az network public-ip list` before attempting any connection is a mandatory first step.

**3. Confirm the broken state before applying any fix.**
Running `ping` and `apt-get update` inside the VM before remediation produces concrete, timestamped evidence of the failure. This documents the incident accurately and verifies the fix is targeting the correct symptom.

**4. Chain delete and verify atomically with `&&`.**
Joining `az network nsg rule delete` with a follow-up `az network nsg rule list` in a single `&&` chain ensures the deletion is confirmed immediately. There is no risk of acting on stale state or missing a failed deletion.

**5. Use non-interactive SSH for post-fix verification.**
Passing all verification commands as a single SSH argument eliminates interactive session overhead and produces one clean, unbroken output block that fully documents the restored state.

---





