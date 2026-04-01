# Azure Security

![Azure](https://img.shields.io/badge/Azure-Security-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![RSA](https://img.shields.io/badge/Cryptography-RSA--OAEP%204096-00B388?style=for-the-badge)
![SSH](https://img.shields.io/badge/Access-SSH%20Key%20Auth-F05032?style=for-the-badge&logo=openssh&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

---

## Overview

This directory covers cloud security implementations on Azure, focusing on two foundational controls that appear in every production security baseline: **cryptographic key management** and **hardened remote access**.

Both projects reflect real-world constraints, including policy-restricted environments, Azure CLI version compatibility, and the principle of least privilege. The work is drawn from hands-on KodeKloud/Nautilus lab scenarios and documented to production standards.

---

## Directory Structure

```
azure/security/
├── key-vault-cryptography/     # RSA-OAEP encryption and decryption using Azure Key Vault
├── ssh-key-authentication/     # Password-less root SSH access on an Azure Linux VM
└── README.md                   # This file
```

---

## Project Summaries

### [key-vault-cryptography](./key-vault-cryptography/)

**Quick Summary:** Provisioned an Azure Key Vault and used a 4096-bit RSA key to encrypt and decrypt a sensitive file, keeping all private key material server-side and validating integrity with MD5 checksum comparison.

| | |
|---|---|
| **Purpose** | Protect a sensitive file at rest using cloud-managed cryptography, without exposing private key material outside the vault boundary. |
| **Approach** | Created the Key Vault with Vault Access Policy (not RBAC), generated a non-exportable RSA-4096 key, Base64-encoded the plaintext per CLI requirements, and ran a full encrypt-decrypt round-trip using `RSA-OAEP`. |
| **Key Decisions** | Omitted `--enable-purge-protection` intentionally to allow lab cleanup; documented the irreversible nature of that flag for production contexts. Captured the versioned Key ID URI alongside ciphertext to preserve decryption capability across key rotations. |
| **Outcome** | `diff` and `md5sum` confirmed byte-for-byte file integrity. All operations executed server-side within the vault. Ciphertext size: 685 bytes from a 26-byte input, consistent with RSA-4096 block size behavior. |

---

### [ssh-key-authentication](./ssh-key-authentication/)

**Quick Summary:** Configured password-less root SSH access on an Azure Ubuntu 22.04 VM by injecting a public key into `/root/.ssh/authorized_keys` and hardening the SSH daemon to enforce key-only root login.

| | |
|---|---|
| **Purpose** | Establish cryptographic root access to an Azure VM while disabling password-based login and preserving Azure's default `azureuser` admin access. |
| **Approach** | Piped the landing host's public key through `azureuser` to the VM's root trust store via a single compound SSH command, then configured `sshd_config` with `PermitRootLogin prohibit-password` and restarted the daemon. A `sed` pass sanitized `authorized_keys` to strip Azure metadata prefixes. |
| **Key Decisions** | Used `prohibit-password` rather than `yes` to satisfy both security posture and Azure image defaults. Preserved `azureuser` access throughout to avoid lockout. |
| **Outcome** | Verified direct root SSH access (`root@devops-vm:~#`) with no password prompt. No credentials stored, no permanent privilege escalation, and all changes are fully reversible. |

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Cloud Platform | Azure (Key Vault, Virtual Machines) |
| CLI | Azure CLI (`az keyvault`, `az ad sp`) |
| Cryptography | RSA-4096, RSA-OAEP, Base64, MD5 |
| Access Control | SSH key pairs, `sshd_config`, `authorized_keys` |
| Scripting | Bash, `sed`, `base64`, `md5sum`, `diff` |
| OS | Ubuntu 22.04 LTS |

---

## Key Outcomes

- Provisioned and configured Azure Key Vault under a Vault Access Policy permission model with 7-day soft delete
- Executed server-side RSA-OAEP encryption and decryption with zero private key exposure
- Applied versioned Key ID URI discipline to preserve decryption capability across key rotation
- Hardened SSH daemon to enforce key-based root authentication (`prohibit-password`) on a live Azure VM
- Identified and resolved two Azure CLI breaking changes (`--enable-soft-delete` removal, `enablePurgeProtection` immutability)
- Applied least-privilege access patterns across both implementations

---

## How to Navigate

Each subdirectory contains a self-contained `README.md` with:

- Full CLI commands and expected output
- Screenshots at every verification step
- A dedicated Troubleshooting section covering real errors encountered
- Best Practices and Lessons Learned sections scoped to production applicability

Start with the project `README.md` files for implementation detail. This file is intended as an index and summary layer.

---

> Part of the [`cloud-infrastructure-devops-labs`](../../) portfolio by [@arinze-edeh](https://github.com/arinze-edeh).
