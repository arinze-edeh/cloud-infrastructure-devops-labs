# Terraform EC2 Instance Type Modification Using In-Place Update

Modifying a live AWS EC2 instance type from `t2.micro` to `t2.nano` using Terraform's in-place update workflow on a pre-provisioned instance within the Nautilus Datacenter migration environment.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Implementation](#implementation)
  - [Phase 1: Inspect the Working Directory and Current Configuration](#phase-1-inspect-the-working-directory-and-current-configuration)
  - [Phase 2: Modify the Instance Type in main.tf](#phase-2-modify-the-instance-type-in-maintf)
  - [Phase 3: Plan the Infrastructure Change](#phase-3-plan-the-infrastructure-change)
  - [Phase 4: Apply the Infrastructure Change](#phase-4-apply-the-infrastructure-change)
  - [Phase 5: Validate the Applied Change](#phase-5-validate-the-applied-change)
- [Verification](#verification)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Technologies Used](#technologies-used)

---

## Overview

During an ongoing cloud migration, the Nautilus DevOps team identified an EC2 instance running at a higher resource tier than its actual workload required. To optimise cost and resource utilisation, the team downsized the instance from `t2.micro` to `t2.nano`. This change was executed entirely through Terraform's declarative workflow, updating only the existing `main.tf` file and applying the change as an in-place modification without provisioning a new resource.

---

## Problem Statement

The Nautilus DevOps team provisioned several EC2 instances across multiple regions during an active migration. Upon reviewing utilisation metrics, one instance, `datacenter-ec2`, was confirmed to be underutilised. The team required a controlled, auditable method to resize the instance while preserving its existing configuration, networking, and tags. Manual console changes were ruled out to maintain infrastructure-as-code consistency.

**Constraints enforced:**

- The change must be applied only to the existing `main.tf` file located at `/home/bob/terraform`
- No separate `.tf` file may be created for this change
- The instance must remain in `running` state after the modification
- All changes must be driven through Terraform, not direct AWS CLI mutation

---

## Solution Architecture

Terraform tracks the current state of infrastructure in `terraform.tfstate`. When a configuration attribute changes and the provider supports in-place modification for that attribute, Terraform issues an update to the existing resource rather than destroying and recreating it. The AWS provider supports in-place `instance_type` changes, making this a non-disruptive resize operation. The plan-then-apply workflow ensures the change is reviewed before execution.

```
main.tf (instance_type: t2.micro)
        |
        v
   sed in-place edit
        |
        v
main.tf (instance_type: t2.nano)
        |
        v
terraform plan   -->   diff: t2.micro -> t2.nano (1 change, 0 destroy)
        |
        v
terraform apply  -->   aws_instance.ec2 modified in-place
        |
        v
terraform show + aws ec2 describe-instances  -->  validated: t2.nano, running
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Terraform | Installed and initialised in `/home/bob/terraform` |
| AWS CLI | Configured with credentials and region access |
| AWS Provider | Initialised via `.terraform/` and locked in `.terraform.lock.hcl` |
| Existing State | `terraform.tfstate` present with `aws_instance.ec2` tracked |
| Instance Status | EC2 instance must have completed status checks before modification |

---

## Project Structure

```
/home/bob/terraform/
├── .terraform/                  # Provider plugins and modules
├── .terraform.lock.hcl          # Provider version lock file
├── main.tf                      # Primary resource configuration (modified)
├── provider.tf                  # AWS provider and region configuration
├── terraform.tfstate            # Current state of managed infrastructure
└── README.MD                    # Original task reference file
```

---

## Implementation

### Phase 1: Inspect the Working Directory and Current Configuration

Before making any changes, the working directory structure and the current state of `main.tf` were verified to confirm what Terraform was managing and which files were in scope.

```bash
ls -la
```

**Output:**

```
total 44
drwxr-xr-x 1 bob bob 4096 Apr 27 01:18 .
drwxr-x--- 1 bob bob 4096 Apr 27 01:17 ..
drwxr-xr-x 3 bob bob 4096 Apr 27 01:17 .terraform
-rw-r--r-- 1 bob bob 1406 Apr 27 01:17 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  254 Apr 27 01:17 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-r--r-- 1 bob bob 4103 Apr 27 01:18 terraform.tfstate
```

The `terraform.tfstate` file confirmed that an existing instance was already being tracked. The `main.tf` was then inspected to confirm the current instance type:

```bash
cat main.tf
```

**Output:**

```hcl
# Provision EC2 instance
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  subnet_id     = ""
  vpc_security_group_ids = [
    "sg-4f180ca45cbdd1d10"
  ]

  tags = {
    Name = "datacenter-ec2"
  }
}
```

*Screenshot: Working directory listing confirming presence of main.tf, terraform.tfstate, and provider.tf*

<img width="1039" height="644" alt="image" src="https://github.com/user-attachments/assets/e9453574-50a0-4966-a31b-0e4f446a1c93" />

---

### Phase 2: Modify the Instance Type in main.tf

The instance type was changed from `t2.micro` to `t2.nano` using an in-place `sed` substitution directly on `main.tf`. No new `.tf` file was created.

```bash
sed -i 's/instance_type = "t2.micro"/instance_type = "t2.nano"/' /home/bob/terraform/main.tf
```

The file was immediately re-inspected to verify the substitution was applied correctly:

```bash
cat main.tf
```

**Output:**

```hcl
# Provision EC2 instance
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.nano"
  subnet_id     = ""
  vpc_security_group_ids = [
    "sg-4f180ca45cbdd1d10"
  ]

  tags = {
    Name = "datacenter-ec2"
  }
}
```

*Screenshot: Terminal output of cat main.tf confirming instance_type updated to t2.nano*

<img width="1044" height="588" alt="image" src="https://github.com/user-attachments/assets/b6471f69-c48f-4136-a57c-088ce2423ba0" />

---

### Phase 3: Plan the Infrastructure Change

With the configuration updated, `terraform plan` was executed to preview exactly what Terraform intended to change before any modification was applied to the live resource.

```bash
terraform plan
```

**Output:**

```
aws_instance.ec2: Refreshing state... [id=i-6da7f406663e229a8]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_instance.ec2 will be updated in-place
  ~ resource "aws_instance" "ec2" {
        id                                   = "i-6da7f406663e229a8"
      ~ instance_type                        = "t2.micro" -> "t2.nano"
      ~ public_dns                           = "ec2-54-214-93-66.compute-1.amazonaws.com" -> (known after apply)
      ~ public_ip                            = "54.214.93.66" -> (known after apply)
        tags                                 = {
            "Name" = "datacenter-ec2"
        }
        # (34 unchanged attributes hidden)

        # (2 unchanged blocks hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

The plan confirmed:

- **0 resources** to add
- **1 resource** to change (in-place)
- **0 resources** to destroy

The `~` symbol indicates an in-place update, not a replacement. This is the expected behaviour for an `instance_type` change on a stopped-and-restarted instance managed by the AWS provider.

*Screenshot: terraform plan output showing instance_type diff from t2.micro to t2.nano with 0 to destroy*

---

### Phase 4: Apply the Infrastructure Change

After reviewing and confirming the plan output, `terraform apply` was executed. The interactive prompt was confirmed with `yes` to authorise the change.

```bash
terraform apply
```

**Prompt response:**

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

**Apply output:**

```
aws_instance.ec2: Modifying... [id=i-6da7f406663e229a8]
aws_instance.ec2: Still modifying... [id=i-6da7f406663e229a8, 10s elapsed]
aws_instance.ec2: Still modifying... [id=i-6da7f406663e229a8, 20s elapsed]
aws_instance.ec2: Modifications complete after 20s [id=i-6da7f406663e229a8]

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

The modification completed in approximately 20 seconds. The instance ID `i-6da7f406663e229a8` remained unchanged, confirming this was a true in-place resize with no resource replacement.

*Screenshot: terraform apply output showing Modifications complete after 20s with 1 changed and 0 destroyed*

---

### Phase 5: Validate the Applied Change

Two independent validation methods were used to confirm the instance type change and running state.

**Method 1: Terraform State Inspection**

```bash
terraform show | grep -E "instance_type|tags|public_ip"
```

**Output:**

```
    associate_public_ip_address          = true
    instance_type                        = "t2.nano"
    public_ip                            = "54.214.150.90"
    tags                                 = {
    tags_all                             = {
        instance_metadata_tags      = "disabled"
        tags                  = {}
        tags_all              = {}
```

The Terraform state confirmed `instance_type = "t2.nano"`. Note that the public IP changed from `54.214.93.66` to `54.214.150.90` as expected, since EC2 instances assigned dynamic public IPs receive a new IP upon stop and start during a resize operation.

**Method 2: AWS CLI Verification**

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query "Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name]" \
  --output table
```

**Output:**

```
-----------------------------------------------
|              DescribeInstances              |
+----------------------+----------+-----------+
|  i-6da7f406663e229a8 |  t2.nano |  running  |
+----------------------+----------+-----------+
```

The AWS control plane confirmed the instance `i-6da7f406663e229a8` is running as `t2.nano`.

*Screenshot: AWS CLI describe-instances table output showing t2.nano and running state for i-6da7f406663e229a8*

---

## Verification

| Check | Expected | Actual | Status |
|---|---|---|---|
| `main.tf` instance_type | `t2.nano` | `t2.nano` | Passed |
| Terraform plan destroy count | 0 | 0 | Passed |
| Terraform apply result | 1 changed, 0 destroyed | 1 changed, 0 destroyed | Passed |
| `terraform show` instance_type | `t2.nano` | `t2.nano` | Passed |
| AWS CLI instance type | `t2.nano` | `t2.nano` | Passed |
| AWS CLI instance state | `running` | `running` | Passed |
| Instance ID preserved | `i-6da7f406663e229a8` | `i-6da7f406663e229a8` | Passed |

---

## Best Practices Applied

**Plan before apply**
`terraform plan` was always executed before `terraform apply`. This previews the exact diff and catches unintended resource replacements before they reach the live environment.

**Single-file scope**
The change was made exclusively within the existing `main.tf` rather than creating a new `.tf` file. Terraform merges all `.tf` files in a directory; adding unnecessary files increases cognitive overhead and the potential for conflicting declarations.

**In-place sed substitution with verification**
Using `sed -i` for a targeted string substitution is repeatable and auditable. The file was immediately re-read with `cat` to confirm the change before any Terraform command was executed.

**Dual-source validation**
Both `terraform show` and `aws ec2 describe-instances` were used to confirm the applied state. Terraform state reflects what Terraform knows; the AWS CLI independently queries the control plane. Agreement between both sources confirms consistency.

**Tag-based AWS CLI query**
The `--filters "Name=tag:Name,Values=datacenter-ec2"` approach targets the instance by its Name tag rather than by instance ID. This is portable and works correctly even if the instance ID were to change.

**State file present before modification**
The presence of `terraform.tfstate` was confirmed before proceeding. Without an existing state file, Terraform would attempt to provision a new resource rather than updating the existing one.

---

## Lessons Learned

**In-place vs. replacement behaviour depends on the attribute**
Not all EC2 attributes can be modified in-place. Attributes such as `ami` or `subnet_id` typically trigger a destroy-and-recreate cycle. Instance type is one of the attributes the AWS provider can change without replacement by stopping, resizing, and restarting the instance. Always confirm the plan output for `~` (update) versus `-/+` (replace) before applying.

**Dynamic public IPs change on resize**
Because `instance_type` changes require the instance to stop and start, any dynamically assigned public IP is released and a new one is allocated. If a stable IP is required across resizes, an Elastic IP should be associated before performing this operation.

**State must match reality before applying changes**
The `terraform plan` step begins with a state refresh (`aws_instance.ec2: Refreshing state...`). If the actual AWS resource has drifted from the state file, Terraform will detect and account for that drift during planning. This makes reviewing plan output essential, not optional.

**`-out` flag for production apply workflows**
The plan in this implementation was not saved with `-out`. In production environments, saving the plan with `terraform plan -out=tfplan` and applying it with `terraform apply tfplan` guarantees that exactly what was reviewed gets applied, with no possibility of drift between plan and apply if the environment changes in the interim.

**Verification requires both layers**
`terraform show` reflects Terraform's local state. The AWS CLI reflects the actual cloud resource. Confirming both ensures there is no state file inconsistency masking a failed apply.

---

## Technologies Used

| Technology | Purpose |
|---|---|
| Terraform | Infrastructure provisioning and state management |
| AWS EC2 | Compute resource being modified |
| AWS CLI | Independent control plane verification |
| HashiCorp AWS Provider | Terraform provider handling EC2 API calls |
| `sed` | In-place configuration file modification |
| Bash | Command execution environment |




<img width="1043" height="598" alt="image" src="https://github.com/user-attachments/assets/99a15e56-8183-47de-968e-e3d955e02a81" />

<img width="1050" height="722" alt="image" src="https://github.com/user-attachments/assets/d2d1f3e9-c4b8-4fb0-83c0-9458fd48e96f" />

<img width="1045" height="765" alt="image" src="https://github.com/user-attachments/assets/36c611b1-a440-41d4-88b6-3aea23eb4aa8" />
<img width="1043" height="762" alt="image" src="https://github.com/user-attachments/assets/25b1bfbf-b3bd-4e91-b710-5ea5ac4ae398" />
<img width="1045" height="475" alt="image" src="https://github.com/user-attachments/assets/09cc742e-06a5-4406-9471-d6bc2d0a6e76" />
<img width="1048" height="620" alt="image" src="https://github.com/user-attachments/assets/53e2d84f-ec49-4d9e-9e87-0449da6e2348" />




