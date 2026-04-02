# Containerized Image Delivery: Private Amazon ECR Repository Provisioning and Docker Image Deployment
---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Objectives](#objectives)
- [Architecture Overview](#architecture-overview)
- [Environment and Constraints](#environment-and-constraints)
- [Solution Strategy](#solution-strategy)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Verify AWS Identity](#step-1-verify-aws-identity)
  - [Step 2: Create Private ECR Repository](#step-2-create-private-ecr-repository)
  - [Step 3: Navigate to Application Directory](#step-3-navigate-to-application-directory)
  - [Step 4: Build Docker Image from Dockerfile](#step-4-build-docker-image-from-dockerfile)
  - [Step 5: Authenticate Docker with ECR](#step-5-authenticate-docker-with-ecr)
  - [Step 6: Tag Image for ECR](#step-6-tag-image-for-ecr)
  - [Step 7: Push Image to ECR](#step-7-push-image-to-ecr)
  - [Step 8: Verify Image Availability in ECR](#step-8-verify-image-availability-in-ecr)
- [Final Validation Checklist](#final-validation-checklist)
- [Deliverables](#deliverables)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Key Learnings](#key-learnings)
- [Next Steps](#next-steps)
- [Conclusion](#conclusion)

---

## Overview

This document details the end-to-end provisioning of a **private Amazon Elastic Container Registry (ECR) repository** and the complete workflow for building, authenticating, tagging, pushing, and verifying a Docker image within it. The implementation follows a strict AWS-native approach and is scoped exclusively to the **us-east-1** region using pre-configured IAM credentials.

This work reflects a production-aligned container delivery workflow suitable for CI/CD pipeline integration, image governance, and downstream Kubernetes or ECS-based deployments.

---

## Problem Statement

The Nautilus DevOps team requires a **secure, centralized, and scalable container image registry** to support modern application delivery workflows. Specifically, the team needs to:

- Provision a **private container registry** that is accessible only through authenticated IAM principals
- Build a Docker image from an existing application source hosted on the delivery host
- Securely authenticate Docker with the registry using AWS-native token-based authentication
- Push and verify the image to confirm availability for downstream deployment consumers

Without a centralized registry, container images are at risk of version drift, insecure distribution, and absence of an audit trail, all of which are unacceptable in enterprise delivery pipelines.

---

## Objectives

- Create a **private Amazon ECR** repository named `nautilus-ecr` in `us-east-1`
- Build a Docker image from a `Dockerfile` located at `/root/pyapp`
- Tag the image as `latest` using the full ECR repository URI
- Authenticate the Docker daemon with ECR using the `get-login-password` method
- Push the tagged image to the ECR repository
- Verify the image is present and correctly tagged in the registry

---

## Architecture Overview

```
Developer Host (aws-client)
        |
        |  docker build
        v
Local Docker Image  (nautilus-ecr:latest)
        |
        |  aws ecr get-login-password  |  docker login
        v
Authenticated Docker Session
        |
        |  docker tag + docker push
        v
Private ECR Repository
821328497772.dkr.ecr.us-east-1.amazonaws.com/nautilus-ecr:latest
        |
        |  aws ecr list-images
        v
Verified Image Availability (digest + tag confirmed)
```

---

## Environment and Constraints

| Parameter | Value |
|---|---|
| Cloud Provider | Amazon Web Services |
| Container Registry | Amazon Elastic Container Registry (ECR) |
| Region | `us-east-1` |
| Repository Name | `nautilus-ecr` |
| Image Tag | `latest` |
| Dockerfile Location | `/root/pyapp` |
| Base Image | `python:3.8-slim` |
| Access Type | Private |
| Encryption | AES256 (default) |
| IAM User | `kk_labs_user_307865` |
| Account ID | `821328497772` |

---

## Solution Strategy

The implementation follows a linear, dependency-ordered sequence:

1. **Confirm IAM identity** to validate credentials and region binding before any resource provisioning
2. **Provision the ECR repository** using the AWS CLI with explicit region scoping
3. **Build the Docker image** locally from the application source directory
4. **Authenticate Docker** with ECR using the `get-login-password` piped credential approach
5. **Tag the image** with the fully qualified ECR URI to prepare it for the registry push
6. **Push the image** to ECR and confirm all layers are uploaded
7. **Verify the image** exists in the registry with the correct tag and digest

---

## Step-by-Step Implementation

---

### Step 1: Verify AWS Identity

Before provisioning any infrastructure, validate that the AWS CLI is configured with the correct IAM principal and that the session is scoped to the expected account.

```bash
aws sts get-caller-identity
```

**Expected output fields:**

- `UserId` confirming the IAM user identity
- `Account` confirming the target AWS account
- `Arn` confirming the principal's IAM path

**Why this matters:** Executing ECR or IAM-related operations under an incorrect identity or account can result in resource creation in the wrong scope, unauthorized access errors, or orphaned resources.

**Screenshot: IAM identity confirmation via `aws sts get-caller-identity`**

![Step 1 - AWS STS Get Caller Identity](https://github.com/user-attachments/assets/67326ba6-d431-4512-b651-9455990240d6)

> Identity confirmed: Account `821328497772`, IAM user `kk_labs_user_307865`, region `us-east-1`.

---

### Step 2: Create Private ECR Repository

Provision a private ECR repository using the AWS CLI. Specifying the region explicitly prevents SDK-level default region fallback, which can result in repositories being created in unintended regions.

```bash
aws ecr create-repository \
  --repository-name nautilus-ecr \
  --region us-east-1
```

**Key output fields to validate:**

- `repositoryArn` confirms the resource was created in the correct account and region
- `repositoryUri` is the fully qualified push/pull URI required for subsequent steps
- `encryptionType: AES256` confirms at-rest encryption is active by default
- `imageTagMutability: MUTABLE` is the default; evaluate whether immutable tagging is appropriate for your production environment

**Screenshot: ECR repository creation output confirming ARN, URI, and encryption configuration**

![Step 2 - ECR Repository Creation](https://github.com/user-attachments/assets/cc7d219d-3ffc-4944-9f85-984b7ec2f4ea)

> Repository URI: `821328497772.dkr.ecr.us-east-1.amazonaws.com/nautilus-ecr`

---

### Step 3: Navigate to Application Directory

Change into the application directory containing the `Dockerfile` and associated application source files. This ensures `docker build` resolves the build context correctly.

```bash
cd /root/pyapp
```

**Screenshot: Directory navigation confirming the shell is in `/root/pyapp` before the build**

![Step 3 - Navigate to pyapp Directory](https://github.com/user-attachments/assets/52738a9f-cff0-4680-930e-53c25aa375c4)

> Shell prompt confirms working directory is now `~/pyapp` before initiating the build.

---

### Step 4: Build Docker Image from Dockerfile

Execute the Docker build from the `/root/pyapp` directory. The `.` at the end of the command sets the build context to the current directory, meaning Docker will include all files in that directory unless excluded by a `.dockerignore` file.

```bash
docker build -t nautilus-ecr:latest .
```

**Build stages observed from the output:**

- `[1/4]` Pull of `python:3.8-slim` base image from Docker Hub (resolved via digest)
- `[2/4]` `COPY . /app` copies application source into the image filesystem
- `[3/4]` `WORKDIR /app` sets the working directory for subsequent instructions
- `[4/4]` `RUN pip install -r requirements.txt` installs Python dependencies
- Build completes in `188.2s` with all 9 steps finished successfully

**Operational note:** The base image `python:3.8-slim` is resolved by digest, not just tag, ensuring reproducibility. The 122-second metadata fetch indicates the image was pulled fresh rather than served from local cache, which is expected in a clean environment.

**Screenshot: Docker build output showing all build stages and successful image creation**

![Step 4 - Docker Build](https://github.com/user-attachments/assets/c8f0105e-d172-4eab-a1e0-4d571c1a2e61)

> Image `nautilus-ecr:latest` built and written with digest `sha256:4e2e4dee949d7414bc70d3cc98d06fc2c0648e57d75cfc9a7fb54588fa32c008`.

---

### Step 5: Authenticate Docker with ECR

ECR uses short-lived authentication tokens rather than static credentials. The `get-login-password` command retrieves a 12-hour token, which is piped directly into `docker login` to avoid exposing credentials in shell history or environment variables.

```bash
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin 821328497772.dkr.ecr.us-east-1.amazonaws.com
```

**Security characteristics of this approach:**

- Token is time-bound (expires after 12 hours)
- No plaintext credentials are stored on disk or exposed to the process table
- `--password-stdin` ensures the password is consumed via stdin rather than as a CLI argument

**Warning observed:** Docker emits a warning that credentials are stored unencrypted in `/root/.docker/config.json`. In production environments, configure a [Docker credential store](https://docs.docker.com/go/credential-store/) to eliminate this exposure.

**Screenshot: ECR authentication producing `Login Succeeded` confirmation**

![Step 5 - ECR Docker Login](https://github.com/user-attachments/assets/bee87ed7-4175-4565-be90-fa4d3048211f)

> `Login Succeeded` confirms the Docker daemon is now authorized to push to the ECR registry endpoint.

---

### Step 6: Tag Image for ECR

Docker requires that an image be tagged with the fully qualified repository URI before it can be pushed to a remote registry. This step creates a new tag reference pointing the local `nautilus-ecr:latest` image to the ECR destination URI.

```bash
docker tag nautilus-ecr:latest \
  821328497772.dkr.ecr.us-east-1.amazonaws.com/nautilus-ecr:latest
```

**Why explicit tagging is required:** Docker uses the registry hostname embedded in the image tag to determine the push destination. Without this step, a `docker push nautilus-ecr:latest` would attempt to push to Docker Hub, not ECR, and would fail.

**Screenshot: Docker tag command applied, with no output confirming silent success**

![Step 6 - Docker Tag](https://github.com/user-attachments/assets/2cb46a28-ce5d-41f5-8d11-f12b907d0135)

> Silent exit from `docker tag` confirms successful tag assignment. Verify with `docker images` if confirmation is needed.

---

### Step 7: Push Image to ECR

Push the tagged image to the ECR repository. Docker resolves the registry endpoint from the image URI and transmits each layer individually. Layers already present in ECR from prior pushes are skipped via digest matching.

```bash
docker push 821328497772.dkr.ecr.us-east-1.amazonaws.com/nautilus-ecr:latest
```

**Push output breakdown:**

- 7 image layers were pushed: `7573a134db1f`, `5f70bf18a086`, `ca8f5e0ddf43`, `d2a2207b52a4`, `5d2d143f3d7f`, `c3772b569c3a`, `8d853c8add5d`
- All layers reported `Pushed`, confirming no layer deduplication occurred (fresh repository)
- Manifest digest returned: `sha256:7777a365e0290b5250d6aa598528c09c01b66d2a4f333f24fc8d3829246d6343`
- Manifest size: `1783` bytes

**Screenshot: Docker push output showing all layers pushed and the final manifest digest**

![Step 7 - Docker Push](https://github.com/user-attachments/assets/a750cd1d-723b-4e3a-8637-eb82fa0688d6)

> All 7 layers confirmed pushed. Manifest digest provides cryptographic proof of image integrity for downstream consumers.

---

### Step 8: Verify Image Availability in ECR

After pushing, use the AWS CLI to confirm the image is visible in the ECR repository with the expected tag and digest. This step is critical for validating that the push was accepted by the registry and that the image is available for pull by downstream systems.

```bash
aws ecr list-images --repository-name nautilus-ecr
```

**Expected output:**

```json
{
    "imageIds": [
        {
            "imageDigest": "sha256:7777a365e0290b5250d6aa598528c09c01b66d2a4f333f24fc8d3829246d6343",
            "imageTag": "latest"
        }
    ]
}
```

**Validation points:**

- `imageTag: latest` confirms the correct tag is registered
- `imageDigest` matches the digest returned by `docker push`, confirming end-to-end integrity
- A non-empty `imageIds` array confirms the image is indexed and queryable by the registry

**Screenshot: ECR `list-images` output confirming the image tag and digest match the pushed artifact**

![Step 8 - ECR List Images](https://github.com/user-attachments/assets/54d2759f-9b7a-410d-a6e1-0c6f3e5bcc44)

> Image confirmed present in `nautilus-ecr` with tag `latest` and digest matching the push output exactly.

---

## Final Validation Checklist

| Validation Check | Status |
|---|---|
| AWS IAM identity confirmed | passed |
| ECR repository created in `us-east-1` | passed |
| Docker image built from Dockerfile | passed |
| ECR authentication succeeded | passed |
| Image tagged with full ECR URI | passed |
| All layers pushed to ECR | passed |
| Image tag and digest verified in registry | passed |
| Region compliance (`us-east-1`) | passed |

---

## Deliverables

- **Private ECR Repository:** `nautilus-ecr`
- **Docker Image Tag:** `latest`
- **Registry URI:** `821328497772.dkr.ecr.us-east-1.amazonaws.com/nautilus-ecr`
- **Image Digest:** `sha256:7777a365e0290b5250d6aa598528c09c01b66d2a4f333f24fc8d3829246d6343`
- **Base Image:** `python:3.8-slim`

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Used `--password-stdin` for Docker login | Prevents token exposure in shell history or process table; AWS-recommended approach |
| Explicit `--region us-east-1` on all CLI commands | Avoids silent fallback to a misconfigured default region in multi-region environments |
| Tagged image with full ECR URI before push | Docker uses the registry hostname in the tag to route the push; required for non-Docker-Hub registries |
| Used `aws ecr list-images` for post-push verification | Independently confirms registry acceptance separate from local Docker state |
| Accepted default `imageTagMutability: MUTABLE` | Appropriate for development workflows; production environments should evaluate `IMMUTABLE` to prevent tag overwriting |

---

## Errors and Resolutions

**No blocking errors were encountered during this implementation.** However, the following warning was observed and is documented for operational awareness:

**Warning: Unencrypted credential storage**

```
WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'.
Configure a credential helper to remove this warning.
See https://docs.docker.com/go/credential-store/
```

**Context:** This warning is emitted by Docker when `docker login` stores the registry token in `~/.docker/config.json` without a configured credential store. In this environment, the host is a short-lived delivery host and the token is a 12-hour ECR session credential, so the risk exposure is low.

**Resolution for production environments:** Configure a Docker credential helper such as `docker-credential-ecr-login` or an OS-native keychain store. The AWS ECR credential helper can be enabled in `~/.docker/config.json` to eliminate manual login and token storage entirely:

```json
{
  "credHelpers": {
    "821328497772.dkr.ecr.us-east-1.amazonaws.com": "ecr-login"
  }
}
```

---

## Best Practices and Operational Considerations

**Image tagging strategy**

Using `latest` is acceptable for development and demonstration workflows, but in production, images should be tagged with immutable identifiers such as Git commit SHAs, semantic versions, or build numbers. This enables precise rollback, deployment traceability, and prevents accidental overwrites.

**Image scanning**

ECR supports automated vulnerability scanning on push via Amazon Inspector. Enable `--image-scanning-configuration scanOnPush=true` at repository creation or update it post-creation:

```bash
aws ecr put-image-scanning-configuration \
  --repository-name nautilus-ecr \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

**Immutable image tags**

For production repositories, set `--image-tag-mutability IMMUTABLE` to prevent tag overwriting. This enforces image promotion discipline and reduces the risk of deploying unintended image versions.

**Lifecycle policies**

Without a lifecycle policy, ECR repositories accumulate untagged and stale images indefinitely, incurring unnecessary storage costs. Define a lifecycle policy to automatically expire images that are no longer needed:

```bash
aws ecr put-lifecycle-policy \
  --repository-name nautilus-ecr \
  --lifecycle-policy-text file://lifecycle-policy.json \
  --region us-east-1
```

**ECR authentication token expiry**

ECR tokens expire after 12 hours. In long-running CI/CD pipelines, ensure that re-authentication logic is embedded before each push phase rather than relying on a token obtained at pipeline start.

**IAM least privilege**

The IAM principal used for ECR operations should be scoped with the minimum required permissions: `ecr:GetAuthorizationToken`, `ecr:CreateRepository`, `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`, and `ecr:ListImages`. Avoid attaching `AdministratorAccess` or broad `ecr:*` policies in production.

---

## Key Learnings

- **ECR repository URIs encode the account and region:** The URI format `<account_id>.dkr.ecr.<region>.amazonaws.com/<repo>` means that images are inherently scoped to a specific account and region. Cross-account or cross-region access requires explicit ECR repository policies.
- **Docker tagging is a routing mechanism, not just a label:** The registry hostname prefix in the image tag determines where `docker push` sends the image. Tagging with the ECR URI is a functional requirement, not an organizational preference.
- **Token-based authentication is stateless and time-bound:** Unlike static registry credentials, ECR tokens expire. Automation scripts and CI systems must handle re-authentication gracefully.
- **Image digest is the ground truth for integrity:** The manifest digest (`sha256:...`) returned by `docker push` and visible in `ecr list-images` provides cryptographic verification that the pushed artifact and the stored artifact are identical.
- **Build layer caching significantly affects build time:** The 122-second metadata fetch observed here is expected on a cold host with no cached layers. Subsequent builds on the same host will be substantially faster due to layer cache reuse.

---

## Next Steps

- Enable **automated image scanning on push** using Amazon Inspector integration for vulnerability detection
- Configure **ECR lifecycle policies** to expire untagged images after a defined retention window
- Apply **immutable image tags** (`IMMUTABLE` mutability) for all production repositories to enforce deployment traceability
- Integrate this workflow into a **CI/CD pipeline** (Jenkins, GitHub Actions, or AWS CodePipeline) to automate build, tag, push, and verify on every commit
- Define and apply **IAM repository policies** to restrict cross-account access to the ECR repository explicitly
- Evaluate the **AWS ECR credential helper** (`amazon-ecr-credential-helper`) to eliminate manual `docker login` steps and remove unencrypted token storage from the delivery host

---

## Conclusion

This implementation delivers a complete, production-aligned container image lifecycle: from AWS identity validation and private registry provisioning, through authenticated Docker build and push, to independent registry-side verification. The workflow is reproducible, auditable, and directly integrable with CI/CD pipelines and container orchestration platforms. The image is securely stored in ECR with AES256 encryption, tagged `latest`, and verified by digest, establishing a reliable artifact baseline for downstream deployment.
