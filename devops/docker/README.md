# Provisioning a Private AKS Cluster on Azure (Kubernetes 1.33.0)

> **Environment:** Azure Kubernetes Service (AKS) | **Region:** Central US | **Tier:** Free SKU | **Kubernetes:** 1.33.0

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Step-by-Step Deployment Process](#step-by-step-deployment-process)
  - [Step 1: Verify Resource Group](#step-1-verify-resource-group)
  - [Step 2: Resolve Resource Group Region Mismatch (Bug Encountered)](#step-2-resolve-resource-group-region-mismatch-bug-encountered)
  - [Step 3: Query Available Kubernetes Versions](#step-3-query-available-kubernetes-versions)
  - [Step 4: Create the Private AKS Cluster](#step-4-create-the-private-aks-cluster)
  - [Step 5: Verify Addon Profiles](#step-5-verify-addon-profiles)
  - [Step 6: Disable Azure Monitor Metrics](#step-6-disable-azure-monitor-metrics)
- [Cluster Configuration Summary](#cluster-configuration-summary)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Verification and Validation](#verification-and-validation)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Problem Statement

The Nautilus DevOps team required a production-ready, private AKS cluster named `datacenter-aks` deployed to **Central US** with strict configuration requirements:

| Requirement | Value |
|---|---|
| Cluster Name | `datacenter-aks` |
| Kubernetes Version | `1.33.0` |
| Endpoint Access | Private Cluster (no public API server) |
| Region | `Central US` |
| Node Pool Name | `agentpool` |
| Node VM Size | `Standard_D2s_v3` |
| Minimum Node Count | `1` |
| Maximum Node Count | `2` |
| Cluster Autoscaler | Enabled |
| Container Insights / Azure Monitor | Disabled |

**Business Objective:** Deploy a high-availability, cost-optimized Kubernetes cluster with private network access and zero monitoring overhead for workload readiness validation.

---

## Architecture Overview

```
Azure Subscription
 └── Resource Group: kml_rg_main-b4bf4bcbbdf8490c (East US)
      └── AKS Cluster: datacenter-aks (Central US)
           ├── API Server: Private Endpoint Only
           ├── Node Pool: agentpool
           │    ├── VM Size: Standard_D2s_v3
           │    ├── Min Nodes: 1 / Max Nodes: 2
           │    └── Autoscaler: Enabled
           ├── Network Plugin: Azure CNI (Overlay Mode)
           ├── Pod CIDR: 10.244.0.0/16
           ├── Service CIDR: 10.0.0.0/16
           └── Azure Monitor Metrics: DISABLED
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Azure Portal showing the datacenter-aks cluster overview page with private endpoint enabled and region set to Central US]`

---

## Prerequisites

- Azure CLI (`az`) installed and authenticated
- Active Azure subscription with AKS provisioning permissions
- Contributor role or higher on the target resource group
- `kubectl` installed locally (for post-deploy verification)

```bash
# Verify Azure CLI authentication
az account show

# Verify CLI version
az version
```

---

## Environment Setup

Set the working resource group variable used throughout all commands:

```bash
RESOURCE_GROUP="kml_rg_main-b4bf4bcbbdf8490c"
echo $RESOURCE_GROUP
```

**Expected Output:**
```
kml_rg_main-b4bf4bcbbdf8490c
```

> **Screenshot**



> `Terminal showing RESOURCE_GROUP variable set and echoed correctly`

---

## Step-by-Step Deployment Process

### Step 1: Verify Resource Group

List all resource groups in the subscription to confirm the correct group and its location:

```bash
az group list --output table
```

**Observed Output:**
```
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-b4bf4bcbbdf8490c  eastus      Succeeded
```

**Key Observation:** The resource group is located in `eastus`. This is a critical detail because the AKS cluster must be deployed to `centralus` as a separate regional resource. Azure allows AKS clusters to be deployed to a different region than their parent resource group.

> **Screenshot**

<img width="1027" height="445" alt="image" src="https://github.com/user-attachments/assets/9649807c-67d1-4fcd-be5b-396eada93e7d" />

> `Terminal output of az group list --output table showing the resource group in eastus`

---

### Step 2: Resolve Resource Group Region Mismatch (Bug Encountered)

**The Error / Incorrect Attempt:**

An initial attempt was made to dynamically resolve the resource group by filtering on `centralus`:

```bash
# INCORRECT - This query returned an empty result
RESOURCE_GROUP=$(az group list --query "[?location=='centralus'].name" -o tsv)
echo $RESOURCE_GROUP
# Output: (empty - no resource group exists in centralus)
```

**Root Cause:** The resource group `kml_rg_main-b4bf4bcbbdf8490c` resides in `eastus`, not `centralus`. Filtering by `centralus` yielded no results, leaving `$RESOURCE_GROUP` unset.

**Resolution:** Hardcode the correct resource group name after confirming it from `az group list`:

```bash
# CORRECT - Manually set after verifying with az group list
RESOURCE_GROUP="kml_rg_main-b4bf4bcbbdf8490c"
echo $RESOURCE_GROUP
```

**Expected Output:**
```
kml_rg_main-b4bf4bcbbdf8490c
```

> **Screenshot**

<img width="1034" height="500" alt="image" src="https://github.com/user-attachments/assets/70643674-44da-4ec7-a381-3e5c0e0888ed" />

> `Terminal showing the empty output from the centralus query, followed by the manually corrected RESOURCE_GROUP variable`

> **NOTE:** A resource group's location does not constrain where child resources like AKS clusters are deployed. Always verify independently.

---

### Step 3: Query Available Kubernetes Versions

Before provisioning, query the available Kubernetes versions for the **target region** (`centralus`), not the resource group region:

```bash
az aks get-versions --location centralus --output table
```

**Observed Output (abbreviated):**
```
KubernetesVersion    IsPreview    Upgrades                    SupportPlan
-------------------  -----------  --------------------------  --------------------------------------
1.35.0               True         None available              KubernetesOfficial
1.34.3                            1.35.0                      KubernetesOfficial, AKSLongTermSupport
1.33.0                            1.33.1, 1.33.2, ...         KubernetesOfficial, AKSLongTermSupport
1.32.x                            ...                         KubernetesOfficial, AKSLongTermSupport
...
```

**Verification:** Kubernetes `1.33.0` is confirmed available in `centralus` under `KubernetesOfficial` support. It is **not** in preview, making it safe for workload use.

> **Screenshots**

<img width="1128" height="840" alt="image" src="https://github.com/user-attachments/assets/2c4f7380-755d-4096-94e9-ed492f6f48b2" />
<img width="1149" height="864" alt="image" src="https://github.com/user-attachments/assets/02ebea6f-8131-456d-ba44-4b33191321be" />

> `Full terminal output of az aks get-versions --location centralus --output table with version 1.33.0 highlighted`

---

### Step 4: Create the Private AKS Cluster

Deploy the AKS cluster with all required configuration flags in a single `az aks create` command:

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name datacenter-aks \
  --location centralus \
  --kubernetes-version 1.33.0 \
  --enable-private-cluster \
  --node-count 1 \
  --min-count 1 \
  --max-count 2 \
  --enable-cluster-autoscaler \
  --node-vm-size Standard_D2s_v3 \
  --nodepool-name agentpool \
  --generate-ssh-keys
```

**Flag Reference:**

| Flag | Value | Purpose |
|---|---|---|
| `--resource-group` | `kml_rg_main-b4bf4bcbbdf8490c` | Target resource group |
| `--name` | `datacenter-aks` | Cluster identifier |
| `--location` | `centralus` | Deployment region |
| `--kubernetes-version` | `1.33.0` | Pinned stable K8s version |
| `--enable-private-cluster` | (flag) | Disables public API server access |
| `--node-count` | `1` | Initial node count |
| `--min-count` | `1` | Autoscaler lower bound |
| `--max-count` | `2` | Autoscaler upper bound |
| `--enable-cluster-autoscaler` | (flag) | Activates cluster autoscaler |
| `--node-vm-size` | `Standard_D2s_v3` | Node compute SKU (2 vCPU, 8 GB RAM) |
| `--nodepool-name` | `agentpool` | Required node pool name |
| `--generate-ssh-keys` | (flag) | Auto-generates RSA key pair under `~/.ssh/` |

**SSH Key Generation Notice (observed during run):**

```
SSH key files '/root/.ssh/id_rsa' and '/root/.ssh/id_rsa.pub' have been generated
under ~/.ssh to allow SSH access to the VM. If using machines without permanent
storage like Azure Cloud Shell without an attached file share, back up your keys
to a safe location.
```

> **IMPORTANT:** If running in Azure Cloud Shell without a mounted file share, back up the generated SSH keys immediately. They will be lost on session termination.

**Confirmed Output (key fields from JSON response):**

```json
{
  "name": "datacenter-aks",
  "location": "centralus",
  "kubernetesVersion": "1.33.0",
  "currentKubernetesVersion": "1.33.0",
  "provisioningState": "Succeeded",
  "powerState": { "code": "Running" },
  "apiServerAccessProfile": {
    "enablePrivateCluster": true,
    "enablePrivateClusterPublicFqdn": true,
    "privateDnsZone": "system"
  },
  "agentPoolProfiles": [
    {
      "name": "agentpool",
      "vmSize": "Standard_D2s_v3",
      "count": 1,
      "minCount": 1,
      "maxCount": 2,
      "enableAutoScaling": true,
      "mode": "System",
      "orchestratorVersion": "1.33.0"
    }
  ]
}
```

> **Screenshots**

<img width="1263" height="860" alt="image" src="https://github.com/user-attachments/assets/52a739e5-e6d1-4a47-93aa-64fa7443b76b" />
<img width="1263" height="860" alt="image" src="https://github.com/user-attachments/assets/a96b782c-6569-4010-9a27-f2b95bfe45f1" />
<img width="1259" height="863" alt="image" src="https://github.com/user-attachments/assets/7185eb7c-4c80-42ca-9099-5abb6cb1eaec" />
<img width="1256" height="860" alt="image" src="https://github.com/user-attachments/assets/5cb472e2-b5dd-4539-b7d4-7070a9bff3c1" />

> `Terminal output of the az aks create command with provisioningState: Succeeded visible`

---

### Step 5: Verify Addon Profiles

After cluster creation, inspect the addon profiles to confirm no monitoring addons were automatically enabled:

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name datacenter-aks \
  --query "addonProfiles" \
  --output json
```

**Observed Output:**
```
(null / empty)
```

**Interpretation:** No addon profiles are active. This confirms Container Insights was not auto-enabled during cluster creation. The cluster has no monitoring agents deployed.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing null output from the addonProfiles query confirming no addons are active]`

---

### Step 6: Disable Azure Monitor Metrics

Even though `addonProfiles` returned null, Azure Monitor Metrics (Managed Prometheus) may still be enabled at the profile level. Explicitly disable it:

```bash
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name datacenter-aks \
  --disable-azure-monitor-metrics
```

**Observed Output During Run:**
```
Deleting all custom resources for the `azmonitoring.coreos.com` custom resource
definition created by the Managed Prometheus addon
```

**This confirms Managed Prometheus WAS active and has now been fully removed.**

**Confirmed Output (key field from JSON response):**

```json
"azureMonitorProfile": {
  "metrics": {
    "enabled": false,
    "kubeStateMetrics": null
  }
}
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the Managed Prometheus deletion message and the subsequent JSON confirming azureMonitorProfile.metrics.enabled = false]`

> **Screenshot Placeholder**
> `[SCREENSHOT: Azure Portal showing the datacenter-aks Insights/Monitoring blade with monitoring disabled]`

---

## Cluster Configuration Summary

| Property | Configured Value |
|---|---|
| Cluster Name | `datacenter-aks` |
| Resource Group | `kml_rg_main-b4bf4bcbbdf8490c` |
| Location | `centralus` |
| Kubernetes Version | `1.33.0` |
| Node Pool | `agentpool` |
| VM Size | `Standard_D2s_v3` |
| Node Count | `1` (min: 1, max: 2) |
| Autoscaler | Enabled |
| Private Cluster | `true` |
| Public FQDN | Enabled (for portal access only) |
| Private DNS Zone | `system` |
| Network Plugin | `azure` (CNI Overlay) |
| Pod CIDR | `10.244.0.0/16` |
| Service CIDR | `10.0.0.0/16` |
| DNS Service IP | `10.0.0.10` |
| Load Balancer SKU | `standard` |
| Outbound Type | `loadBalancer` |
| Azure Monitor Metrics | `disabled` |
| Container Insights | `disabled` |
| Identity Type | `SystemAssigned` |
| RBAC | Enabled |
| SKU Tier | `Free` |
| OS | Ubuntu 22.04 (AKSUbuntu-2204gen2containerd) |
| Disk CSI Driver | Enabled |
| File CSI Driver | Enabled |
| Snapshot Controller | Enabled |

---

## Errors Encountered and Resolutions

### Error 1: Empty `$RESOURCE_GROUP` Variable from Incorrect Region Filter

**Command That Failed (silently):**
```bash
RESOURCE_GROUP=$(az group list --query "[?location=='centralus'].name" -o tsv)
echo $RESOURCE_GROUP
# Output: (empty string)
```

**Root Cause:** The resource group is in `eastus`, not `centralus`. Filtering by `centralus` returned nothing, setting `$RESOURCE_GROUP` to an empty string. Any subsequent `az aks` command using this variable would fail with a missing resource group error.

**Resolution:**
```bash
# First audit ALL resource groups and their locations
az group list --output table

# Then manually set the variable with the confirmed name
RESOURCE_GROUP="kml_rg_main-b4bf4bcbbdf8490c"
```

**Prevention:** Always verify the variable value with `echo $RESOURCE_GROUP` before running any dependent commands.

---

### Error 2: Managed Prometheus Silently Active Despite Null AddonProfiles

**Observation:** `az aks show --query "addonProfiles"` returned `null`, creating a false impression that all monitoring was disabled.

**Root Cause:** Azure Monitor Managed Prometheus is controlled via `azureMonitorProfile`, a separate API surface from `addonProfiles`. It can be enabled independently of Container Insights addons.

**Evidence of Active Monitoring (seen during disable):**
```
Deleting all custom resources for the `azmonitoring.coreos.com` custom resource
definition created by the Managed Prometheus addon
```

**Resolution:** Always explicitly run `--disable-azure-monitor-metrics` as a post-creation step when monitoring must be fully disabled, regardless of what `addonProfiles` reports.

---

## Verification and Validation

After deployment, run the following checks to validate the cluster meets all requirements:

```bash
# 1. Confirm cluster is running and version is correct
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name datacenter-aks \
  --query "{name:name, k8sVersion:currentKubernetesVersion, state:powerState.code, private:apiServerAccessProfile.enablePrivateCluster}" \
  --output table

# 2. Confirm autoscaler and node pool configuration
az aks nodepool show \
  --resource-group $RESOURCE_GROUP \
  --cluster-name datacenter-aks \
  --name agentpool \
  --query "{vmSize:vmSize, minCount:minCount, maxCount:maxCount, autoscaler:enableAutoScaling}" \
  --output table

# 3. Confirm monitoring is fully disabled
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name datacenter-aks \
  --query "azureMonitorProfile.metrics.enabled" \
  --output tsv
# Expected: false

# 4. Retrieve kubeconfig (requires private network access or AKS command invoke)
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name datacenter-aks

# 5. Verify nodes are ready
kubectl get nodes -o wide
```

> **Screenshot Placeholder**
> `[SCREENSHOT: kubectl get nodes output showing node in Ready state with Kubernetes version 1.33.0]`

---

## Best Practices

### Security
- **Always enable private clusters** for production AKS workloads to prevent public API server exposure.
- **Use System-Assigned Managed Identity** over service principals to eliminate credential rotation overhead.
- **Back up SSH keys** generated with `--generate-ssh-keys` immediately if running in ephemeral environments like Azure Cloud Shell.
- **Disable public FQDN** (`--disable-public-fqdn`) in high-security environments where portal-level DNS resolution is not required.

### Cost Optimization
- **Start with minimum node count of 1** and rely on cluster autoscaler to scale out only under load.
- **Use Free tier SKU** for non-production and lab clusters to avoid unnecessary SLA cost.
- **Disable all monitoring addons** when not required to reduce DaemonSet resource overhead on nodes.

### Infrastructure as Code
- **Never hardcode resource group names** in automation. Use `az group list` with proper tagging strategies (`--query "[?tags.env=='prod'].name"`).
- **Pin Kubernetes versions explicitly** with `--kubernetes-version` to prevent unintended auto-upgrades during cluster creation.
- **Use variable validation** in scripts: always verify critical variables are non-empty before executing dependent commands.

```bash
# Pattern: validate before use
[[ -z "$RESOURCE_GROUP" ]] && { echo "ERROR: RESOURCE_GROUP is not set"; exit 1; }
```

### Operations
- **Query the specific region** when checking available Kubernetes versions. Supported versions differ by region.
- **Separate `addonProfiles` from `azureMonitorProfile`** when auditing monitoring status. They are independent configuration surfaces.
- **Use `az aks update` with intent flags** (`--disable-azure-monitor-metrics`) rather than patching JSON directly to ensure Azure properly cleans up associated CRDs and resources.

---

## Lessons Learned

1. **Resource group location does not constrain AKS cluster location.** An AKS cluster can be deployed to any supported region regardless of where its parent resource group resides. Always specify `--location` explicitly.

2. **Dynamic JMESPath queries on location can silently fail.** A query like `[?location=='centralus']` that returns an empty result will assign an empty string to the variable with no error. This is a silent failure pattern that can cascade into misleading error messages downstream.

3. **`addonProfiles: null` does not mean monitoring is fully disabled.** Azure Monitor Managed Prometheus (`azureMonitorProfile`) operates on a separate API surface and must be explicitly disabled with `--disable-azure-monitor-metrics` after cluster creation.

4. **`--generate-ssh-keys` in Cloud Shell creates ephemeral keys.** Without an Azure Files share mounted to Cloud Shell, generated SSH keys live only for the duration of the session. In production, pre-generate and manage keys externally or use `--no-ssh-key` with AAD-based access.

5. **Version selection requires regional verification.** Kubernetes version availability is region-specific in AKS. Always run `az aks get-versions --location <target-region>` and not `az aks get-versions --location <resource-group-region>`.

6. **Cluster autoscaler requires both `--min-count`, `--max-count`, and `--enable-cluster-autoscaler` together.** Omitting any one flag while providing the others will result in a validation error.

7. **Private clusters with `enablePrivateClusterPublicFqdn: true`** retain a public DNS name that resolves to the private IP. This is useful for Azure Portal connectivity but does not expose the API server publicly. Understand the distinction before assuming it violates your private access requirements.

---

## References

- [Azure AKS Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [az aks create CLI Reference](https://learn.microsoft.com/en-us/cli/azure/aks#az-aks-create)
- [Private AKS Cluster Guide](https://learn.microsoft.com/en-us/azure/aks/private-cluster)
- [AKS Cluster Autoscaler](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler)
- [Disable Azure Monitor Metrics on AKS](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-disable)
- [AKS Supported Kubernetes Versions](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions)

---

*Cluster Provisioned | Kubernetes: 1.33.0 | Region: Central US*






<img width="1258" height="422" alt="image" src="https://github.com/user-attachments/assets/1e8131bb-dccb-46cf-8c16-05454fcf570e" />
<img width="1266" height="820" alt="image" src="https://github.com/user-attachments/assets/7e0c445b-b7e6-4c41-9724-6395e929243d" />
<img width="1259" height="867" alt="image" src="https://github.com/user-attachments/assets/9aa6681f-ede4-48cc-91f1-6abee19f2a30" />
<img width="1263" height="868" alt="image" src="https://github.com/user-attachments/assets/450ff9fd-3d31-484e-9702-7daccd833c85" />
<img width="1258" height="860" alt="image" src="https://github.com/user-attachments/assets/11af9e60-5928-4d53-bf6e-d33d708b783d" />
