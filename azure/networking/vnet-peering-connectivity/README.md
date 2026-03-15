# Enabling Secure Inter-VNet Communication Without a Gateway

> **Category:** Networking | **Cloud:** Microsoft Azure | **Region:** East US | **Difficulty:** Intermediate

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Summary](#environment-summary)
- [Implementation](#implementation)
  - [Phase 0: Authentication and Account Verification](#phase-0-authentication-and-account-verification)
  - [Phase 1: Resource Discovery](#phase-1-resource-discovery)
  - [Phase 2: VNet Peering Creation](#phase-2-vnet-peering-creation)
  - [Phase 3: Connectivity Validation](#phase-3-connectivity-validation)
- [Verification and Results](#verification-and-results)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Overview

This document demonstrates the end-to-end implementation of **Azure Virtual Network (VNet) Peering** to enable private network communication between a publicly accessible VM and a privately isolated VM residing in two separate VNets within the same Azure subscription and region.

The implementation follows a structured, audit-friendly workflow covering resource discovery, bidirectional peering creation, NSG awareness, and live connectivity testing.

---

## Problem Statement

**Context:** The Nautilus DevOps team requires secure, low-latency communication between a public-facing Azure VM and a private Azure VM that has no public IP address.

**Challenge:** By default, resources in separate Azure Virtual Networks cannot communicate with each other, even within the same subscription and region. Direct routing between `devops-pub-vnet` (`10.2.0.0/16`) and `devops-priv-vnet` (`10.1.0.0/16`) was not established.

**Resolution:** Configure Azure VNet Peering bidirectionally between both VNets, enabling the public VM to reach the private VM over the Azure backbone network without exposing traffic to the public internet.

---

## Architecture

```
+--------------------------------------------------+
|              Azure Subscription                  |
|         (Azure Free Labs - East US)              |
|                                                  |
|  +---------------------+  VNet Peering  +---------------------+  |
|  |   devops-pub-vnet   | <============> |  devops-priv-vnet   |  |
|  |   10.2.0.0/16       |                |  10.1.0.0/16        |  |
|  |                     |                |                     |  |
|  |  +---------------+  |                |  +---------------+  |  |
|  |  | devops-pub-vm |  |                |  | devops-priv-vm|  |  |
|  |  | Public IP:    |  |                |  | Private only  |  |  |
|  |  | 20.51.148.146 |  |                |  | 10.1.1.4      |  |  |
|  |  | 10.2.1.4      |  |                |  +---------------+  |  |
|  |  +---------------+  |                |  devops-priv-subnet  |  |
|  +---------------------+                +---------------------+  |
|                                                                  |
|  Resource Group: kml_rg_main-3054985a9dfb4f09                   |
+--------------------------------------------------+
```

**Traffic Flow After Peering:**
```
Internet --> devops-pub-vm (20.51.148.146)
                   |
            [VNet Peering]
                   |
             devops-priv-vm (10.1.1.4) [No Public IP]
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure CLI | Installed and authenticated |
| Permissions | Contributor or Network Contributor on both VNets |
| Non-overlapping address spaces | Required for VNet Peering |
| Both VNets in same subscription | Required for this peering type |
| Region | East US (both VNets must exist before peering) |

---

## Environment Summary

| Resource | Name | Value |
|---|---|---|
| Subscription | Azure Free Labs | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| Resource Group | | `kml_rg_main-3054985a9dfb4f09` |
| Location | | `eastus` |
| Public VNet | `devops-pub-vnet` | `10.2.0.0/16` |
| Private VNet | `devops-priv-vnet` | `10.1.0.0/16` |
| Public VM | `devops-pub-vm` | Public: `20.51.148.146` / Private: `10.2.1.4` |
| Private VM | `devops-priv-vm` | Private only: `10.1.1.4` |
| Private Subnet | `devops-priv-subnet` | Within `devops-priv-vnet` |

---

## Implementation

### Phase 0: Authentication and Account Verification

Verify active session and target the correct subscription before touching any resources.

**Step 1: Confirm active Azure CLI session**

```bash
az account show
```

**Expected Output:**

```json
{
  "environmentName": "AzureCloud",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "isDefault": true,
  "name": "Azure Free Labs",
  "state": "Enabled",
  "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552"
}
```

> **Screenshot: `az account show output confirming Azure Free Labs subscription`**

<img width="1028" height="579" alt="Image" src="https://github.com/user-attachments/assets/47ceb51d-c1a4-4258-af5f-6411883d2b5e" />

---

**Step 2: List all subscriptions and confirm target**

```bash
az account list --output table
```

**Expected Output:**

```
Name             CloudName    SubscriptionId                        TenantId                              State    IsDefault
---------------  -----------  ------------------------------------  ------------------------------------  -------  -----------
Azure Free Labs  AzureCloud   f0c3bcdd-5ce2-4fa0-8cf3-41559747512b  54c1a2d3-d100-453c-9636-3a109eb45552  Enabled  True
```

> **Screenshot: `az account list --output table showing Azure Free Labs as IsDefault = True`**

<img width="1028" height="579" alt="Image" src="https://github.com/user-attachments/assets/47ceb51d-c1a4-4258-af5f-6411883d2b5e" />

---

**Step 3: Explicitly set active subscription**

Even when `IsDefault` is `True`, always set the subscription explicitly to prevent context drift in multi-subscription environments.

```bash
az account set --subscription "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b"
```

> No output on success. This is expected.

---

### Phase 1: Resource Discovery

Never assume resource names or configurations. Enumerate all existing resources before modifying anything.

**Step 4: Discover resource groups**

```bash
az group list --output table
```

**Actual Output:**

```
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-3054985a9dfb4f09  eastus      Succeeded
```

> **Screenshot: `az group list showing kml_rg_main-3054985a9dfb4f09 in eastus with Succeeded status`**

<img width="1029" height="671" alt="Image" src="https://github.com/user-attachments/assets/db25aaba-1f73-4f08-95b1-5a10ea759f35" />

---

**Step 5: Verify public VM existence and location**

```bash
az vm show \
  --name devops-pub-vm \
  --resource-group kml_rg_main-3054985a9dfb4f09 \
  --query "{Name:name, ResourceGroup:resourceGroup, Location:location}" \
  --output table
```

**Actual Output:**

```
Name           ResourceGroup                 Location
-------------  ----------------------------  ----------
devops-pub-vm  kml_rg_main-3054985a9dfb4f09  eastus
```

> **Screenshot: `az vm show confirming devops-pub-vm exists in eastus`**

<img width="1032" height="754" alt="Image" src="https://github.com/user-attachments/assets/9270fc90-4a77-49c6-ae44-72761e8acfac" />

---

**Step 6: Retrieve IP addresses for the public VM**

```bash
az vm list-ip-addresses \
  --name devops-pub-vm \
  --resource-group kml_rg_main-3054985a9dfb4f09 \
  --output table
```

**Actual Output:**

```
VirtualMachine    PublicIPAddresses    PrivateIPAddresses
----------------  -------------------  --------------------
devops-pub-vm     20.51.148.146        10.2.1.4
```

> **Screenshot: `az vm list-ip-addresses for devops-pub-vm showing 20.51.148.146 public and 10.2.1.4 private`**

<img width="1032" height="754" alt="Image" src="https://github.com/user-attachments/assets/9270fc90-4a77-49c6-ae44-72761e8acfac" />

---

**Step 7: Retrieve IP addresses for the private VM**

```bash
az vm list-ip-addresses \
  --name devops-priv-vm \
  --resource-group kml_rg_main-3054985a9dfb4f09 \
  --output table
```

**Actual Output:**

```
VirtualMachine    PrivateIPAddresses
----------------  --------------------
devops-priv-vm    10.1.1.4
```

> Note: The absence of a `PublicIPAddresses` column confirms this VM is fully private.

> **Screenshot: `az vm list-ip-addresses for devops-priv-vm showing only private IP 10.1.1.4 and no public IP`**

<img width="1032" height="754" alt="Image" src="https://github.com/user-attachments/assets/9270fc90-4a77-49c6-ae44-72761e8acfac" />

---

**Step 8: Enumerate all VNets and confirm address spaces do not overlap**

```bash
az network vnet list \
  --resource-group kml_rg_main-3054985a9dfb4f09 \
  --output table
```

**Actual Output:**

```
Name              ResourceGroup                 Location    NumSubnets    Prefixes     DnsServers    DDOSProtection    VMProtection
----------------  ----------------------------  ----------  ------------  -----------  ------------  ----------------  --------------
devops-priv-vnet  kml_rg_main-3054985a9dfb4f09  eastus      1             10.1.0.0/16                False
devops-pub-vnet   kml_rg_main-3054985a9dfb4f09  eastus      1             10.2.0.0/16                False
```

> **Address Space Validation:** `10.1.0.0/16` and `10.2.0.0/16` do not overlap. VNet Peering will succeed.

> **Screenshot Placeholder**
> `[ SCREENSHOT: az network vnet list showing both devops-priv-vnet (10.1.0.0/16) and devops-pub-vnet (10.2.0.0/16) ]`

---

### Phase 2: VNet Peering Creation

Azure VNet Peering is **not automatically bidirectional**. Two separate peering objects must be created: one from each VNet pointing to the other.

**Step 9: Create peering from Public VNet to Private VNet**

```bash
az network vnet peering create \
  --name devops-pub-to-priv-peering \
  --resource-group kml_rg_main-3054985a9dfb4f09 \
  --vnet-name devops-pub-vnet \
  --remote-vnet /subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-3054985a9dfb4f09/providers/Microsoft.Network/virtualNetworks/devops-priv-vnet \
  --allow-vnet-access true \
  --allow-forwarded-traffic true \
  --output json
```

**Actual Output (key fields):**

```json
{
  "name": "devops-pub-to-priv-peering",
  "peeringState": "Initiated",
  "peeringSyncLevel": "RemoteNotInSync",
  "provisioningState": "Succeeded",
  "allowVirtualNetworkAccess": true,
  "allowForwardedTraffic": true,
  "remoteAddressSpace": {
    "addressPrefixes": ["10.1.0.0/16"]
  }
}
```

> `peeringState: Initiated` is expected at this stage. The state transitions to `Connected` once the return peering is created in the next step.

> **Screenshot Placeholder**
> `[ SCREENSHOT: az network vnet peering create output for devops-pub-to-priv-peering showing peeringState: Initiated and provisioningState: Succeeded ]`

---

**Step 10: Create return peering from Private VNet to Public VNet**

```bash
az network vnet peering create \
  --name devops-priv-to-pub-peering \
  --resource-group kml_rg_main-3054985a9dfb4f09 \
  --vnet-name devops-priv-vnet \
  --remote-vnet /subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-3054985a9dfb4f09/providers/Microsoft.Network/virtualNetworks/devops-pub-vnet \
  --allow-vnet-access true \
  --allow-forwarded-traffic true \
  --output json
```

**Actual Output (key fields):**

```json
{
  "name": "devops-priv-to-pub-peering",
  "peeringState": "Connected",
  "peeringSyncLevel": "FullyInSync",
  "provisioningState": "Succeeded",
  "allowVirtualNetworkAccess": true,
  "allowForwardedTraffic": true,
  "remoteAddressSpace": {
    "addressPrefixes": ["10.2.0.0/16"]
  }
}
```

> `peeringState: Connected` and `peeringSyncLevel: FullyInSync` confirm the bidirectional peering is fully established.

> **Screenshot Placeholder**
> `[ SCREENSHOT: az network vnet peering create output for devops-priv-to-pub-peering showing peeringState: Connected and peeringSyncLevel: FullyInSync ]`

---

**Step 11: Verify peering state on the Public VNet side**

```bash
az network vnet peering show \
  --name devops-pub-to-priv-peering \
  --resource-group kml_rg_main-3054985a9dfb4f09 \
  --vnet-name devops-pub-vnet \
  --query "{Name:name, State:peeringState, Provisioning:provisioningState}" \
  --output table
```

**Actual Output:**

```
Name                        State      Provisioning
--------------------------  ---------  --------------
devops-pub-to-priv-peering  Connected  Succeeded
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: az network vnet peering show for devops-pub-to-priv-peering showing State: Connected, Provisioning: Succeeded ]`

---

**Step 12: Verify peering state on the Private VNet side**

```bash
az network vnet peering show \
  --name devops-priv-to-pub-peering \
  --resource-group kml_rg_main-3054985a9dfb4f09 \
  --vnet-name devops-priv-vnet \
  --query "{Name:name, State:peeringState, Provisioning:provisioningState}" \
  --output table
```

**Actual Output:**

```
Name                        State      Provisioning
--------------------------  ---------  --------------
devops-priv-to-pub-peering  Connected  Succeeded
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: az network vnet peering show for devops-priv-to-pub-peering showing State: Connected, Provisioning: Succeeded ]`

> Both peerings show `Connected` and `Succeeded`. Phase 2 is complete.

---

### Phase 3: Connectivity Validation

With peering confirmed, validate that actual network traffic flows from the public VM to the private VM.

**Step 13: SSH into the public VM**

```bash
ssh azureuser@20.51.148.146
```

**Actual Session Output:**

```
The authenticity of host '20.51.148.146 (20.51.148.146)' can't be established.
ECDSA key fingerprint is SHA256:de9G3pgafiW70bxsFDBeOu10ApFZHa1yA8eEj2z6ZQU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '20.51.148.146' (ECDSA) to the list of known hosts.

Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1044-azure x86_64)
...
IPv4 address for eth0: 10.2.1.4
```

> The SSH host key warning on first connection is expected behavior. Typing `yes` adds the host to `~/.ssh/known_hosts`.

> **Screenshot Placeholder**
> `[ SCREENSHOT: SSH connection to 20.51.148.146 showing Ubuntu 22.04 welcome banner and eth0 IP 10.2.1.4 ]`

---

**Step 14: Ping the private VM from within the public VM**

```bash
ping -c 4 10.1.1.4
```

**Actual Output:**

```
PING 10.1.1.4 (10.1.1.4) 56(84) bytes of data.
64 bytes from 10.1.1.4: icmp_seq=1 ttl=64 time=2.21 ms
64 bytes from 10.1.1.4: icmp_seq=2 ttl=64 time=1.29 ms
64 bytes from 10.1.1.4: icmp_seq=3 ttl=64 time=1.07 ms
64 bytes from 10.1.1.4: icmp_seq=4 ttl=64 time=1.10 ms

--- 10.1.1.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 1.071/1.417/2.211/0.465 ms
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: ping -c 4 10.1.1.4 showing 4/4 packets received, 0% packet loss, avg latency 1.417ms ]`

---

## Verification and Results

| Checkpoint | Expected | Actual | Status |
|---|---|---|---|
| Subscription active | Enabled | Enabled | PASS |
| Resource group location | eastus | eastus | PASS |
| VNet address spaces non-overlapping | No overlap | `10.1.0.0/16` vs `10.2.0.0/16` | PASS |
| Peering pub-to-priv provisioning | Succeeded | Succeeded | PASS |
| Peering priv-to-pub provisioning | Succeeded | Succeeded | PASS |
| Peering pub-to-priv state | Connected | Connected | PASS |
| Peering priv-to-pub state | Connected | Connected | PASS |
| SSH to public VM | Success | `azureuser@devops-pub-vm` | PASS |
| Ping private VM from public VM | 0% packet loss | 0% packet loss, avg 1.417ms | PASS |

---

## Best Practices

### Network Design

* **Always validate address space uniqueness** before creating VNet Peering. Overlapping CIDRs (`10.1.0.0/16` vs `10.1.0.0/24`) will cause the peering to fail with a non-obvious error. Plan your IP address scheme across all VNets upfront.
* **Use `/16` supernets for VNets** and subdivide into `/24` or smaller subnets per workload tier. This leaves room to grow without re-IP-ing.
* **Prefer private-only VMs** for backend workloads. The `devops-priv-vm` has no public IP, which is the correct posture: all access goes through the peered network, not the internet.

### Peering Configuration

* **Always create both directions.** The first peering starts in `Initiated` state precisely because the remote side has not yet acknowledged it. The second peering transitions both to `Connected`. Forgetting the return peering is the most common mistake.
* **Use full resource IDs for `--remote-vnet`** rather than just the VNet name. This eliminates ambiguity in multi-subscription or multi-resource-group environments and avoids lookup failures.
* **Set `--allow-vnet-access true` explicitly.** Although it is the default, making it explicit in automation scripts ensures intent is clear and prevents accidental traffic blocking if defaults change in future CLI versions.
* **Set `--allow-forwarded-traffic true`** if you plan to route traffic through Network Virtual Appliances (NVAs) or Azure Firewall. Enable it from the start to avoid revisiting peering settings later.

### Security

* **Do not rely on VNet Peering alone as a security boundary.** Layer Network Security Groups (NSGs) on subnets and NICs to control which protocols and ports are permitted between peered VNets.
* **Limit ICMP inbound rules in NSGs to known source ranges** in production. The lab environment allowed broad ICMP for testing; in production, scope this to specific management CIDRs.
* **Audit peering with `--allow-gateway-transit` carefully.** Gateway transit allows a peered VNet to use the other VNet's VPN or ExpressRoute gateway. Only enable this intentionally.

### Operational Hygiene

* **Tag all peering resources** with environment, owner, and purpose tags for cost attribution and audit trail.
* **Use `--output json`** when creating peerings in automation pipelines. The JSON output is parseable and useful for downstream validation scripts.
* **Store peering IDs and resource IDs** in a state file or Key Vault secret for repeatable operations.

---

## Lessons Learned

**1. VNet Peering is directional at creation time, not symmetric.**
When the first peering (`pub-to-priv`) was created, its `peeringState` showed `Initiated` rather than `Connected`. This is not an error. Azure requires both sides to be configured before traffic flows. Teams unfamiliar with this behavior sometimes open support tickets prematurely. Always create both peering objects before testing connectivity.

**2. `peeringSyncLevel: FullyInSync` is the definitive success indicator.**
The `peeringState: Connected` field alone is necessary but not sufficient in complex scenarios. `peeringSyncLevel: FullyInSync` confirms that both sides are synchronized and no configuration drift exists between the two peering objects.

**3. Resource discovery before modification prevents errors.**
Running `az network vnet list` before creating peerings confirmed the exact VNet names, address prefixes, and resource group. Using the wrong VNet name or an overlapping address space would have caused silent failures or peering errors that are harder to debug retroactively.

**4. The private VM having no public IP is by design, not a gap.**
The absence of a `PublicIPAddresses` column in the `az vm list-ip-addresses` output for `devops-priv-vm` confirmed the VM is fully private. This is correct posture. The only ingress path to this VM is through the peered VNet, which is exactly the architecture VNet Peering is designed to enable.

**5. SSH host key acceptance is a one-time operation.**
The ECDSA fingerprint warning on first SSH connection to `devops-pub-vm` is expected. In production pipelines, pre-populate `known_hosts` using `ssh-keyscan` to avoid interactive prompts breaking automation.

**6. ICMP (ping) works across peered VNets when NSG rules permit.**
The ping succeeded without additional NSG changes in this lab because the existing NSG rules allowed ICMP. In hardened production environments, ICMP may be blocked at the subnet NSG level. Always check NSG effective rules if ping fails despite peering showing `Connected`.

---

## Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| `peeringState: Initiated` after first peering | Return peering not yet created | Create the second peering from the remote VNet side |
| `peeringState: Disconnected` | One side was deleted or misconfigured | Delete both peerings and recreate them in sequence |
| Peering creation fails with overlap error | Address spaces overlap between VNets | Redesign one VNet's address space before peering |
| Ping fails despite peering `Connected` | NSG blocking ICMP | Add inbound NSG rule allowing ICMP from source range |
| SSH connection refused | NSG blocking port 22 or VM not running | Check NSG rules for port 22 and verify VM power state |
| `ResourceNotFound` on peering create | Wrong resource group or VNet name | Re-run `az network vnet list` to confirm exact names |

---

## References

* [Azure Virtual Network Peering Documentation](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)
* [az network vnet peering CLI Reference](https://learn.microsoft.com/en-us/cli/azure/network/vnet/peering)
* [Azure Network Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
* [Azure VNet Address Space Planning](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq)

---







<img width="1071" height="822" alt="Image" src="https://github.com/user-attachments/assets/a8e8fc90-ff4e-438f-8f3b-5447c32ecd5a" />
<img width="1071" height="868" alt="Image" src="https://github.com/user-attachments/assets/c2e69fa6-9eb9-47a2-9f02-fb88f60d7b5a" />
<img width="1069" height="862" alt="Image" src="https://github.com/user-attachments/assets/49c52bfb-dd6c-465f-aa58-45169fcd9073" />
<img width="1074" height="865" alt="Image" src="https://github.com/user-attachments/assets/45f894be-4e47-48b5-871b-4246135da61b" />
<img width="1074" height="863" alt="Image" src="https://github.com/user-attachments/assets/e053c363-2a45-42c7-ab3d-48179568d3fc" />
<img width="1074" height="524" alt="Image" src="https://github.com/user-attachments/assets/82920699-61a1-4b5e-a59c-d2b8bd2bc714" />
