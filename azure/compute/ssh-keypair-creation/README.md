# Azure SSH Key Pair Provisioning Using the Azure Portal Control Plane

> **Platform:** Microsoft Azure | **Method:** Azure Portal (Web UI) | **Key Type:** RSA

---

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Prerequisites](#prerequisites)
- [Architecture and Resource Context](#architecture-and-resource-context)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Log In to the Azure Portal](#step-1-log-in-to-the-azure-portal)
  - [Step 2: Navigate to the SSH Keys Service](#step-2-navigate-to-the-ssh-keys-service)
  - [Step 3: Initiate SSH Key Creation](#step-3-initiate-ssh-key-creation)
  - [Step 4: Review Configuration and Submit](#step-4-review-configuration-and-submit)
  - [Step 5: Download the Private Key and Create the Resource](#step-5-download-the-private-key-and-create-the-resource)
  - [Step 6: Verify SSH Key Resource Creation](#step-6-verify-ssh-key-resource-creation)
- [Validation Outcome](#validation-outcome)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Operational Best Practices](#operational-best-practices)
- [Key Learnings](#key-learnings)
- [Conclusion](#conclusion)

---

## Overview

| Field | Value |
|---|---|
| Document Type | Engineering Runbook / Implementation Guide |
| Scope | Azure SSH Key Pair Provisioning via Portal |
| Subscription | Azure Free Labs |
| Resource Group | kml_rg_main-513d23304c80408c |
| Region | East US |
| Key Pair Name | devops-kp |
| SSH Key Format | RSA (Azure-generated) |
| Audience | DevOps Engineers, Cloud Administrators, Onboarding Teams |

This runbook documents the end-to-end process for provisioning an Azure-managed SSH key pair using the Microsoft Azure Portal. The procedure establishes a cryptographic key resource within Azure's control plane, enabling passwordless, secure SSH connectivity to Azure Virtual Machines.

Azure-managed SSH keys are stored as first-class Azure resources, making them subject to Azure Role-Based Access Control (RBAC), tagging, auditing, and lifecycle management policies. This approach is the recommended standard for teams operating within Azure-governed environments.

### Problem Statement

SSH key pairs generated locally (e.g., via `ssh-keygen` on a workstation) are not registered as Azure resources. This means they bypass Azure's control plane, making them invisible to identity governance tooling, auditing pipelines, and automated validation workflows. Teams relying on locally generated keys risk compliance gaps, onboarding friction, and failed infrastructure validation checks.

### Solution

Provisioning SSH key pairs directly through the Azure Portal creates a managed resource object. Azure generates the RSA key pair server-side, retains the public key within the resource, and delivers the private key to the operator at creation time as a one-time download. The resulting key resource is verifiable, auditable, and compatible with automated control-plane validation.

---

## Objectives

- Create an RSA SSH key pair registered as a native Azure resource
- Name the key pair **devops-kp** per project naming conventions
- Provision the resource using the Azure Portal web interface
- Download the private key securely at creation time
- Verify the key exists as a managed Azure resource post-provisioning

---

## Prerequisites

> **Important:** The private key is available for download **only once**, immediately at the moment of creation. Azure does not store the private key after this point. Ensure your secure storage destination is ready before initiating creation.

- **Azure Account:** Valid Azure subscription with Contributor or Owner permissions on the target resource group
- **Resource Group:** `kml_rg_main-513d23304c80408c` must exist in East US prior to key creation
- **Browser Access:** Access to [https://portal.azure.com](https://portal.azure.com) from a supported browser (Edge, Chrome, Firefox)
- **Secure Storage:** A secure local path ready to receive the downloaded private key (`.pem` file)

---

## Architecture and Resource Context

| Property | Detail |
|---|---|
| Azure Resource Type | `Microsoft.Compute/sshPublicKeys` |
| Key Algorithm | RSA (Azure default) |
| Public Key Storage | Azure Resource Manager (ARM) |
| Private Key Storage | Operator's local secure storage (one-time download) |
| Access Control | Azure RBAC on the sshPublicKeys resource |
| Auditability | Azure Activity Log, Azure Monitor |
| Naming Constraint | Alphanumeric and hyphens allowed; must match automation expectations exactly |

---

## Step-by-Step Implementation

### Step 1: Log In to the Azure Portal

Navigate to [https://portal.azure.com](https://portal.azure.com) and authenticate using your Azure credentials. Verify you are operating under the correct subscription (**Azure Free Labs**) and directory context before proceeding. Subscription context is visible in the top-right account panel.

![Azure Portal Home page confirming successful authentication and subscription context](https://github.com/user-attachments/assets/e420a513-502a-42a0-b3bb-f241060b19bf)
*Figure 1: Azure Portal Home page confirming successful authentication and subscription context.*

---

### Step 2: Navigate to the SSH Keys Service

From the Azure Portal home page, use the top search bar to locate the SSH Keys service:

- Click the search bar (shortcut: `G + /`)
- Type **SSH keys** and select the SSH keys service from the results
- The SSH Keys blade opens, displaying all existing SSH key resources in scope

> **Operational Note:** The SSH Keys service is scoped to the selected subscription. Confirm the subscription filter is set correctly before creating the resource.

![SSH Keys service blade showing no existing keys](https://github.com/user-attachments/assets/91d6c39d-068e-4268-951e-0c5133f7e41e)
*Figure 2: SSH Keys service blade showing no existing keys. The empty state confirms a clean starting point for provisioning.*

---

### Step 3: Initiate SSH Key Creation

From the SSH Keys blade, click the **+ Create** button in the top action bar. This launches the **Create an SSH key** provisioning wizard. The wizard follows a **Basics > Tags > Review + create** flow.

#### Configure SSH Key Properties (Basics Tab)

On the Basics tab, enter the following configuration values:

| Field | Value |
|---|---|
| Subscription | Azure Free Labs |
| Resource Group | kml_rg_main-513d23304c80408c |
| Region | East US |
| Key pair name | devops-kp |
| SSH public key source | Generate new key pair |
| Key Type | RSA SSH Format |

> **Best Practice:** Always use **Generate new key pair** to ensure Azure controls the cryptographic material. Uploading an existing key from a local machine creates an Azure resource record but offers no additional control-plane guarantees on key provenance.

---

### Step 4: Review Configuration and Submit

After completing the Basics tab, click **Next: Tags >** to optionally add resource tags (recommended for cost allocation and environment labeling), then click **Next: Review + create >**.

The portal performs a final ARM validation. A green **Validation passed** banner confirms the configuration is complete and deployable.

Review the following summary values before proceeding:

| Field | Value |
|---|---|
| Subscription | Azure Free Labs |
| Resource Group | kml_rg_main-513d23304c80408c |
| Region | East US |
| Key pair name | devops-kp |
| SSH Key format | RSA |

Click **Create** to initiate deployment.

![Review and Create tab showing all configuration values and a Validation passed banner](https://github.com/user-attachments/assets/969681b1-babd-42dc-a5fc-ee842c6ee10c)
*Figure 3: Review + create tab showing all configuration values and a Validation passed confirmation banner prior to deployment.*

---

### Step 5: Download the Private Key and Create the Resource

Immediately upon clicking **Create**, Azure displays the **Generate new key pair** dialog. This is the **only opportunity** to download the private key. The dialog explicitly states:

> *"Azure doesn't store the private key. After the SSH key resource is created, you won't be able to download the private key again."*

**Action Required:**

- Click **Download private key and create resource** to save the `.pem` file to a secure local directory
- Azure simultaneously finalizes the resource deployment and triggers the private key download

> **Warning:** If you close or dismiss this dialog without downloading, the private key is permanently lost. You would need to delete and recreate the resource.

> **Post-Download:** Set the downloaded `.pem` file to read-only for the owner immediately:
> ```bash
> chmod 400 devops-kp.pem
> ```

![The Generate new key pair modal confirming Azure does not retain the private key](https://github.com/user-attachments/assets/a70469d7-bcbb-40ec-9830-fd25af60ab70)
*Figure 4: The "Generate new key pair" modal confirming Azure does not retain the private key. The operator must download it at this moment.*

---

### Step 6: Verify SSH Key Resource Creation

After the download completes, Azure finalizes deployment and redirects to the resource overview page. To independently confirm the resource exists:

- Navigate back to **Home > SSH keys** via the search bar or breadcrumb
- Confirm **devops-kp** appears in the resource list with the correct resource group, region, and key type
- Click the resource entry to view the full resource blade and confirm all properties match the specification

Validation is complete when the resource record is visible in the SSH Keys service and displays the correct key pair name, subscription, and region.

---

## Validation Outcome

| Check | Result |
|---|---|
| Resource Name | devops-kp |
| Key Type | RSA |
| Azure Resource | Registered as `Microsoft.Compute/sshPublicKeys` |
| Private Key | Downloaded and secured locally (.pem) |
| Validation Status | **PASSED** - Resource confirmed in Azure control plane |

---

## Risks, Edge Cases, and Troubleshooting

| Risk / Issue | Mitigation / Resolution |
|---|---|
| Private key not downloaded at creation | Delete the `sshPublicKeys` resource and recreate. There is no recovery path. |
| Wrong subscription context | Verify subscription in the portal header before starting the wizard. Use the directory and subscription filter. |
| Resource group does not exist | Create the resource group in the target region before provisioning the key. |
| Naming mismatch with automation | Confirm key pair name matches downstream automation expectations exactly. Azure names are case-sensitive. |
| RBAC permissions error | Ensure your account has at least Contributor on the resource group. Contact your Azure administrator if the Create button is greyed out. |
| Local key upload (wrong source) | Using "Upload existing public key" creates a record but does not use Azure-generated cryptographic material. Use "Generate new key pair" for full control-plane compliance. |

---

## Operational Best Practices

- Store the downloaded `.pem` file in a secrets manager (e.g., Azure Key Vault, HashiCorp Vault, or a secure secrets store) immediately after download
- Apply `chmod 400` to the private key file on any Unix-based workstation to prevent unauthorized reads
- Tag the SSH key resource with **environment** (e.g., `dev`, `staging`, `prod`) and **owner** metadata for cost tracking and lifecycle management
- Use a consistent naming convention such as `<project>-<environment>-kp` to support automated validation and discovery
- Rotate SSH key pairs periodically in alignment with your organization's key rotation policy; decommission old keys by deleting the Azure resource and updating associated VM configurations
- Avoid reusing the same key pair across multiple environments to limit the blast radius of a potential key compromise

---

## Key Learnings

- **Azure control-plane registration is mandatory:** Azure-managed SSH keys must be provisioned through the Azure control plane to appear as verifiable resources. Local key generation does not satisfy control-plane validation requirements.
- **The private key download is a one-time event:** There is a single, non-repeatable download window at creation time. Operational readiness for secure storage must be confirmed before initiating the wizard.
- **The Azure Portal is the authoritative provisioning interface:** It enforces validation, applies RBAC, and integrates with Azure Monitor and Activity Log automatically.
- **Naming precision matters:** Correct and consistent key naming is critical when downstream automation relies on resource discovery by name.
- **Azure retains only the public key:** The operator is solely responsible for the security, backup, and lifecycle of the private key.

---

## Conclusion

This runbook demonstrates that provisioning SSH key pairs via the Azure Portal is the correct, compliant, and operationally sound method for teams operating within Azure-governed environments. By using the Azure Portal's built-in SSH key provisioning flow, engineers ensure that cryptographic resources are registered in the Azure control plane, subject to RBAC, and visible to auditing and monitoring tooling.

The procedure described in this document establishes a repeatable, enterprise-grade pattern for SSH key lifecycle management. Teams onboarding to Azure infrastructure should adopt this approach as the default standard for all SSH key provisioning activities.

---
e ensures compatibility, security, and compliance with Azure-managed workflows.
