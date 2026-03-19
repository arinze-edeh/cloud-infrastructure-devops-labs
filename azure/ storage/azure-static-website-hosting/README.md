# Hosting a Static Website on Azure Blob Storage

![Azure](https://img.shields.io/badge/Azure-Storage-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Region](https://img.shields.io/badge/Region-East%20US-blue?style=for-the-badge)
![CLI](https://img.shields.io/badge/Tool-Azure%20CLI-0078D4?style=for-the-badge&logo=microsoftazure)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Task Requirements](#task-requirements)
- [Step-by-Step Resolution](#step-by-step-resolution)
  - [Step 1: Verify Azure Account and Resource Group](#step-1-verify-azure-account-and-resource-group)
  - [Step 2: Create the Azure Storage Account](#step-2-create-the-azure-storage-account)
  - [Step 3: Enable Static Website Hosting](#step-3-enable-static-website-hosting)
  - [Step 4: Allow Public Blob Access](#step-4-allow-public-blob-access)
  - [Step 5: Retrieve the Storage Account Key](#step-5-retrieve-the-storage-account-key)
  - [Step 6: Upload the Static Website File](#step-6-upload-the-static-website-file)
  - [Step 7: Verify Public Accessibility](#step-7-verify-public-accessibility)
- [Validation and Output](#validation-and-output)
- [Resource Summary](#resource-summary)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Problem Statement

The **Nautilus DevOps** team was tasked with creating an internal information portal accessible to the public. The requirement was to host a static website on **Microsoft Azure** using an **Azure Blob Storage Account**, configured for public access so that external users can access the site directly via the Azure Storage static website URL.

This is a common real-world scenario in enterprise environments where lightweight, highly available static content (documentation portals, status pages, landing pages) must be served without provisioning a full web server or container infrastructure.

---

## Architecture Overview

```
Developer Workstation (Azure CLI)
          |
          | az storage blob upload
          v
+-------------------------------+
|   Azure Storage Account       |
|   devopswebst23591            |
|   SKU: Standard_LRS           |
|   Region: East US             |
|                               |
|  +--------------------------+ |
|  |  $web Container          | |
|  |  (Static Website Host)   | |
|  |  index.html              | |
|  +--------------------------+ |
+-------------------------------+
          |
          | HTTPS (Public)
          v
https://devopswebst23591.z13.web.core.windows.net/
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| Azure CLI | Installed and authenticated |
| Azure Subscription | Active subscription with contributor access |
| Resource Group | Pre-existing resource group in East US |
| Static HTML File | `index.html` present at `/root/` on the Azure client host |
| Permissions | Storage Account Contributor or Owner role |

---

## Task Requirements

The following objectives were defined for this lab:

1. Create an Azure Storage Account named **`devopswebst23591`** inside the existing resource group.
2. Configure the Storage Account for **static website hosting** with `index.html` as the index document.
3. **Allow public access** to the static website so external users can access it.
4. Upload the `index.html` file from the `/root/` directory on the Azure client host to the storage account's **`$web`** container.
5. Verify that the website is accessible directly through the **Azure Storage static website URL**.

---

## Step-by-Step Resolution

### Step 1: Verify Azure Account and Resource Group

Before provisioning any resources, confirm the active subscription context and identify the target resource group.

```bash
az account show
```

**Expected Output (Key Fields):**

```json
{
  "name": "Azure Free Labs",
  "state": "Enabled",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552"
}
```

List available resource groups to confirm the target group exists:

```bash
az group list --output table
```

**Expected Output:**

```
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-21feedb32ed4471f  eastus      Succeeded
```

> **Why this matters:** Deploying into the wrong subscription or resource group is one of the most common and costly mistakes in cloud environments. Always verify context before provisioning.

---

**SCREENSHOT:** 

<img width="1030" height="563" alt="image" src="https://github.com/user-attachments/assets/f460eb4c-2034-4eb2-aabf-4e66dade540a" />

>Terminal output showing `az account show` and `az group list --output table` confirming the active subscription and resource group in `East US`

---

### Step 2: Create the Azure Storage Account

Create a new **StorageV2** general-purpose storage account with **Standard_LRS** (Locally Redundant Storage) replication in the East US region.

```bash
az storage account create \
  --name devopswebst23591 \
  --resource-group kml_rg_main-21feedb32ed4471f \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

**Key Parameters Explained:**

| Parameter | Value | Reason |
|---|---|---|
| `--name` | `devopswebst23591` | Globally unique storage account name |
| `--sku` | `Standard_LRS` | Cost-effective redundancy for lab/non-critical workloads |
| `--kind` | `StorageV2` | Required for static website hosting feature |
| `--location` | `eastus` | Matches existing resource group region |

**Confirm provisioning state in the response:**

```json
{
  "provisioningState": "Succeeded",
  "primaryEndpoints": {
    "web": "https://devopswebst23591.z13.web.core.windows.net/"
  }
}
```

> **Note:** The `web` endpoint is only active after static website hosting is explicitly enabled in the next step.

---

**SCREENSHOTS:** 

<img width="1032" height="865" alt="image" src="https://github.com/user-attachments/assets/6944a481-1b3d-4845-ab19-a3cd0b3b7967" />
<img width="1033" height="868" alt="image" src="https://github.com/user-attachments/assets/9d726326-03ce-4664-8d67-8562672edf01" />
<img width="1029" height="630" alt="image" src="https://github.com/user-attachments/assets/ed21c32c-1f00-4318-855c-f6cd6d87f0ea" />

>Terminal showing the full JSON response from `az storage account create` with `provisioningState: Succeeded` and the `primaryEndpoints.web` URL highlighted

---

### Step 3: Enable Static Website Hosting

Enable the static website feature on the storage account and designate `index.html` as the default index document.

```bash
az storage blob service-properties update \
  --account-name devopswebst23591 \
  --static-website \
  --index-document index.html
```

**Verify the `staticWebsite` block in the response:**

```json
{
  "staticWebsite": {
    "enabled": true,
    "indexDocument": "index.html",
    "errorDocument_404Path": null,
    "defaultIndexDocumentPath": null
  }
}
```

> **Critical Detail:** This command automatically creates the special **`$web`** container in the storage account. All static assets must be uploaded to this container to be served via the website endpoint.

---

**[SCREENSHOT PLACEHOLDER: Terminal output from `az storage blob service-properties update` confirming `staticWebsite.enabled: true` and `indexDocument: index.html`]**

---

### Step 4: Allow Public Blob Access

By default, Azure Storage Accounts created after November 2023 have public blob access disabled. This must be explicitly enabled to allow anonymous public access to the static website.

```bash
az storage account update \
  --name devopswebst23591 \
  --resource-group kml_rg_main-21feedb32ed4471f \
  --allow-blob-public-access true
```

**Confirm in the response:**

```json
{
  "allowBlobPublicAccess": true
}
```

> **Security Context:** In production environments, enabling public blob access requires a deliberate decision aligned with the data classification of the content being served. Static website content that is intentionally public is an appropriate use case.

---

**[SCREENSHOT PLACEHOLDER: Terminal showing `az storage account update` response with `allowBlobPublicAccess: true` confirmed]**

---

### Step 5: Retrieve the Storage Account Key

Retrieve the primary storage account key. This key is used to authenticate the blob upload command in the next step.

```bash
az storage account keys list \
  --account-name devopswebst23591 \
  --resource-group kml_rg_main-21feedb32ed4471f \
  --query "[0].value" \
  --output tsv
```

**Expected Output:**

```
NBdBm8MFJVQlWL4dpvD1Yx9rSDSCO5yJQcPf5yxQSXZC78bOH4Neee18Z1GW2DJvyeVDsNJxhejw+AStW5BHvw==
```

> **Security Warning:** Storage account keys grant full access to all data in the account. In production environments, prefer **Azure AD authentication** (`--auth-mode login`) with RBAC-scoped roles over shared key access. Keys should be rotated regularly and never committed to source control.

---

**[SCREENSHOT PLACEHOLDER: Terminal output from `az storage account keys list` showing the masked or visible key returned in tsv format]**

---

### Step 6: Upload the Static Website File

Upload the `index.html` file from the local `/root/` directory to the `$web` container of the storage account, specifying the correct content type for browser rendering.

```bash
az storage blob upload \
  --account-name devopswebst23591 \
  --account-key "NBdBm8MFJVQlWL4dpvD1Yx9rSDSCO5yJQcPf5yxQSXZC78bOH4Neee18Z1GW2DJvyeVDsNJxhejw+AStW5BHvw==" \
  --container-name '$web' \
  --file /root/index.html \
  --name index.html \
  --content-type "text/html"
```

**Key Parameters Explained:**

| Parameter | Value | Reason |
|---|---|---|
| `--container-name` | `'$web'` | The special container created by static website hosting |
| `--file` | `/root/index.html` | Source file path on the Azure client host |
| `--name` | `index.html` | Blob name must match the configured index document |
| `--content-type` | `text/html` | Ensures browsers render the file correctly, not download it |

**Confirm upload success in the response:**

```json
{
  "lastModified": "2026-03-19T00:56:15+00:00",
  "request_server_encrypted": true,
  "version": "2022-11-02"
}
```

---

**[SCREENSHOT PLACEHOLDER: Terminal showing the `az storage blob upload` progress bar completing at 100% and the JSON response with `lastModified` timestamp confirmed]**

---

### Step 7: Verify Public Accessibility

Perform an HTTP `HEAD` request against the static website endpoint to confirm the site is live and publicly accessible.

```bash
curl -I https://devopswebst23591.z13.web.core.windows.net/
```

**Expected Response (HTTP 200 OK):**

```
HTTP/1.1 200 OK
Content-Length: 56
Content-Type: text/html
Last-Modified: Thu, 19 Mar 2026 00:56:15 GMT
ETag: "0x8DE855252F89B7E"
Server: Windows-Azure-Web/1.0 Microsoft-HTTPAPI/2.0
```

> An `HTTP/1.1 200 OK` with `Content-Type: text/html` confirms the static website is publicly accessible and the index document is being served correctly.

---

**[SCREENSHOT PLACEHOLDER: Terminal showing `curl -I` output with `HTTP/1.1 200 OK`, `Content-Type: text/html`, and the `Last-Modified` timestamp matching the upload time]**

---

## Validation and Output

Open the static website URL in a browser to visually confirm the deployed content.

**Static Website URL:**

```
https://devopswebst23591.z13.web.core.windows.net/
```

---

**[SCREENSHOT PLACEHOLDER: Browser window navigated to `https://devopswebst23591.z13.web.core.windows.net/` displaying the text "Welcome to KKE labs!" confirming successful static website deployment]**

---

The browser renders the uploaded `index.html` file, displaying **"Welcome to KKE labs!"** confirming the full end-to-end deployment is functional.

---

## Resource Summary

| Resource | Value |
|---|---|
| **Subscription** | Azure Free Labs |
| **Resource Group** | `kml_rg_main-21feedb32ed4471f` |
| **Storage Account Name** | `devopswebst23591` |
| **Region** | East US |
| **SKU** | Standard_LRS |
| **Kind** | StorageV2 |
| **Static Website Container** | `$web` |
| **Index Document** | `index.html` |
| **Public Access** | Enabled |
| **Website Endpoint** | `https://devopswebst23591.z13.web.core.windows.net/` |
| **TLS** | Enforced (HTTPS only) |
| **Encryption** | Microsoft-managed keys (enabled at rest) |

---

## Best Practices

### Security

* **Prefer Azure AD authentication over shared key access.** Add `--auth-mode login` to storage commands when the identity has the `Storage Blob Data Contributor` RBAC role assigned. This eliminates the need to retrieve and expose account keys.
* **Rotate storage account keys regularly.** If key-based access is required, use Azure Key Vault to store and rotate keys automatically.
* **Limit public access scope.** Only enable `--allow-blob-public-access true` for storage accounts that are explicitly serving public content. Keep all other storage accounts locked down.
* **Enforce minimum TLS version.** Upgrade `minimumTlsVersion` from `TLS1_0` (the default) to `TLS1_2` for all production workloads:

  ```bash
  az storage account update \
    --name devopswebst23591 \
    --resource-group kml_rg_main-21feedb32ed4471f \
    --min-tls-version TLS1_2
  ```

### Reliability and Performance

* **Use Azure CDN or Azure Front Door** in front of the static website endpoint for production deployments. This provides global edge caching, custom domain support, and SSL certificate management.
* **Implement versioned deployments.** When updating static assets, use versioned filenames (e.g., `index.v2.html`) or integrate with CI/CD pipelines to ensure atomic deployments without stale cache issues.
* **Configure a custom 404 error document** using `--error-document-404-path` for a professional user experience when resources are not found.

### Cost Management

* **Standard_LRS** is the most cost-effective replication tier for non-critical or lab workloads. Evaluate **Standard_GRS** or **Standard_ZRS** for production workloads requiring higher durability guarantees.
* **Monitor storage consumption** using Azure Cost Management and set budget alerts to prevent unexpected charges on free-tier subscriptions.

### Operational Excellence

* **Tag all resources** at creation time to support cost attribution, governance, and automation:

  ```bash
  az storage account create \
    --name devopswebst23591 \
    ... \
    --tags Environment=Lab Project=NautilusPortal Owner=DevOpsTeam
  ```

* **Use infrastructure as code (IaC).** Reproduce this deployment using ARM templates, Bicep, or Terraform to ensure repeatable, auditable, and version-controlled infrastructure provisioning.
* **Always set `--content-type` on blob uploads.** Missing or incorrect content types cause browsers to prompt downloads instead of rendering HTML, CSS, or JavaScript files.

---

## Lessons Learned

### 1. Public Access is Disabled by Default Post-2023

Azure enforces `allowBlobPublicAccess: false` by default on newly created storage accounts. Omitting the `az storage account update --allow-blob-public-access true` step results in a website endpoint that returns `HTTP 409` or serves no content publicly, even when static website hosting is enabled. These two settings are independent and both must be configured.

### 2. The `$web` Container is Not Pre-existing

The `$web` container is created automatically when static website hosting is enabled via `az storage blob service-properties update --static-website`. Attempting to upload to `$web` before enabling this feature will result in a `ContainerNotFound` error. Always enable static website hosting before attempting any uploads.

### 3. `--content-type` is Not Optional

Uploading HTML files without specifying `--content-type "text/html"` causes Azure to default to `application/octet-stream`. Browsers receiving this content type will trigger a file download instead of rendering the page. This is a silent failure that does not produce an error but results in a broken website experience.

### 4. Shared Key vs. Azure AD Authentication

Using `--account-key` directly in CLI commands works but is not secure for production use. The warning message from Azure CLI (`There are no credentials provided in your command...`) is an important signal. Adopt `--auth-mode login` with properly scoped RBAC roles as the standard pattern in CI/CD pipelines and automated deployments.

### 5. TLS Version Defaults Require Hardening

The default `minimumTlsVersion: TLS1_0` on newly created storage accounts is insecure by modern standards. TLS 1.0 and 1.1 have known vulnerabilities and are deprecated by most compliance frameworks (PCI DSS, HIPAA, SOC 2). Explicitly setting `TLS1_2` as the minimum should be a standard post-provisioning step in any hardening runbook.

### 6. Verify with `curl -I` Before Browser Testing

A `curl -I` check against the endpoint gives an immediate, unambiguous signal of the HTTP response code, content type, and server headers without any browser caching, DNS, or rendering layers involved. This is the fastest and most reliable first-pass verification step for any web endpoint deployment.

---

## References

* [Azure Blob Storage Static Website Hosting Documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website)
* [az storage account CLI Reference](https://learn.microsoft.com/en-us/cli/azure/storage/account)
* [az storage blob service-properties CLI Reference](https://learn.microsoft.com/en-us/cli/azure/storage/blob/service-properties)
* [Azure Storage Security Guide](https://learn.microsoft.com/en-us/azure/storage/common/storage-security-guide)
* [Azure Storage RBAC Roles](https://learn.microsoft.com/en-us/azure/storage/common/storage-auth-aad-rbac-cli)
* [Azure CDN Integration with Static Websites](https://learn.microsoft.com/en-us/azure/storage/blobs/static-website-content-delivery-network)

---





<img width="1035" height="851" alt="image" src="https://github.com/user-attachments/assets/d3d0821a-94d2-4a41-9c90-757738600157" />
<img width="1032" height="865" alt="image" src="https://github.com/user-attachments/assets/d52d2926-a25e-41c2-bc24-3ad31c448dc2" />
<img width="1032" height="868" alt="image" src="https://github.com/user-attachments/assets/bcb677a0-bf96-4c79-9bda-2fb9bfbf1a56" />
<img width="1030" height="863" alt="image" src="https://github.com/user-attachments/assets/f7a52b2d-fbdb-4500-8f47-79351799b53c" />
<img width="1032" height="321" alt="image" src="https://github.com/user-attachments/assets/88224ab6-c428-4073-888c-a0fd9f84e475" />
<img width="1032" height="599" alt="image" src="https://github.com/user-attachments/assets/1f4394a4-819d-4f05-9f5c-58221e71d49e" />
<img width="1035" height="858" alt="image" src="https://github.com/user-attachments/assets/74dee2d0-9a9d-4e2a-897b-6cd525ae24d0" />
<img width="1036" height="728" alt="image" src="https://github.com/user-attachments/assets/f24b171a-c505-406e-ac74-dccbbf20e94d" />
<img width="1031" height="303" alt="image" src="https://github.com/user-attachments/assets/dd6b827d-2bb9-472f-9040-8597c83a7461" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/efce9e5c-0354-4247-ba32-cfd43e047e76" />


