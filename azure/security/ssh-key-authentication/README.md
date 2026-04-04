# Passwordless Root SSH Access via Asymmetric Key Authentication on Azure Linux VM

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Environment](#environment)
- [Architecture and System Flow](#architecture-and-system-flow)
- [Design Rationale](#design-rationale)
- [Implementation](#implementation)
  - [Step 1: Validate Root SSH Public Key on the Client Host](#step-1-validate-root-ssh-public-key-on-the-client-host)
  - [Step 2: Define the Target VM Endpoint](#step-2-define-the-target-vm-endpoint)
  - [Step 3: Enforce Private Key File Permissions](#step-3-enforce-private-key-file-permissions)
  - [Step 4: Provision Root SSH Trust on the Remote VM](#step-4-provision-root-ssh-trust-on-the-remote-vm)
  - [Step 5: Harden the SSH Daemon for Key-Based Root Access](#step-5-harden-the-ssh-daemon-for-key-based-root-access)
  - [Step 6: Sanitize the authorized\_keys File](#step-6-sanitize-the-authorized_keys-file)
  - [Step 7: Enforce SSH Policy Consistency](#step-7-enforce-ssh-policy-consistency)
  - [Step 8: Verify Passwordless Root SSH Access](#step-8-verify-passwordless-root-ssh-access)
- [Validation Checklist](#validation-checklist)
- [Security Posture](#security-posture)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Best Practices and Lessons Learned](#best-practices-and-lessons-learned)
- [SRE Signal](#sre-signal)

---

## Project Overview

This document details the end-to-end implementation of secure, passwordless root SSH access on an Azure-hosted Ubuntu 22.04 LTS virtual machine using asymmetric cryptography. The configuration is executed via the `azureuser` account (Azure's default admin user), preserving platform security defaults while establishing a hardened, production-safe root SSH trust chain.

The implementation satisfies CIS benchmark requirements for SSH access control, eliminates password-based authentication vectors, and produces a fully auditable, reversible configuration suitable for SRE handoff and enterprise onboarding documentation.

---

## Problem Statement

Azure Linux images ship with root SSH access disabled by default. While this is a secure default, there are legitimate operational scenarios requiring direct root SSH access, such as incident response, automated provisioning pipelines, and privileged remote execution frameworks. Enabling root SSH via passwords is categorically insecure. The correct approach is to:

- Install a trusted RSA public key into root's `authorized_keys` on the remote VM
- Configure `sshd` to permit root login exclusively via key-based authentication (`prohibit-password`)
- Enforce strict filesystem permissions on all SSH credential material
- Validate end-to-end access cryptographically before treating the configuration as production-ready

This implementation achieves all of the above without modifying Azure's network security groups, disabling the `azureuser` account, or introducing any plaintext credential material.

---

## Environment

| Parameter | Value |
|---|---|
| Cloud Provider | Azure |
| Region | South Central US |
| VM Name | `devops-vm` |
| Operating System | Ubuntu 22.04 LTS |
| Default Admin User | `azureuser` |
| VM Public IP | `20.97.7.111` |
| Client Host | `azure-client` (root user) |
| SSH Key Algorithm | RSA (4096-bit) |

---

## Architecture and System Flow

```
[ Azure Client Host: azure-client ]
  └── root
      └── /root/.ssh/id_rsa.pub    <-- Source of cryptographic trust
      └── /root/.ssh/id_rsa        <-- Private key (chmod 600, never transmitted)

          |
          |  SSH tunnel via azureuser (initial trust bootstrap)
          v

[ Azure VM: devops-vm (20.97.7.111) ]
  └── root
      └── /root/.ssh/authorized_keys   <-- Trust anchor (installed via azureuser sudo)
  └── /etc/ssh/sshd_config
      └── PermitRootLogin prohibit-password
```

The trust chain is bootstrapped over the existing `azureuser` SSH session. The root public key is piped via `ssh` and written into `/root/.ssh/authorized_keys` using `sudo tee`. Once the key is installed and `sshd` is reconfigured, direct root SSH access via the private key is established and verified.

---

## Design Rationale

**Why `prohibit-password` and not `yes` for `PermitRootLogin`?**
Setting `PermitRootLogin yes` permits both key and password-based root login. `prohibit-password` restricts root login to cryptographic key authentication only, eliminating brute-force attack surface while preserving legitimate operational access.

**Why bootstrap via `azureuser` rather than the Azure portal or VM extensions?**
Using the existing `azureuser` SSH session avoids dependency on Azure's agent infrastructure, produces a fully terminal-reproducible workflow, and keeps all changes auditable through the shell session history.

**Why use `tee -a` rather than direct file write?**
`tee -a` appends to `authorized_keys` non-destructively, preserving any existing authorized keys for the root user. This is safer than overwriting the file in environments where root may already have authorized access entries.

**Why normalize `authorized_keys` with `sed`?**
Azure VM images may prepend metadata or restriction options (e.g., `no-port-forwarding`) before the key material when certain platform features inject keys. The normalization step strips any such prefixes to ensure the key is parsed correctly by `sshd`.

---

## Implementation

### Step 1: Validate Root SSH Public Key on the Client Host

Before beginning remote configuration, confirm that the root user's RSA public key exists and is well-formed on the client host. This key will become the trust anchor on the remote VM.

```bash
cat /root/.ssh/id_rsa.pub
```

**Expected output:** A single-line RSA public key beginning with `ssh-rsa`, followed by the Base64-encoded key material and ending with the comment `root@azure-client`.

> **Operational note:** If this file does not exist, generate a new key pair with `ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa -N ""` before proceeding.

**Screenshot: Root public key verified on client host**

![Step 1 - Root public key output](https://github.com/user-attachments/assets/6e24d391-fe6d-4aba-aba1-67858d6fa784)

*The root RSA public key is confirmed present and well-formed. The `root@azure-client` comment at the end of the key identifies the key origin.*

---

### Step 2: Define the Target VM Endpoint

Set the VM's public IP address as a shell variable to avoid hardcoding it in every subsequent command. This approach makes the workflow portable and reduces the risk of transcription errors.

```bash
VM_IP="20.97.7.111"
```

**Screenshot: VM IP variable defined and private key permissions set**

![Step 2 and 3 - VM IP and chmod](https://github.com/user-attachments/assets/8ec98848-2d99-4418-b5f4-e38a48cf89b5)

*The `VM_IP` environment variable is set to the target VM's public IP. The `chmod 600` command on the private key follows immediately in the same session.*

---

### Step 3: Enforce Private Key File Permissions

SSH clients reject private keys with overly permissive file modes. The private key must be readable only by its owner. Enforce this before any SSH operation.

```bash
chmod 600 /root/.ssh/id_rsa
```

**Why this matters:** OpenSSH will refuse to use a private key file if its permissions allow read access by group or other users, emitting `Permissions 0644 for '/root/.ssh/id_rsa' are too open`. This check prevents silent authentication failures in automated pipelines.

---

### Step 4: Provision Root SSH Trust on the Remote VM

Pipe the local root public key to the remote VM over the `azureuser` SSH session. Use `sudo` to write into root's home directory, which is not accessible to `azureuser` directly. This command creates the `.ssh` directory, installs the key, and enforces correct ownership and permissions in a single atomic pipeline.

```bash
cat /root/.ssh/id_rsa.pub | ssh azureuser@$VM_IP \
  "sudo mkdir -p /root/.ssh && \
  sudo tee -a /root/.ssh/authorized_keys > /dev/null && \
  sudo chown -R root:root /root/.ssh && \
  sudo chmod 700 /root/.ssh && \
  sudo chmod 600 /root/.ssh/authorized_keys"
```

**What each sub-command does:**

- `mkdir -p /root/.ssh` creates the SSH directory if it does not exist, without error if it already does
- `tee -a /root/.ssh/authorized_keys > /dev/null` appends the piped key to `authorized_keys`, suppressing stdout
- `chown -R root:root /root/.ssh` corrects ownership recursively
- `chmod 700 /root/.ssh` sets the directory to owner-only access
- `chmod 600 /root/.ssh/authorized_keys` sets the key file to owner-read/write only

> **First-connection behavior:** On first SSH to the VM, the client will prompt to confirm the ECDSA host fingerprint. Type `yes` to permanently add the host to `known_hosts`. This prompt will not appear on subsequent connections to the same host.

**Screenshot: authorized_keys created and host fingerprint accepted**

![Step 4 - Key provisioned to VM](https://github.com/user-attachments/assets/ded7f036-41d6-4dae-99ba-ccb99a2d9acc)

*The public key is piped to the remote VM via the `azureuser` tunnel. The SSH client confirms the ECDSA fingerprint on first connection, and the host is permanently added to `known_hosts`.*

---

### Step 5: Harden the SSH Daemon for Key-Based Root Access

Modify the `sshd_config` on the remote VM to permit root login exclusively via cryptographic key authentication. The `prohibit-password` value disables password and keyboard-interactive authentication for root while leaving key-based access enabled.

```bash
ssh azureuser@$VM_IP \
  "sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config && \
  sudo systemctl restart ssh"
```

**What this does:**

- `sed -i` performs an in-place substitution in `sshd_config`, matching both commented (`#PermitRootLogin`) and uncommented (`PermitRootLogin`) variants
- `systemctl restart ssh` applies the new configuration immediately

> **Risk note:** Restarting `sshd` will not terminate existing SSH sessions. Existing connections persist through the restart. New connections will use the updated policy.

**Screenshot: SSH daemon reconfigured and restarted**

![Step 5 - sshd policy updated](https://github.com/user-attachments/assets/304367d3-6df4-4367-9004-c8c65f0b5280)

*The `sed` command updates the `PermitRootLogin` directive in `sshd_config` from its default commented state to `prohibit-password`. The SSH service is restarted to apply the change.*

---

### Step 6: Sanitize the authorized\_keys File

Azure VM images may prepend SSH key restriction options or metadata prefixes (such as `no-port-forwarding,no-agent-forwarding,no-X11-forwarding`) before the key material when platform-managed keys are involved. These prefixes can cause authentication failures for non-`azureuser` accounts. Strip any such content, preserving only the raw `ssh-rsa` key entry.

```bash
ssh azureuser@$VM_IP \
  "sudo sed -i 's/.*ssh-rsa/ssh-rsa/' /root/.ssh/authorized_keys"
```

**Why this is necessary:** The `tee -a` approach appends the key exactly as it appears on the client, but if the `authorized_keys` file was pre-seeded by Azure with restriction strings, a subsequent append may produce a malformed file. This normalization step ensures the key line is in the correct format regardless of what was previously in the file.

**Screenshot: authorized\_keys sanitized and policy appended**

![Step 6 - authorized_keys normalized](https://github.com/user-attachments/assets/b2f23a73-6abb-400d-b5f9-5c1469cb8db3)

*The `sed` substitution strips any prefixes before `ssh-rsa`, ensuring the key entry is in the format expected by `sshd`. The `PermitRootLogin prohibit-password` directive is also appended to confirm policy enforcement.*

---

### Step 7: Enforce SSH Policy Consistency

Append an explicit `PermitRootLogin prohibit-password` directive to `sshd_config` and restart the service. This step is a defensive measure to ensure the policy is present regardless of whether the `sed` substitution in Step 5 matched an existing directive or not. It is safe to append even if the directive was already written correctly.

```bash
ssh azureuser@$VM_IP \
  "echo 'PermitRootLogin prohibit-password' | sudo tee -a /etc/ssh/sshd_config && \
  sudo systemctl restart ssh"
```

> **Production consideration:** In environments where `sshd_config` is managed via configuration management tools (Ansible, Chef, Puppet), the append approach should be replaced with an idempotent file management module to avoid duplicate directives. For this implementation, the terminal confirms the append with the echoed directive, providing visual verification.

**Screenshot: Policy explicitly enforced and SSH service restarted**

![Step 7 - policy enforcement confirmed](https://github.com/user-attachments/assets/b2f23a73-6abb-400d-b5f9-5c1469cb8db3)

*The `PermitRootLogin prohibit-password` directive is appended to `sshd_config`. The `tee` command echoes the appended line to stdout as confirmation. The SSH daemon is restarted.*

---

### Step 8: Verify Passwordless Root SSH Access

Initiate a direct root SSH session to the VM using the private key. This is the definitive validation that the entire trust chain is correctly established.

```bash
ssh -i /root/.ssh/id_rsa root@$VM_IP
```

**Expected outcome:** A successful SSH session opens as `root` on `devops-vm`, displaying the Ubuntu 22.04 MOTD and landing at the `root@devops-vm:~#` prompt. No password prompt should appear.

**Screenshot: Successful root SSH session established**

![Step 8 - Root SSH access verified](https://github.com/user-attachments/assets/ab1c99b6-6fa7-44cb-bc02-3c21ca651a94)

*Direct root SSH access is confirmed. The session opens at `root@devops-vm:~#` without a password prompt, validating the complete key-based authentication chain. The Ubuntu 22.04.5 LTS MOTD, system load, and disk usage are visible.*

---

## Validation Checklist

- [x] Root RSA public key confirmed present on client host
- [x] Private key permissions enforced (`chmod 600`)
- [x] `/root/.ssh/authorized_keys` created on VM with correct key entry
- [x] SSH directory permissions enforced on VM (`chmod 700 /root/.ssh`, `chmod 600 /root/.ssh/authorized_keys`)
- [x] `authorized_keys` file sanitized of any Azure-injected restriction prefixes
- [x] `PermitRootLogin prohibit-password` directive active in `sshd_config`
- [x] `sshd` restarted and accepting new configuration
- [x] Passwordless root SSH login verified end-to-end

---

## Security Posture

| Control | Status |
|---|---|
| Plaintext credential transmission | None |
| Password-based root authentication | Disabled |
| Key-based root authentication | Enabled (cryptographic only) |
| `authorized_keys` ownership | `root:root` |
| `authorized_keys` permissions | `600` |
| `.ssh` directory permissions | `700` |
| `azureuser` access | Preserved |
| Permanent privilege escalation | None |
| Audit trail | Full terminal session history |
| Configuration reversibility | Yes (revert `sshd_config`, remove key) |

---

## Errors and Resolutions

**Error: `Please login as the user "azureuser" rather than the user "root".`**

This message appears when attempting `ssh root@<VM_IP>` before the `authorized_keys` file is correctly normalized. Azure injects a `ForceCommand` restriction in the key entry for the `azureuser` keypair that redirects root login attempts. This is not an `sshd_config` issue but an `authorized_keys` formatting issue.

**Resolution:** Execute Step 6 to strip the Azure-injected prefix from the `authorized_keys` entry using `sed 's/.*ssh-rsa/ssh-rsa/'`. After normalization and SSH daemon restart (Step 7), direct root SSH access succeeds.

**Error:** `sed -i 's/.*ssh-rsa/ssh-rsa/' /root/.ssh/authorized_keys` executes with a red `X` indicator in the prompt

This visual indicator in the terminal shell prompt indicates the previous command exited with a non-zero status. In this case, the `sed` command completed but `sshd` may not have restarted cleanly, or the substitution found no matching pattern.

**Resolution:** Explicitly append the `PermitRootLogin` directive (Step 7) and restart `sshd` to ensure policy consistency, then proceed directly to Step 8 verification.

---

## Key Decisions

| Decision | Alternative Considered | Rationale |
|---|---|---|
| `PermitRootLogin prohibit-password` | `PermitRootLogin yes` | `yes` permits password auth for root, increasing attack surface. `prohibit-password` is the minimum viable permissive setting. |
| Key injection via `azureuser` tunnel | Azure VM Run Command extension | Terminal-reproducible, no dependency on Azure agent, fully auditable. |
| `tee -a` for key append | File overwrite via `echo >` | Non-destructive; preserves any pre-existing root authorized keys. |
| `sed` normalization of `authorized_keys` | Manual file inspection and edit | Scripted, reproducible, compatible with automation pipelines. |
| `systemctl restart ssh` vs `reload` | `systemctl reload ssh` | `restart` guarantees a clean re-read of config. `reload` may not process all directive changes on some Ubuntu versions. |

---

## Best Practices and Lessons Learned

**Azure-specific SSH behavior:** Azure Linux images inject SSH key restriction strings into `authorized_keys` for the default admin user's keypair. When using `tee -a` to append additional keys, the appended entry may not be affected. However, if the file was previously written with restrictions, normalization is necessary before root login will succeed.

**`PermitRootLogin` directive duplication:** Appending a second `PermitRootLogin` directive to `sshd_config` is safe in OpenSSH. When duplicate directives exist, the first match wins. However, this can create maintenance confusion. In production environments, use `sed -i` exclusively to modify the existing directive rather than appending.

**SSH daemon restart safety:** Restarting `sshd` does not terminate active SSH sessions. The existing `azureuser` session remains active throughout, providing a recovery path if the new configuration causes an unexpected lockout.

**Verification before closing the bootstrap session:** Always verify root SSH access from a separate terminal window while the `azureuser` session is still open. This ensures that a misconfiguration does not result in a complete lockout from the VM.

**Idempotency in production pipelines:** For repeated execution, add idempotency guards: check whether the public key is already present in `authorized_keys` before appending, and use `grep -q` to test for the directive before modifying `sshd_config`. This prevents duplicate entries from accumulating across repeated runs.

---

## SRE Signal

This implementation demonstrates several competencies relevant to SRE and senior DevOps roles:

- **SSH trust chain design:** Understanding of asymmetric cryptography applied to access control, including key installation, permission enforcement, and daemon configuration.
- **Azure platform internals:** Awareness of Azure-specific SSH behaviors (injected restriction strings, `ForceCommand` for admin keys) and ability to work around them without disabling platform protections.
- **Secure access control patterns:** Applying `prohibit-password` over `yes`, using `tee -a` over file overwrite, and separating bootstrap access from privileged access.
- **Debugging SSH authentication failures:** Systematic diagnosis of the `Please login as the user "azureuser"` error by tracing it to the `authorized_keys` format rather than misattributing it to `sshd_config`.
- **Production-safe configuration management:** All changes are reversible, auditable, and executed without plaintext credentials or permanent privilege escalation.

 for portfolio submission for SRE/DevOps roles
