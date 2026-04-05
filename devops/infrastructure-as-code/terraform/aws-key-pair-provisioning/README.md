# AWS EC2 Key Pair Provisioning via Terraform

Automated generation and registration of an RSA 4096-bit EC2 key pair using Terraform, with deterministic local private key persistence and cryptographic validation against the AWS control plane.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Architecture and Design Intent](#architecture-and-design-intent)
* [Prerequisites](#prerequisites)
* [Implementation](#implementation)
  * [Phase 1: Terraform Configuration Authoring](#phase-1-terraform-configuration-authoring)
  * [Phase 2: Provider Initialization](#phase-2-provider-initialization)
  * [Phase 3: Infrastructure Provisioning](#phase-3-infrastructure-provisioning)
  * [Phase 4: Validation and State Verification](#phase-4-validation-and-state-verification)
* [Resource Inventory](#resource-inventory)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)

---

## Project Overview

The Nautilus DevOps team is executing a phased migration of its infrastructure to AWS. Rather than a single large-scale cutover, the team has adopted an incremental approach, decomposing the migration into discrete, independently verifiable units of work.

This deliverable addresses the foundational compute access layer: provisioning an RSA EC2 key pair using Terraform. The key pair is required before any EC2 instances can be launched and accessed over SSH. All key material is generated locally by the `hashicorp/tls` provider and registered with AWS EC2, ensuring the private key never transits the AWS API.

**Task requirements:**

* Key pair name: `nautilus-kp`
* Key type: RSA (4096-bit)
* Private key saved to: `/home/bob/nautilus-kp.pem`
* Terraform working directory: `/home/bob/terraform`
* Single configuration file: `main.tf`

---

## Architecture and Design Intent

```
+--------------------------+
|  hashicorp/tls provider  |
|  RSA 4096-bit key gen    |
+----------+---------------+
           |
           | public_key_openssh
           v
+--------------------------+        +-----------------------------+
|  aws_key_pair resource   +------->|  AWS EC2 Key Pair Registry  |
|  key_name: nautilus-kp   |        |  key-d7b36c72a78a73f09      |
+--------------------------+        +-----------------------------+
           |
           | private_key_pem
           v
+--------------------------+
|  local_file resource     |
|  /home/bob/nautilus-kp   |
|  .pem (mode 0400)        |
+--------------------------+
```

The `tls_private_key` resource acts as the single source of truth for key material. The public key is extracted via `public_key_openssh` and registered with AWS. The private key is written to disk via `local_file` with strict `0400` permissions, preventing any read access by users other than the file owner.

This pattern keeps private key material entirely within the local execution environment. AWS receives only the public key, which is consistent with asymmetric cryptography best practices for SSH authentication.

---

## Prerequisites

| Requirement | Version Used |
|---|---|
| Terraform CLI | >= 1.0 |
| AWS CLI configured | `us-east-1` region |
| AWS provider | `hashicorp/aws v5.91.0` |
| TLS provider | `hashicorp/tls v4.2.1` |
| Local provider | `hashicorp/local v2.8.0` |
| Working directory | `/home/bob/terraform` |

Ensure the AWS credentials environment (`💠 default`) is active before running any Terraform commands.

---

## Implementation

### Phase 1: Terraform Configuration Authoring

Navigate to the Terraform working directory and author `main.tf` using a heredoc to prevent shell interpretation of HCL syntax:

```bash
cd ~/terraform
```

```bash
cat > main.tf << 'EOF'
resource "tls_private_key" "nautilus_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "nautilus_kp" {
  key_name   = "nautilus-kp"
  public_key = tls_private_key.nautilus_key.public_key_openssh
}

resource "local_file" "private_key" {
  content         = tls_private_key.nautilus_key.private_key_pem
  filename        = "/home/bob/nautilus-kp.pem"
  file_permission = "0400"
}
EOF
```

Verify the file was written correctly:

```bash
cat main.tf
```

*Screenshot: main.tf contents displayed in terminal confirming all three resource blocks*

<img width="1230" height="724" alt="image" src="https://github.com/user-attachments/assets/bda574a9-455c-4900-9487-3c9e60257288" />

**Resource breakdown:**

* `tls_private_key.nautilus_key` generates an RSA 4096-bit key pair entirely in Terraform memory. Both the public and private keys are exposed as output attributes.
* `aws_key_pair.nautilus_kp` registers the public key with the AWS EC2 service under the name `nautilus-kp`. The `public_key` attribute is populated at plan time via interpolation.
* `local_file.private_key` writes the PEM-encoded private key to `/home/bob/nautilus-kp.pem` with `0400` permissions, restricting access to the file owner only.

---

### Phase 2: Provider Initialization

Initialize the Terraform working directory to download and lock the required provider plugins:

```bash
terraform init
```

Expected output confirms provider resolution and installation:

```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "5.91.0"...
- Finding latest version of hashicorp/tls...
- Finding latest version of hashicorp/local...
- Installing hashicorp/aws v5.91.0...
- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)
- Installing hashicorp/tls v4.2.1...
- Installed hashicorp/tls v4.2.1 (signed by HashiCorp)
- Installing hashicorp/local v2.8.0...
- Installed hashicorp/local v2.8.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

*Screenshot: terraform init output showing successful provider installations and lock file creation*

The `.terraform.lock.hcl` file is generated at this step, pinning exact provider versions for reproducible runs. This file should be committed to version control.

---

### Phase 3: Infrastructure Provisioning

Apply the configuration without an interactive approval prompt:

```bash
terraform apply -auto-approve
```

Terraform produces an execution plan followed by sequential resource creation:

```
Plan: 3 to add, 0 to change, 0 to destroy.

tls_private_key.nautilus_key: Creating...
tls_private_key.nautilus_key: Creation complete after 4s [id=676a44929afac2f675c4c4ad8dfb76887951c179]
local_file.private_key: Creating...
local_file.private_key: Creation complete after 0s [id=098853ea4c082fa3e46297a955fc7f19e8ecb0b6]
aws_key_pair.nautilus_kp: Creating...
aws_key_pair.nautilus_kp: Creation complete after 0s [id=nautilus-kp]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

*Screenshot: Full terraform apply output with plan summary and resource creation completion messages*

**Creation order observed:**

1. `tls_private_key.nautilus_key` is created first (4 seconds) as no dependencies exist on other resources.
2. `local_file.private_key` is created immediately after, as it depends only on the TLS key.
3. `aws_key_pair.nautilus_kp` is created last, registering the public key with the EC2 API.

Terraform resolves the dependency graph automatically based on interpolation references, ensuring correct ordering without explicit `depends_on` declarations.

---

### Phase 4: Validation and State Verification

**4.1 Confirm key pair registration in Terraform state:**

```bash
terraform show | grep key_name
```

Expected output:

```
key_name        = "nautilus-kp"
key_name_prefix = null
```

*Screenshot: grep output confirming key_name value in Terraform state*

**4.2 Verify private key file permissions and existence:**

```bash
ls -la /home/bob/nautilus-kp.pem
```

Expected output:

```
-r-------- 1 bob bob 3243 Apr  5 00:14 /home/bob/nautilus-kp.pem
```

The `0400` permission set (`-r--------`) confirms owner-only read access with no write or execute bits set for any user.

*Screenshot: ls -la output showing nautilus-kp.pem with 0400 permissions*

**4.3 Verify private key file format:**

```bash
head -2 /home/bob/nautilus-kp.pem
```

Expected output:

```
-----BEGIN RSA PRIVATE KEY-----
MIIJKQIBAAKCAgEAyYl8hMEu5hXcnG7RZsSDb6QiToL7cXvzEH7ajbcMD7Mn3g+t
```

The `-----BEGIN RSA PRIVATE KEY-----` header confirms a valid PKCS#1-encoded RSA private key, compatible with standard OpenSSH clients.

*Screenshot: head -2 output confirming RSA private key PEM header*

**4.4 Inspect full AWS key pair state:**

```bash
terraform state show aws_key_pair.nautilus_kp
```

Expected output:

```hcl
# aws_key_pair.nautilus_kp:
resource "aws_key_pair" "nautilus_kp" {
    arn             = "arn:aws:ec2:us-east-1::key-pair/nautilus-kp"
    fingerprint     = "f7:4e:11:e6:c8:fe:be:b3:77:3f:29:2d:7c:4b:b3:b1"
    id              = "nautilus-kp"
    key_name        = "nautilus-kp"
    key_name_prefix = null
    key_pair_id     = "key-d7b36c72a78a73f09"
    key_type        = null
    public_key      = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQAB..."
    tags_all        = {}
}
```

*Screenshot: terraform state show output displaying full resource attributes including ARN, fingerprint, and key_pair_id*

The `key_pair_id` (`key-d7b36c72a78a73f09`) confirms successful registration with the AWS EC2 control plane. The MD5 fingerprint can be cross-referenced in the AWS Console under EC2 > Key Pairs.

---

## Resource Inventory

| Terraform Resource | AWS Resource Type | Identifier |
|---|---|---|
| `tls_private_key.nautilus_key` | Local (in-memory) | `676a449...` |
| `aws_key_pair.nautilus_kp` | AWS EC2 Key Pair | `key-d7b36c72a78a73f09` |
| `local_file.private_key` | Local file | `/home/bob/nautilus-kp.pem` |

---

## Best Practices Applied

**Private key locality:** The `hashicorp/tls` provider generates key material within the Terraform process. The private key is never transmitted to or stored by AWS. Only the public key is registered with the EC2 API. This is the recommended pattern for key pair management when operator control of the private key is required.

**Strict file permissions:** The `file_permission = "0400"` attribute on `local_file.private_key` enforces owner-only read access. This mirrors the behavior required by OpenSSH, which refuses to use private key files with overly permissive modes.

**Heredoc configuration authoring:** Using `cat > main.tf << 'EOF'` with a quoted delimiter prevents shell variable expansion inside the heredoc body. This is essential when HCL contains `${}` interpolation syntax that would otherwise be misinterpreted by the shell.

**State inspection over console verification:** Using `terraform state show` and `terraform show` for validation rather than relying solely on the AWS Console ensures that Terraform's internal state is consistent with the actual infrastructure, catching any drift at the Terraform layer.

**Single-file configuration scope:** The task constraint of using only `main.tf` without splitting into `variables.tf` or `outputs.tf` is appropriate for single-resource modules. Splitting configuration files adds overhead that is only justified at module scale or when values need to be parameterized across environments.

**Lock file version pinning:** The `.terraform.lock.hcl` file generated during `terraform init` pins exact provider versions (`aws v5.91.0`, `tls v4.2.1`, `local v2.8.0`). Committing this file to version control ensures that every team member and CI pipeline uses identical provider binaries, eliminating version-drift failures.

---

## Lessons Learned

**RSA key generation is compute-intensive at higher bit lengths.** The `tls_private_key` resource with `rsa_bits = 4096` took 4 seconds to complete, which is normal. At 2048 bits this would be near-instantaneous. For production environments where key provisioning is a one-time bootstrap operation, 4096-bit RSA is the appropriate choice. For ephemeral or short-lived infrastructure, 2048-bit or Ed25519 keys offer equivalent security with faster generation.

**Terraform dependency resolution is implicit, not explicit.** No `depends_on` block was needed in this configuration. Terraform inferred the correct creation order from the interpolation references: `aws_key_pair` references `tls_private_key.nautilus_key.public_key_openssh`, so Terraform knows to create the TLS resource first. Understanding implicit dependency resolution prevents unnecessary `depends_on` declarations that add complexity without benefit.

**The `local_file` resource is not idempotent in the traditional sense.** If the file is deleted outside of Terraform, the next `terraform plan` will show it as needing recreation because Terraform tracks its hash in state. This is expected behavior. For key files that must survive Terraform state loss, the PEM content should be backed up to a secrets manager such as AWS Secrets Manager or HashiCorp Vault immediately after provisioning.

**Sensitive values in plan output.** The `terraform apply` plan correctly masked the private key content with `(sensitive value)` in the plan output. This is handled automatically by the `tls` provider, which marks `private_key_pem` and related attributes as sensitive. Sensitive attribute masking prevents accidental key exposure in CI logs or terminal recordings.

**Key pair ARN does not include an account ID segment in this environment.** The ARN returned was `arn:aws:ec2:us-east-1::key-pair/nautilus-kp` (double colon, no account ID). This is expected for EC2 key pairs in sandbox environments where the IAM account context may be scoped differently. In production accounts, the ARN will include the twelve-digit account ID between the region and resource segments.


<img width="1257" height="559" alt="image" src="https://github.com/user-attachments/assets/5aea0b43-18c2-451f-a1ca-2d27797bffac" />

<img width="1227" height="769" alt="image" src="https://github.com/user-attachments/assets/d82c3683-0ddb-45d3-bb9f-14023cb2ccf0" />
<img width="1407" height="775" alt="image" src="https://github.com/user-attachments/assets/8cb7c1cd-a9c1-45df-acd0-e76a35e65ce5" />
<img width="1401" height="773" alt="image" src="https://github.com/user-attachments/assets/dfa99929-66c7-4c3e-ad03-6ee47c14738e" />
<img width="1239" height="280" alt="image" src="https://github.com/user-attachments/assets/adea2578-8de3-4c50-839b-6cc04d7a781a" />
<img width="1246" height="592" alt="image" src="https://github.com/user-attachments/assets/3816f74a-6df5-4ba0-9ef0-f1002bfbd0a7" />
