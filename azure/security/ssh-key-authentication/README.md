# Secure Password-less Root SSH Access on Azure VM (devops-vm)

## 📌 Project Overview

## OBJECTIVE:
- Provision secure, password-less root SSH access to an Azure Linux VM
using asymmetric cryptography, while preserving Azure default security
controls and least-privilege access patterns.

## Environment:

- Cloud Provider: `Azure`

- Region: `southcentralus`

- VM Name: `devops-vm`

- OS: `Ubuntu 22.04 LTS`

- Default Admin User: `azureuser`

## Design Rationale

### RATIONALE:
- Root login via SSH passwords is insecure and disabled by default
- SSH key-based authentication provides cryptographic identity assurance
- Azure images restrict root SSH by default; configuration must be explicit
- All changes must be auditable, reversible, and production-safe

### Implementation ensures:

- Key-based authentication only

- Azure admin access preserved

- CIS & cloud security best practices followed

### 🧩 System Flow
- [ Azure Client / Landing Host ]
-  └── root
-      └── /root/.ssh/id_rsa.pub   → source of trust

- [ Azure VM: devops-vm ]
-  └── root
-      └── /root/.ssh/authorized_keys  ← trust anchor

## Step 1: Validate Root SSH Public Key (Landing Host)
- `cat /root/.ssh/id_rsa.pub`

📸 Screenshot: `root public key verified`

## Step 2: Define Target VM Endpoint
`VM_IP="20.97.7.111"`

📸 Screenshot: `vm ip defined`

## Step 3: Enforce Private Key Security
`chmod 600 /root/.ssh/id_rsa`

📸 Screenshot:`private key permissions`

## Step 4: Provision Root SSH Trust on VM

- `cat /root/.ssh/id_rsa.pub | ssh azureuser@$VM_IP \`
- `"sudo mkdir -p /root/.ssh && \`
- `sudo tee -a /root/.ssh/authorized_keys > /dev/null && \`
- `sudo chown -R root:root /root/.ssh && \`
- `sudo chmod 700 /root/.ssh && \`
- `sudo chmod 600 /root/.ssh/authorized_keys"`

📸 Screenshot: `authorized_keys created`

## Step 5: Configure SSH Daemon (Key-Based Root Access Only)
- `ssh azureuser@$VM_IP \`
- `"sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config && \`
- `sudo systemctl restart ssh"`

📸 Screenshot: `sshd root policy`

## Step 6: Normalize authorized_keys
- `ssh azureuser@$VM_IP \`
- `"sudo sed -i 's/.*ssh-rsa/ssh-rsa/' /root/.ssh/authorized_keys"`

📸 Screenshot: `authorized_keys sanitized`

## Step 7 — Enforce SSH Policy Consistency
- `ssh azureuser@$VM_IP \`
- `"echo 'PermitRootLogin prohibit-password' | sudo tee -a /etc/ssh/sshd_config && \`
- `sudo systemctl restart ssh"`

📸 Screenshot: `policy enforced`

## Step 8: Verify Password-less Root SSH Access
- `ssh -i /root/.ssh/id_rsa root@20.97.7.111`

- Expected Result:

- `root@devops-vm:~#`

📸 Screenshot:`root ssh success`

## Validation Checklist

- Root public key installed correctly
- Strict permissions enforced (700 / 600)
- SSH daemon hardened
- Password authentication disabled
- Root SSH access verified cryptographically

## Security Posture

- No plaintext credentials used
- No password-based root access
- No permanent privilege escalation
- Fully auditable changes
- Cloud-provider defaults respected

## SRE Signal

- Demonstrates secure access control design
- Highlights SSH key management and troubleshooting
- Shows cloud image security constraints
- Illustrates production-grade operational hygiene
- Suitable for portfolio submission for SRE/DevOps roles







<img width="1035" height="331" alt="image" src="https://github.com/user-attachments/assets/6e24d391-fe6d-4aba-aba1-67858d6fa784" />
<img width="1033" height="597" alt="image" src="https://github.com/user-attachments/assets/8ec98848-2d99-4418-b5f4-e38a48cf89b5" />
<img width="1031" height="585" alt="image" src="https://github.com/user-attachments/assets/ded7f036-41d6-4dae-99ba-ccb99a2d9acc" />
<img width="1034" height="728" alt="image" src="https://github.com/user-attachments/assets/304367d3-6df4-4367-9004-c8c65f0b5280" />
<img width="1031" height="786" alt="image" src="https://github.com/user-attachments/assets/b2f23a73-6abb-400d-b5f9-5c1469cb8db3" />
<img width="1034" height="860" alt="image" src="https://github.com/user-attachments/assets/ab1c99b6-6fa7-44cb-bc02-3c21ca651a94" />








