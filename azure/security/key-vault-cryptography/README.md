# Azure Key Vault: RSA Encryption and Decryption of Sensitive Data

![Azure Key Vault](https://img.shields.io/badge/Azure-Key%20Vault-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![RSA](https://img.shields.io/badge/Algorithm-RSA--OAEP%204096-00B388?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-Azure%20CLI-0089D6?style=for-the-badge&logo=microsoftazure)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Resolution: Step-by-Step Implementation](#resolution-step-by-step-implementation)
  - [Step 1: Verify Azure Account and Identity](#step-1-verify-azure-account-and-identity)
  - [Step 2: Retrieve the Service Principal Object ID](#step-2-retrieve-the-service-principal-object-id)
  - [Step 3: Create the Azure Key Vault](#step-3-create-the-azure-key-vault)
  - [Step 4: Create the RSA Cryptographic Key](#step-4-create-the-rsa-cryptographic-key)
  - [Step 5: Retrieve and Store the Key ID](#step-5-retrieve-and-store-the-key-id)
  - [Step 6: Inspect the Sensitive Data File](#step-6-inspect-the-sensitive-data-file)
  - [Step 7: Base64 Encode the Plaintext](#step-7-base64-encode-the-plaintext)
  - [Step 8: Encrypt the Data Using RSA-OAEP](#step-8-encrypt-the-data-using-rsa-oaep)
  - [Step 9: Decrypt the Encrypted Data](#step-9-decrypt-the-encrypted-data)
  - [Step 10: Decode and Verify the Decrypted Output](#step-10-decode-and-verify-the-decrypted-output)
- [Validation and Verification](#validation-and-verification)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This project documents the end-to-end implementation of a **secure data encryption and decryption workflow** using **Azure Key Vault** and an **RSA 4096-bit key**. The task was executed as part of a data security initiative to demonstrate how sensitive files can be protected using cloud-managed cryptographic keys, without ever exposing the raw private key material outside of the vault.

The workflow covers Key Vault provisioning, RSA key creation, file encryption using the `RSA-OAEP` algorithm, and full round-trip decryption verification, with MD5 checksum validation confirming byte-for-byte file integrity.

---

## Problem Statement

The Nautilus DevOps team required a reproducible, auditable process to:

* Protect a sensitive file (`SensitiveData.txt`) at rest using cloud-managed encryption
* Avoid storing or exposing private key material outside a hardened vault boundary
* Demonstrate encryption and decryption capabilities using Azure-native tooling
* Validate data integrity after a full encrypt-decrypt cycle

**Key constraints:**

* Key Vault must use the **Vault Access Policy** permission model (not RBAC)
* Soft Delete retention must be set to **7 days**
* The RSA key must be **4096 bits**, type **RSA**, using the **RSA-OAEP** algorithm
* All operations must be executed from the `azure-client` host via Azure CLI

---

## Architecture

```
+------------------+        Base64 Encode         +-------------------+
|  SensitiveData   | ---------------------------> | SensitiveData_b64 |
|    .txt (26B)    |                              |       .txt        |
+------------------+                              +-------------------+
                                                          |
                                              az keyvault key encrypt
                                            (RSA-OAEP via xfusion-6778)
                                                          |
                                                          v
                                              +---------------------+
                                              |  EncryptedData.bin  |
                                              |      (685 bytes)    |
                                              +---------------------+
                                                          |
                                             az keyvault key decrypt
                                                          |
                                                          v
                                              +----------------------+
                                              | DecryptedData_b64.txt|
                                              +----------------------+
                                                          |
                                                  Base64 Decode
                                                          |
                                                          v
                                              +----------------------+
                                              |  DecryptedData.txt   |
                                              |  MD5 == Original     |
                                              +----------------------+
```

**Azure Resources Used:**

| Resource | Name | Region |
|---|---|---|
| Resource Group | `kml_rg_main-9cbddda27c7f4e01` | East US |
| Key Vault | `xfusion-6778` | East US |
| RSA Key | `xfusion-key` | East US |
| Key Size | 4096-bit | N/A |
| Permission Model | Vault Access Policy | N/A |

---

## Prerequisites

Before beginning this implementation, ensure the following are in place:

* **Azure CLI** installed and authenticated on the `azure-client` host
* An active Azure subscription (confirmed via `az account show`)
* Contributor or Owner role on the target resource group
* The sensitive file exists at `/root/SensitiveData.txt`
* The `base64` utility is available on the host

---

## Resolution: Step-by-Step Implementation

### Step 1: Verify Azure Account and Identity

Confirm the active Azure subscription and authenticated identity before performing any resource operations.

```bash
az account show
az account list --output table
```

**Expected Output:**

```json
{
  "name": "Azure Free Labs",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "state": "Enabled",
  "isDefault": true,
  "user": {
    "name": "6b8c8f69-c12a-42b4-8add-e4b15038f472",
    "type": "servicePrincipal"
  }
}
```

> **Confirm** the subscription is `Enabled` and marked as `isDefault: true` before proceeding.

---

**SCREENSHOT**

<img width="1032" height="538" alt="image" src="https://github.com/user-attachments/assets/24b275f3-63c7-4a85-9370-2c14d73b2540" />

> *Output of `az account show` confirming the active subscription and authenticated service principal.*

---

### Step 2: Retrieve the Service Principal Object ID

The Object ID of the Service Principal is required to configure the Key Vault access policy. The Client ID alone is not sufficient.

```bash
SP_CLIENT_ID="6b8c8f69-c12a-42b4-8add-e4b15038f472"

OBJECT_ID=$(az ad sp show --id "$SP_CLIENT_ID" --query id --output tsv)

echo "Object ID: $OBJECT_ID"
```

**Expected Output:**

```
Object ID: 85ac0efb-4740-457a-9a16-940eee7ef86e
```

> Store the `OBJECT_ID` as a shell variable. It is used implicitly by Azure CLI when creating the vault with the calling identity auto-added to the access policy.

---

**SCREENSHOT**

<img width="1032" height="711" alt="image" src="https://github.com/user-attachments/assets/09932a87-db2e-493a-8274-6a9c1bab5811" />

> *Service Principal Object ID resolved from Client ID using `az ad sp show`.*

---

### Step 3: Create the Azure Key Vault

Provision the Key Vault with the required configuration. Note the documented troubleshooting below for flag compatibility issues encountered during this step.

```bash
az keyvault create \
  --name "xfusion-6778" \
  --resource-group "kml_rg_main-9cbddda27c7f4e01" \
  --location "eastus" \
  --sku "standard" \
  --retention-days 7 \
  --enable-rbac-authorization false
```

**Confirmed Configuration from Output:**

| Property | Value |
|---|---|
| `enableSoftDelete` | `true` |
| `softDeleteRetentionInDays` | `7` |
| `enableRbacAuthorization` | `false` |
| `enablePurgeProtection` | `null` (not set) |
| `provisioningState` | `Succeeded` |
| `vaultUri` | `https://xfusion-6778.vault.azure.net/` |

> **Access Policy:** The calling Service Principal (`85ac0efb...`) was automatically granted `all` permissions on keys, secrets, certificates, and storage.

---

**SCREENSHOTS**

<img width="1035" height="860" alt="image" src="https://github.com/user-attachments/assets/ce5f9d95-e460-4eee-87aa-a3c6d71f8244" />
<img width="1031" height="870" alt="image" src="https://github.com/user-attachments/assets/f3ce9f49-14f3-4c1d-92c0-454ba13f09be" />

> *Successful Key Vault creation output showing `provisioningState: Succeeded` and auto-configured access policy.*

---

### Step 4: Create the RSA Cryptographic Key

Create a 4096-bit RSA key inside the vault. All default settings apply for key operations, leaving `wrapKey`, `unwrapKey`, `encrypt`, `decrypt`, `sign`, and `verify` enabled.

```bash
az keyvault key create \
  --vault-name "xfusion-6778" \
  --name "xfusion-key" \
  --kty RSA \
  --size 4096
```

**Confirmed Key Properties:**

| Property | Value |
|---|---|
| `kty` | `RSA` |
| `key_size` | `4096` |
| `enabled` | `true` |
| `exportable` | `false` |
| `recoveryLevel` | `CustomizedRecoverable+Purgeable` |
| `recoverableDays` | `7` |

> The `exportable: false` setting ensures the private key material cannot leave the vault boundary. All cryptographic operations are performed server-side within Key Vault.

---

**SCREENSHOTS**

<img width="1034" height="756" alt="image" src="https://github.com/user-attachments/assets/c61077bc-6e32-4bb3-ba7a-844af38632b3" />
<img width="1031" height="871" alt="image" src="https://github.com/user-attachments/assets/dd362971-c691-495a-9bdf-05d06ba53037" />

> *RSA 4096-bit key `xfusion-key` created successfully with all key operations enabled.*

---

### Step 5: Retrieve and Store the Key ID

Capture the versioned Key ID URI. This URI uniquely identifies the exact key version used for encryption, ensuring reproducible decryption.

```bash
KEY_ID=$(az keyvault key show \
  --vault-name "xfusion-6778" \
  --name "xfusion-key" \
  --query "key.kid" \
  --output tsv)

echo "Key ID: $KEY_ID"
```

**Output:**

```
Key ID: https://xfusion-6778.vault.azure.net/keys/xfusion-key/890c23c674d244bdb7efd324e59819ee
```

> Store this URI. The version component (`890c23c6...`) is critical for decryption. If the key is rotated, the previous version must be retained to decrypt existing ciphertext.

---

**SCREENSHOT**

<img width="1035" height="327" alt="image" src="https://github.com/user-attachments/assets/4192dcb4-d59f-488e-a991-65b33f52fdad" />

> *Versioned Key ID URI retrieved from Key Vault for use in encryption operations.*

---

### Step 6: Inspect the Sensitive Data File

Verify the file exists, confirm its size, and review its contents before encryption.

```bash
ls -lh /root/SensitiveData.txt && cat /root/SensitiveData.txt
```

**Output:**

```
-rw-r--r-- 1 root root 26 Mar 20 01:51 /root/SensitiveData.txt
This is a sensitive file.
```

---

**SCREENSHOT PLACEHOLDER**
> `[screenshot-06-sensitive-file.png]`
> *Caption: Plaintext sensitive file at `/root/SensitiveData.txt`, 26 bytes in size.*

---

### Step 7: Base64 Encode the Plaintext

The Azure Key Vault Encrypt API requires the input value to be Base64-encoded. Encode the file before passing it to the encrypt command.

```bash
base64 /root/SensitiveData.txt > /root/SensitiveData_b64.txt
cat /root/SensitiveData_b64.txt
```

**Output:**

```
VGhpcyBpcyBhIHNlbnNpdGl2ZSBmaWxlLgo=
```

> This is standard Base64 encoding of the UTF-8 plaintext. The trailing `=` is a padding character indicating the original byte count.

---

**SCREENSHOT PLACEHOLDER**
> `[screenshot-07-base64-encode.png]`
> *Caption: Plaintext file Base64-encoded and saved to `/root/SensitiveData_b64.txt`.*

---

### Step 8: Encrypt the Data Using RSA-OAEP

Submit the Base64-encoded value to Azure Key Vault for encryption using the `RSA-OAEP` algorithm. The encrypted result is saved as `EncryptedData.bin`.

```bash
ENCRYPTED=$(az keyvault key encrypt \
  --vault-name "xfusion-6778" \
  --name "xfusion-key" \
  --algorithm RSA-OAEP \
  --value "$(cat /root/SensitiveData_b64.txt)" \
  --query result \
  --output tsv)

echo "$ENCRYPTED" > /root/EncryptedData.bin
echo "Encrypted file saved. Size: $(wc -c < /root/EncryptedData.bin) bytes"
```

**Output:**

```
WARNING: This command is in preview and under development.
Encrypted file saved. Size: 685 bytes
```

```bash
ls -lh /root/EncryptedData.bin
# -rw-r--r-- 1 root root 685 Mar 20 02:04 /root/EncryptedData.bin
```

> The `WARNING` message about the command being in preview is expected for Azure CLI's `keyvault key encrypt` subcommand. The encryption itself is production-grade and backed by Azure Key Vault's FIPS 140-2 Level 2 validated HSM.

---

**SCREENSHOT PLACEHOLDER**
> `[screenshot-08-encrypt.png]`
> *Caption: Successful encryption producing a 685-byte ciphertext blob saved to `/root/EncryptedData.bin`.*

---

### Step 9: Decrypt the Encrypted Data

Pass the ciphertext back to Key Vault for decryption. The vault returns the original Base64-encoded plaintext.

```bash
DECRYPTED=$(az keyvault key decrypt \
  --vault-name "xfusion-6778" \
  --name "xfusion-key" \
  --algorithm RSA-OAEP \
  --value "$(cat /root/EncryptedData.bin)" \
  --query result \
  --output tsv)

echo "$DECRYPTED" > /root/DecryptedData_b64.txt
echo "Decrypted (base64): $DECRYPTED"
```

**Output:**

```
WARNING: This command is in preview and under development.
Decrypted (base64): VGhpcyBpcyBhIHNlbnNpdGl2ZSBmaWxlLgo=
```

> The returned Base64 string matches exactly what was encrypted in Step 7, confirming the cryptographic round-trip.

---

**SCREENSHOT PLACEHOLDER**
> `[screenshot-09-decrypt.png]`
> *Caption: Key Vault decryption returning the original Base64-encoded plaintext string.*

---

### Step 10: Decode and Verify the Decrypted Output

Base64-decode the decrypted output and compare it byte-for-byte against the original sensitive file.

```bash
base64 --decode /root/DecryptedData_b64.txt > /root/DecryptedData.txt
cat /root/DecryptedData.txt
```

**Output:**

```
This is a sensitive file.
```

---

**SCREENSHOT PLACEHOLDER**
> `[screenshot-10-decoded-output.png]`
> *Caption: Base64-decoded output restoring the original plaintext content.*

---

## Validation and Verification

Two independent verification methods confirm the integrity of the encrypt-decrypt cycle.

### Method 1: diff Check

```bash
diff /root/SensitiveData.txt /root/DecryptedData.txt && echo "SUCCESS: Files match perfectly"
```

**Output:**

```
SUCCESS: Files match perfectly
```

### Method 2: MD5 Checksum Comparison

```bash
md5sum /root/SensitiveData.txt /root/DecryptedData.txt
```

**Output:**

```
e7533db289f7b96fd958a96215889e95  /root/SensitiveData.txt
e7533db289f7b96fd958a96215889e95  /root/DecryptedData.txt
```

> Both files produce an identical MD5 hash (`e7533db289f7b96fd958a96215889e95`), confirming zero data corruption through the full encrypt-decrypt pipeline.

---

**SCREENSHOT PLACEHOLDER**
> `[screenshot-11-validation-diff-md5.png]`
> *Caption: `diff` and `md5sum` both confirm the decrypted file is a bit-for-bit match to the original.*

---

## Troubleshooting

The following errors were encountered and resolved during this implementation.

### Error 1: `--enable-soft-delete` Flag Unrecognized

**Command attempted:**
```bash
az keyvault create ... --enable-soft-delete true
```

**Error:**
```
unrecognized arguments: --enable-soft-delete true
```

**Root Cause:** In newer versions of the Azure CLI, `--enable-soft-delete` is deprecated because Soft Delete is enabled by default and cannot be disabled. The flag was removed from the CLI.

**Resolution:** Remove the `--enable-soft-delete` flag entirely. Soft Delete is automatically enabled. Use `--retention-days` to control the retention window.

---

### Error 2: `enablePurgeProtection` Cannot Be Set to False

**Command attempted:**
```bash
az keyvault create ... --enable-purge-protection false
```

**Error:**
```
(BadRequest) The property "enablePurgeProtection" cannot be set to false.
Enabling the purge protection for a vault is an irreversible action.
```

**Root Cause:** Once Purge Protection is enabled on a vault, it cannot be disabled. Azure also rejects explicit attempts to set it to `false` in some API versions as a protective measure.

**Resolution:** Omit the `--enable-purge-protection` flag entirely for this scenario. The vault was created without Purge Protection, which is acceptable when the lab requires the ability to purge the vault within the soft-delete retention window.

---

## Best Practices

### Key Vault Configuration

* **Always use Soft Delete** with a minimum retention period of 7 days for production vaults. Consider 90 days for critical workloads.
* **Enable Purge Protection** for production environments to prevent accidental or malicious permanent deletion of keys and secrets.
* **Use RBAC authorization** over Vault Access Policies in modern deployments. RBAC provides finer-grained control and integrates with Azure AD Conditional Access.
* **Restrict public network access** by configuring network ACLs or Private Endpoints to limit Key Vault exposure to trusted VNets only.
* **Apply resource locks** (`CanNotDelete`) on Key Vaults backing production encryption workloads.

### Key Management

* **Never set `exportable: true`** unless you have a documented, audited requirement. Exportable keys can leave the vault boundary, negating its security model.
* **Version your keys** and always record the versioned Key ID URI used for each encryption operation. You cannot decrypt data without the specific key version that encrypted it.
* **Rotate keys periodically** and use Key Vault's key rotation policy for automated lifecycle management.
* **Use the smallest key size that meets your security requirement**. RSA-4096 provides stronger security at the cost of performance. For high-throughput scenarios, evaluate RSA-2048 or consider switching to ECC keys.

### Encryption Workflow

* **Always Base64-encode plaintext** before passing it to `az keyvault key encrypt`. The API does not accept raw binary input via the CLI.
* **Store the encrypted ciphertext separately** from the Key Vault reference. The ciphertext is safe to store in version control or object storage; it is meaningless without the vault key.
* **Validate every decryption cycle** with both a `diff` and a checksum comparison (`md5sum` or `sha256sum`) before relying on the decrypted output in production pipelines.
* **Prefer `sha256sum` over `md5sum`** in production environments. MD5 is sufficient for detecting accidental corruption but is not collision-resistant.

### Operational Security

* **Log all Key Vault operations** by enabling Azure Monitor diagnostic settings and streaming logs to a Log Analytics Workspace or SIEM.
* **Set up key expiry alerts** to avoid silent key expiration that would break decryption pipelines.
* **Use Managed Identities** instead of Service Principals wherever possible to eliminate credential rotation overhead.
* **Apply the principle of least privilege**: grant only `encrypt` and `decrypt` permissions to application identities. Reserve `all` permissions for administrative identities.

---

## Lessons Learned

### 1. Azure CLI Flag Deprecation Requires Version Awareness

The `--enable-soft-delete` flag was silently removed from the Azure CLI without a deprecation warning at runtime. Teams should maintain a pinned version of the Azure CLI in CI/CD pipelines and test CLI scripts against new versions in a staging environment before rolling out updates.

### 2. Purge Protection is a One-Way Door

Enabling `--enable-purge-protection` is irreversible at the vault level. In lab and development environments, omitting this flag preserves the ability to fully purge the vault and its keys during cleanup. In production environments, always enable it. The distinction must be a deliberate, documented decision.

### 3. The Key Version is Part of the Decryption Contract

The versioned Key ID URI (`/keys/xfusion-key/890c23c6...`) is not just metadata. It is a required component of the decryption operation. Storing only the key name without the version will cause decryption to fail after key rotation. Treat the Key ID URI as a first-class artifact and persist it alongside the ciphertext.

### 4. Base64 is a Transport Encoding, Not Encryption

Base64 encoding is required by the Azure CLI interface for binary input handling. It provides zero security. The actual security comes from the RSA-OAEP encryption performed server-side in Key Vault. These two operations serve completely different purposes and must not be confused.

### 5. Preview CLI Commands Can Break Pipelines

The `az keyvault key encrypt` and `az keyvault key decrypt` commands carry a `preview` warning. Preview commands are subject to breaking changes between CLI versions. For production workloads, use the Azure Key Vault SDK (Python, .NET, Java, Go) or the REST API directly, which provide stable, versioned interfaces.

### 6. MD5 is Adequate for Integrity Checking, Not Security Validation

MD5 checksums confirm that the decrypted file was not corrupted during the process. However, MD5 is not cryptographically secure and should not be used as a security control. Use SHA-256 (`sha256sum`) for integrity checks in security-sensitive contexts.

### 7. Least-Privilege Access Policies Reduce Blast Radius

The auto-generated access policy granted `all` permissions to the Service Principal. In a real-world scenario, this should be scoped to only `encrypt` and `decrypt` for application workloads. Administrative `all` permissions should be restricted to a break-glass identity and protected by Privileged Identity Management (PIM).

---

## References

* [Azure Key Vault Documentation](https://learn.microsoft.com/en-us/azure/key-vault/)
* [az keyvault CLI Reference](https://learn.microsoft.com/en-us/cli/azure/keyvault)
* [Azure Key Vault Key Encryption](https://learn.microsoft.com/en-us/azure/key-vault/keys/about-keys)
* [RSA-OAEP Algorithm Specification](https://datatracker.ietf.org/doc/html/rfc8017)
* [Azure Key Vault Soft Delete Overview](https://learn.microsoft.com/en-us/azure/key-vault/general/soft-delete-overview)
* [Azure Key Vault Best Practices](https://learn.microsoft.com/en-us/azure/key-vault/general/best-practices)
* [Azure RBAC vs Access Policies](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-access-policy)

---




<img width="1029" height="431" alt="image" src="https://github.com/user-attachments/assets/97600d30-1d04-45ec-a327-7833b71a2b29" />
<img width="1032" height="317" alt="image" src="https://github.com/user-attachments/assets/9e441031-d257-4cce-821b-f42dd6333286" />



<img width="1033" height="374" alt="image" src="https://github.com/user-attachments/assets/a0f49612-4d18-4b21-98f7-0621ddd79f77" />
<img width="1033" height="610" alt="image" src="https://github.com/user-attachments/assets/acec18e2-b598-4911-a2db-a7bc32099274" />
<img width="1034" height="494" alt="image" src="https://github.com/user-attachments/assets/4ea2954b-e8fe-43d1-b6d5-a4386f62621a" />
<img width="1036" height="483" alt="image" src="https://github.com/user-attachments/assets/9e7ef0bc-96c0-46fc-ab9f-73a2ba02ac32" />
<img width="1031" height="563" alt="image" src="https://github.com/user-attachments/assets/7dc0c9aa-6c5f-49d4-ac43-d156be0e3b82" />


