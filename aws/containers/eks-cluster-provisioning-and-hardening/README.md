# eks-private-cluster-deployment

> **Enterprise-grade provisioning of a private Amazon EKS cluster using AWS CLI with IAM role configuration, EKS Auto Mode disabled, and multi-AZ high availability.**

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Details](#environment-details)
- [Implementation](#implementation)
  - [Phase 1: Environment Verification](#phase-1-environment-verification)
  - [Phase 2: Tool Installation](#phase-2-tool-installation)
  - [Phase 3: Network Resource Discovery](#phase-3-network-resource-discovery)
  - [Phase 4: IAM Role Provisioning](#phase-4-iam-role-provisioning)
  - [Phase 5: Cluster Creation](#phase-5-cluster-creation)
  - [Phase 6: Verification](#phase-6-verification)
- [Problems Encountered and Resolutions](#problems-encountered-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Reference](#reference)

---

## Overview

This repository documents the end-to-end provisioning of a **private Amazon EKS cluster** named `devops-eks` using exclusively the **AWS CLI**. The cluster is deployed with custom configuration, a dedicated IAM role, EKS Auto Mode explicitly disabled, and private-only endpoint access enforced across three availability zones for production-grade high availability.

**Business Context:** The Nautilus DevOps team required a Kubernetes-based infrastructure baseline on Amazon EKS that satisfies internal security and scalability standards, specifically minimizing external exposure by keeping the cluster control plane endpoint private and distributing workload capacity across multiple physical locations.

---

## Architecture

```
AWS Region: us-east-1
|
+-- Default VPC (vpc-057870f174cae731a)
    |
    +-- Subnet: us-east-1a (subnet-055fe6bb0490b771f)
    +-- Subnet: us-east-1b (subnet-0eba79c5af9ad1fa4)
    +-- Subnet: us-east-1c (subnet-0531e56a775ebf5fc)
    |
    +-- EKS Control Plane: devops-eks
        |-- Kubernetes v1.35 (latest stable)
        |-- IAM Role: eksClusterRole
        |-- Endpoint: PRIVATE ONLY (endpointPublicAccess: false)
        |-- EKS Auto Mode: DISABLED
        |-- Block Storage Auto Provisioning: DISABLED
        |-- Elastic Load Balancing Auto Provisioning: DISABLED
```

**Key Security Posture:**
- Zero public endpoint exposure
- No automatic compute, storage, or load balancer provisioning
- All control plane access restricted to within the VPC

---

## Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| AWS CLI | >= 1.40.x | Primary provisioning tool |
| kubectl | >= 1.35.x | Kubernetes cluster interaction |
| AWS IAM permissions | CreateRole, AttachRolePolicy, eks:CreateCluster, ec2:DescribeVpcs, ec2:DescribeSubnets | Minimum required permissions |
| AWS Region | us-east-1 | Target deployment region |

> **Note:** `eksctl` is NOT required. This guide uses AWS CLI exclusively, which is the most portable and dependency-light approach.

---

## Environment Details

| Parameter | Value |
|-----------|-------|
| AWS Account ID | `914784730395` |
| IAM User | `kk_labs_user_868512` |
| Region | `us-east-1` |
| Cluster Name | `devops-eks` |
| Kubernetes Version | `1.35` |
| VPC ID | `vpc-057870f174cae731a` |
| Subnet 1a | `subnet-055fe6bb0490b771f` |
| Subnet 1b | `subnet-0eba79c5af9ad1fa4` |
| Subnet 1c | `subnet-0531e56a775ebf5fc` |
| IAM Role | `eksClusterRole` |

---

## Implementation

### Phase 1: Environment Verification

Confirm AWS CLI is functional and the correct identity and region are active before proceeding with any provisioning.

**Step 1.1 - Verify AWS CLI version and caller identity**

```bash
aws --version
aws sts get-caller-identity
```

**Expected Output:**
```json
{
    "UserId": "AIDA5J7LLHUN3JJPYO2OT",
    "Account": "914784730395",
    "Arn": "arn:aws:iam::914784730395:user/kk_labs_user_868512"
}
```

> **SCREENSHOT**

<img width="1030" height="478" alt="image" src="https://github.com/user-attachments/assets/a7441750-426b-4748-93d9-c4f793d2855d" />

> *Caption: AWS CLI version confirmed and caller identity verified showing account 914784730395 in us-east-1*

**Step 1.2 - Confirm active region**

```bash
aws configure get region
```

**Expected Output:**
```
us-east-1
```

> If region is not set correctly, run:
> ```bash
> aws configure set region us-east-1
> ```

---

### Phase 2: Tool Installation

#### 2.1 Install kubectl

`eksctl` and `kubectl` are not pre-installed in this environment. Since `eksctl` is not needed (AWS CLI handles all provisioning), only `kubectl` is required for post-creation cluster interaction.

**Step 2.1 - Download kubectl binary**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

**Step 2.2 - Set execute permissions**

```bash
chmod +x kubectl
```

**Step 2.3 - Move to system PATH**

```bash
sudo mv kubectl /usr/local/bin/
```

**Step 2.4 - Verify installation**

```bash
kubectl version --client
```

**Expected Output:**
```
Client Version: v1.35.2
Kustomize Version: v5.7.1
```

> **[SCREENSHOT PLACEHOLDER]**
> *Caption: kubectl v1.35.2 successfully installed and verified*

---

### Phase 3: Network Resource Discovery

Identify the default VPC and retrieve subnet IDs for availability zones a, b, and c before cluster creation. Storing the VPC ID as a shell variable reduces manual error risk.

**Step 3.1 - Retrieve default VPC ID**

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text) && echo $VPC_ID
```

**Expected Output:**
```
vpc-057870f174cae731a
```

**Step 3.2 - List all subnets in the default VPC**

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].{AZ:AvailabilityZone,SubnetId:SubnetId}" \
  --output table
```

**Expected Output:**
```
--------------------------------------------
|              DescribeSubnets             |
+-------------+----------------------------+
|     AZ      |         SubnetId           |
+-------------+----------------------------+
|  us-east-1a |  subnet-055fe6bb0490b771f  |
|  us-east-1b |  subnet-0eba79c5af9ad1fa4  |
|  us-east-1c |  subnet-0531e56a775ebf5fc  |
|  us-east-1d |  subnet-0fa660b4f32b68b9c  |
|  us-east-1e |  subnet-00197a84a99208a35  |
|  us-east-1f |  subnet-00d19661e6f64e225  |
+-------------+----------------------------+
```

> **Subnets selected for the cluster:** `us-east-1a`, `us-east-1b`, `us-east-1c` only, as specified by the task requirements.

> **[SCREENSHOT PLACEHOLDER]**
> *Caption: Default VPC and all 6 subnets listed. Subnets for AZs a, b, and c identified for cluster use*

---

### Phase 4: IAM Role Provisioning

The EKS control plane requires an IAM role with the `AmazonEKSClusterPolicy` managed policy. This role does not exist by default and must be created with the correct trust relationship allowing `eks.amazonaws.com` to assume it.

**Step 4.1 - Create the IAM trust policy document**

```bash
cat > eks-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

**Step 4.2 - Create the eksClusterRole IAM role**

```bash
aws iam create-role \
  --role-name eksClusterRole \
  --assume-role-policy-document file://eks-trust-policy.json \
  --query "Role.Arn" \
  --output text
```

**Expected Output:**
```
arn:aws:iam::914784730395:role/eksClusterRole
```

**Step 4.3 - Attach the required managed policy**

```bash
aws iam attach-role-policy \
  --role-name eksClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

> This command returns no output on success. No output means success.

**Step 4.4 - Verify policy attachment**

```bash
aws iam list-attached-role-policies \
  --role-name eksClusterRole \
  --output table
```

**Expected Output:**
```
--------------------------------------------------------------------------------
|                           ListAttachedRolePolicies                           |
+------------------------------------------------------------------------------+
||                              AttachedPolicies                              ||
|+-------------------------------------------------+--------------------------+|
||                    PolicyArn                    |       PolicyName         ||
|+-------------------------------------------------+--------------------------+|
||  arn:aws:iam::aws:policy/AmazonEKSClusterPolicy |  AmazonEKSClusterPolicy  ||
|+-------------------------------------------------+--------------------------+|
```

> **[SCREENSHOT PLACEHOLDER]**
> *Caption: eksClusterRole created with ARN confirmed and AmazonEKSClusterPolicy successfully attached*

---

### Phase 5: Cluster Creation

With all pre-requisites confirmed, create the EKS cluster. The critical lesson from earlier attempts in this session is that AWS requires `computeConfig`, `kubernetesNetworkConfig` (elasticLoadBalancing), and `storageConfig` (blockStorage) to ALL be explicitly disabled together when opting out of EKS Auto Mode. Disabling only `computeConfig` results in an `InvalidParameterException`.

**Step 5.1 - Verify latest stable Kubernetes version available on EKS**

```bash
aws eks describe-cluster-versions \
  --query "sort_by(clusterVersions, &clusterVersion)[-1].clusterVersion" \
  --output text
```

**Expected Output:**
```
1.35
```

**Step 5.2 - Create the private EKS cluster**

```bash
aws eks create-cluster \
  --name devops-eks \
  --region us-east-1 \
  --kubernetes-version 1.35 \
  --role-arn arn:aws:iam::914784730395:role/eksClusterRole \
  --resources-vpc-config subnetIds=subnet-055fe6bb0490b771f,subnet-0eba79c5af9ad1fa4,subnet-0531e56a775ebf5fc,endpointPublicAccess=false,endpointPrivateAccess=true \
  --compute-config enabled=false \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled":false}}' \
  --storage-config '{"blockStorage":{"enabled":false}}'
```

**Expected Output (truncated):**
```json
{
    "cluster": {
        "name": "devops-eks",
        "arn": "arn:aws:eks:us-east-1:914784730395:cluster/devops-eks",
        "version": "1.35",
        "roleArn": "arn:aws:iam::914784730395:role/eksClusterRole",
        "resourcesVpcConfig": {
            "endpointPublicAccess": false,
            "endpointPrivateAccess": true
        },
        "status": "CREATING",
        "computeConfig": { "enabled": false },
        "storageConfig": { "blockStorage": { "enabled": false } }
    }
}
```

> **[SCREENSHOT PLACEHOLDER]**
> *Caption: Cluster creation initiated with status CREATING and all configuration parameters confirmed in the JSON response*

**Step 5.3 - Poll cluster status until ACTIVE**

```bash
aws eks describe-cluster \
  --name devops-eks \
  --region us-east-1 \
  --query "cluster.status" \
  --output text
```

> Re-run this command every 2-3 minutes. Cluster provisioning typically takes 10-15 minutes.

**Expected progression:**
```
CREATING  (run 1-4)
ACTIVE    (run 5+)
```

> **[SCREENSHOT PLACEHOLDER]**
> *Caption: Multiple status polls showing CREATING transitioning to ACTIVE after approximately 10 minutes*

---

### Phase 6: Verification

Run all verification checks sequentially after the cluster reaches `ACTIVE` status to confirm every requirement is met.

**Step 6.1 - Verify cluster name, status, and Kubernetes version**

```bash
aws eks describe-cluster \
  --name devops-eks \
  --query "cluster.{Name:name, Version:version, Status:status}" \
  --output table
```

**Expected Output:**
```
-------------------------------------
|          DescribeCluster          |
+------------+----------+-----------+
|    Name    | Status   |  Version  |
+------------+----------+-----------+
|  devops-eks|  ACTIVE  |  1.35     |
+------------+----------+-----------+
```

**Step 6.2 - Verify IAM role**

```bash
aws eks describe-cluster \
  --name devops-eks \
  --query "cluster.roleArn" \
  --output text
```

**Expected Output:**
```
arn:aws:iam::914784730395:role/eksClusterRole
```

**Step 6.3 - Verify endpoint access configuration**

```bash
aws eks describe-cluster \
  --name devops-eks \
  --query "cluster.resourcesVpcConfig.{PublicAccess:endpointPublicAccess,PrivateAccess:endpointPrivateAccess}" \
  --output table
```

**Expected Output:**
```
-----------------------------------
|         DescribeCluster         |
+----------------+----------------+
|  PrivateAccess | PublicAccess   |
+----------------+----------------+
|  True          |  False         |
+----------------+----------------+
```

**Step 6.4 - Verify subnets across all three AZs**

```bash
aws eks describe-cluster \
  --name devops-eks \
  --query "cluster.resourcesVpcConfig.subnetIds" \
  --output table
```

**Expected Output:**
```
------------------------------
|       DescribeCluster      |
+----------------------------+
|  subnet-055fe6bb0490b771f  |
|  subnet-0eba79c5af9ad1fa4  |
|  subnet-0531e56a775ebf5fc  |
+----------------------------+
```

**Step 6.5 - Verify EKS Auto Mode is fully disabled**

```bash
aws eks describe-cluster \
  --name devops-eks \
  --query "cluster.{ComputeConfig:computeConfig,StorageConfig:storageConfig}" \
  --output json
```

**Expected Output:**
```json
{
    "ComputeConfig": {
        "enabled": false,
        "nodePools": []
    },
    "StorageConfig": {
        "blockStorage": {
            "enabled": false
        }
    }
}
```

> **[SCREENSHOT PLACEHOLDER]**
> *Caption: Full verification suite completed showing all parameters matching requirements including private endpoint, correct subnets, and EKS Auto Mode fully disabled*

**Step 6.6 - Update kubeconfig for cluster access (within VPC only)**

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name devops-eks
```

```bash
kubectl get svc
```

> **Important:** Because the cluster endpoint is private, `kubectl` commands will only succeed from within the VPC (for example, from an EC2 instance or a bastion host in the same VPC). Running these commands from outside the VPC will time out by design.

---

## Problems Encountered and Resolutions

### Problem: `eksClusterRole` Does Not Exist

**Error:**
```
An error occurred (NoSuchEntity) when calling the GetRole operation:
The role with name eksClusterRole cannot be found.
```

**Trigger Command:**
```bash
aws iam get-role \
  --role-name eksClusterRole \
  --query "Role.Arn" \
  --output text
```

**Root Cause:** Lab environments are ephemeral and do not pre-provision IAM roles. Each new session starts with a completely clean account state. The `eksClusterRole` that EKS requires to manage the control plane on your behalf does not exist until explicitly created.

**Resolution:** Created the role programmatically using a trust policy document that delegates `sts:AssumeRole` to the `eks.amazonaws.com` service principal, then attached the required managed policy before proceeding to cluster creation:

```bash
# Step 1 - Create the trust policy document
cat > eks-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Step 2 - Create the role
aws iam create-role \
  --role-name eksClusterRole \
  --assume-role-policy-document file://eks-trust-policy.json \
  --query "Role.Arn" \
  --output text

# Step 3 - Attach the required EKS managed policy
aws iam attach-role-policy \
  --role-name eksClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# Step 4 - Verify attachment
aws iam list-attached-role-policies \
  --role-name eksClusterRole \
  --output table
```

**Outcome:** Role created with ARN `arn:aws:iam::914784730395:role/eksClusterRole` and `AmazonEKSClusterPolicy` confirmed attached. Cluster creation proceeded without further IAM errors.

> **Prevention:** Always run `aws iam get-role --role-name eksClusterRole` as a mandatory pre-flight check at the start of every session. Treat role creation as a conditional prerequisite step, not an assumed given.

---

## Best Practices

**IAM Role Management**

Always verify the target IAM role exists before initiating cluster creation. Use `aws iam get-role` as a pre-flight check. In ephemeral lab and CI/CD environments, never assume roles carry over between sessions. Codify role creation as an idempotent step that checks for existence first.

**Shell Variables for Resource IDs**

Store dynamically retrieved values such as VPC IDs in shell variables rather than copying and pasting raw strings. This eliminates transcription errors and makes the sequence reproducible as a script:

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
```

**EKS Auto Mode: Treat as Atomic**

When disabling EKS Auto Mode, always disable all three sub-components in the same API call: `computeConfig`, `elasticLoadBalancing`, and `blockStorage`. Partial configuration is rejected by the EKS API. Treat it as a single toggle, not three independent flags.

**Private Endpoint Clusters Require In-VPC Access**

Clusters with `endpointPublicAccess: false` are intentionally inaccessible from the public internet. Always provision a bastion host, AWS Cloud9 environment, or VPN connection within the same VPC before attempting `kubectl` operations. Document this constraint explicitly in team runbooks.

**Always Pin Kubernetes Version**

Use `aws eks describe-cluster-versions` to programmatically retrieve the latest stable version rather than hardcoding a version number. Hardcoded versions lead to drift and failed deployments when the pinned version is deprecated.

**Verification as a Gate**

Never consider provisioning complete without running the full verification suite. Each check targets a specific requirement and provides an audit trail. Treat verification as a non-negotiable final phase, not an optional step.

**Subnet Selection**

Always confirm that selected subnets correspond to distinct physical availability zones before creating the cluster. Selecting subnets in the same AZ defeats the purpose of multi-AZ deployment and does not guarantee high availability.

---

## Lessons Learned

**1. EKS Auto Mode is an all-or-nothing API contract.**
AWS does not allow partial configuration of EKS Auto Mode components. This is an underdocumented constraint that is only surfaced at runtime. Any automation that disables Auto Mode must disable all three components simultaneously: compute, load balancing, and block storage.

**2. Tool availability cannot be assumed in ephemeral environments.**
In cloud lab environments, common tools like `eksctl` and `kubectl` may not be present. Building a workflow that depends only on the AWS CLI as the primary tool eliminates this class of failure entirely. `eksctl` adds a convenience layer but is not necessary for cluster provisioning.

**3. IAM roles are session-scoped in lab environments.**
Lab environments frequently rotate AWS accounts and reset IAM state. Every provisioning workflow must include a role existence check and a creation step that runs conditionally. This prevents both "role not found" failures and "entity already exists" conflicts.

**4. Shell variables eliminate copy-paste errors in multi-step workflows.**
Manually copying VPC IDs and subnet IDs across multiple commands is error-prone. Using shell variable assignment at the discovery phase and referencing the variable throughout the workflow reduces errors and makes the sequence scriptable.

**5. Private clusters require access planning before provisioning.**
A private EKS endpoint is only useful if access from within the VPC is planned in advance. Teams should provision access infrastructure (bastion, VPN, or Cloud9) as part of the same deployment pipeline, not as an afterthought after finding that `kubectl` commands time out.

**6. Status polling requires patience.**
EKS control plane provisioning consistently takes 10-15 minutes. Implementing a polling loop with a reasonable interval (30 seconds) rather than hammering the API is both more efficient and more professional. Use `watch -n 30` or a loop script rather than manual repeated invocations.

---

## Reference

| Resource | URL |
|----------|-----|
| AWS EKS CreateCluster API | https://docs.aws.amazon.com/eks/latest/APIReference/API_CreateCluster.html |
| AmazonEKSClusterPolicy | https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonEKSClusterPolicy.html |
| EKS Auto Mode Documentation | https://docs.aws.amazon.com/eks/latest/userguide/automode.html |
| kubectl Installation | https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/ |
| EKS Private Cluster Access | https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html |

---

*Infrastructure provisioned on Amazon EKS us-east-1 | Kubernetes v1.35*



<img width="1035" height="422" alt="image" src="https://github.com/user-attachments/assets/ff4f8237-e27f-47bb-aa36-76c995c28197" />
<img width="1035" height="547" alt="image" src="https://github.com/user-attachments/assets/8dd2bee0-ca4c-4bdf-8b76-726a4d2328df" />
<img width="1031" height="500" alt="image" src="https://github.com/user-attachments/assets/5c94059f-8463-49c3-90a4-571a883117bb" />
<img width="1027" height="798" alt="image" src="https://github.com/user-attachments/assets/41ce8e8d-369a-4987-a8e5-822f89adecb5" />
<img width="1031" height="848" alt="image" src="https://github.com/user-attachments/assets/584697d6-2358-419b-bfa3-e239bfae99b8" />
<img width="1026" height="856" alt="image" src="https://github.com/user-attachments/assets/2352f9a7-601a-4109-8a0d-06aae981cd88" />
<img width="1037" height="870" alt="image" src="https://github.com/user-attachments/assets/b7f978f4-4340-43b5-9c29-e890a418d084" />
<img width="1026" height="517" alt="image" src="https://github.com/user-attachments/assets/7959cb15-95b2-4d71-a36d-34dcb6e47d38" />
<img width="1031" height="861" alt="image" src="https://github.com/user-attachments/assets/c5187a3b-3959-48c2-a24a-ee2a77d23879" />
