# Containers

> AWS container infrastructure projects covering private image registries, serverless container deployments, and managed Kubernetes cluster provisioning. All work is CLI-driven, production-scoped, and documented to reflect real operational workflows.

---

## Directory Structure

```
containers/
├── container-registry-ecr/           # Private ECR repository with Docker build and push pipeline
├── ecr-ecs-fargate-deployment/        # End-to-end ECS Fargate deployment with IAM, networking, and CloudWatch
├── eks-cluster-provisioning-and-hardening/   # Private EKS cluster with IAM, multi-AZ subnets, Auto Mode disabled
└── README.md
```

---

## Project Summaries

### [container-registry-ecr](./container-registry-ecr)

**Quick Summary:** Provisions a private Amazon ECR repository, builds a Docker image from a local Dockerfile, and pushes the tagged image using token-based authentication. Validates availability with `aws ecr list-images`.

| | |
|---|---|
| **Purpose** | Establish a secure, IAM-integrated container image registry as a foundation for downstream ECS and EKS deployments. |
| **Approach** | Created the repository via AWS CLI, authenticated Docker using `get-login-password` (no plaintext credentials), built and tagged the image against the full ECR URI, then verified the digest post-push. |
| **Outcome** | Private registry operational in `us-east-1` with `latest` image confirmed present and ready for deployment pipelines. |

---

### [ecr-ecs-fargate-deployment](./ecr-ecs-fargate-deployment)

**Quick Summary:** Full-stack container deployment pipeline from Dockerfile to publicly accessible nginx service. Covers IAM role creation, private ECR registry, ECS Fargate cluster, task definition, CloudWatch logging, VPC networking, and HTTP verification.

| | |
|---|---|
| **Purpose** | Deploy a containerized application on serverless compute without managing EC2 instances, with centralized logging and a publicly reachable endpoint. |
| **Approach** | Created `ecsTaskExecutionRole` with the AWS-managed execution policy, provisioned a private ECR repository with `scanOnPush` enabled, registered a Fargate task definition with `awsvpc` networking and `awslogs` driver, and created the ECS service with `assignPublicIp=ENABLED`. CloudWatch log group was pre-created to avoid task launch failures. |
| **Outcome** | Service reached steady state (`Running: 1 / Desired: 1`). HTTP 200 confirmed via `curl` against the Fargate task's public IP. Full cleanup sequence documented. |

**Key decisions:**
- Log group provisioned before task definition registration to prevent silent task failures at launch.
- `scanOnPush=true` applied at ECR creation to meet image security compliance requirements.
- Public IP enabled at the service level, not at the task level, to match ECS Fargate `awsvpc` behavior.

---

### [eks-cluster-provisioning-and-hardening](./eks-cluster-provisioning-and-hardening)

**Quick Summary:** Provisions a private Amazon EKS cluster using AWS CLI exclusively. Covers IAM role setup, multi-AZ subnet selection, private endpoint enforcement, and full EKS Auto Mode opt-out. No `eksctl` dependency.

| | |
|---|---|
| **Purpose** | Stand up a production-scoped Kubernetes control plane with zero public endpoint exposure and no auto-provisioned compute, storage, or load balancers. |
| **Approach** | Created `eksClusterRole` with `AmazonEKSClusterPolicy` from a trust policy document. Retrieved default VPC and mapped subnets to `us-east-1a/b/c`. Disabled EKS Auto Mode atomically: `computeConfig`, `elasticLoadBalancing`, and `blockStorage` all set in a single API call to satisfy the EKS validation contract. Cluster status polled until `ACTIVE`. |
| **Outcome** | Cluster `devops-eks` active on Kubernetes v1.35. `endpointPublicAccess: false`, `endpointPrivateAccess: true` confirmed. All Auto Mode components verified disabled. Full verification suite executed and documented. |

**Key decisions:**
- AWS CLI used exclusively to maximize portability and eliminate `eksctl` as a runtime dependency.
- All three Auto Mode sub-components disabled in a single `create-cluster` call. Partial configuration is rejected by the EKS API; this constraint is explicitly documented.
- IAM pre-flight check included as a mandatory step due to ephemeral lab environments resetting account state between sessions.

---

## Technologies and Tools

| Layer | Tools |
|---|---|
| Container Registry | Amazon ECR (private, AES256, scan on push) |
| Container Orchestration | Amazon ECS (Fargate), Amazon EKS (Kubernetes v1.35) |
| Compute Model | AWS Fargate (serverless), EKS control plane (managed) |
| IAM | Custom roles, AWS-managed policies, trust policy documents |
| Networking | Default VPC, `awsvpc` mode, security group ingress rules, multi-AZ subnets |
| Observability | Amazon CloudWatch Logs (`awslogs` driver) |
| CLI | AWS CLI v2 (primary provisioning tool across all projects) |
| Runtime | Docker Engine (build, tag, push) |

---

## Key Skills Demonstrated

- Private container registry provisioning with image lifecycle and security scanning
- IAM role creation from trust policy documents for ECS task execution and EKS cluster management
- Serverless container deployment using ECS Fargate with `awsvpc` networking and CloudWatch log integration
- EKS cluster provisioning via AWS CLI with private endpoint enforcement and manual Auto Mode opt-out
- Multi-AZ subnet selection and VPC resource discovery using query-filtered CLI commands
- Shell variable usage to reduce transcription errors across multi-step provisioning workflows
- Structured troubleshooting documentation covering root cause, resolution, and prevention for each encountered error
- Cleanup sequences documented in reverse dependency order to prevent dangling resource costs

---

## Navigation

Each subdirectory contains a self-contained `README.md` with:
- Full CLI command sequences
- Expected outputs and screenshots at each step
- Errors encountered with root cause analysis and resolution
- Best practices and lessons learned sections

Start with `container-registry-ecr` for ECR fundamentals, then `ecr-ecs-fargate-deployment` for a complete ECS deployment lifecycle, then `eks-cluster-provisioning-and-hardening` for Kubernetes control plane provisioning.

---

*All projects executed in AWS `us-east-1`. Infrastructure provisioned via AWS CLI. No console-only steps.*
