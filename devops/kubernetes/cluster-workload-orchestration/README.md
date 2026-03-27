# Kubernetes Deployment: Apache HTTPD on a Single-Node K3s Cluster

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34.1-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![K3s](https://img.shields.io/badge/K3s-v1.34.1%2Bk3s1-FFC61C?style=for-the-badge&logo=k3s&logoColor=black)
![Apache HTTPD](https://img.shields.io/badge/Apache_HTTPD-latest-D22128?style=for-the-badge&logo=apache&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production_Ready-00C853?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Verification](#environment-verification)
- [Resolution: Step-by-Step Deployment](#resolution-step-by-step-deployment)
  - [Step 1: Verify Host Identity](#step-1-verify-host-identity)
  - [Step 2: Confirm kubectl Availability](#step-2-confirm-kubectl-availability)
  - [Step 3: Validate kubectl Client Version](#step-3-validate-kubectl-client-version)
  - [Step 4: Verify Cluster Connectivity](#step-4-verify-cluster-connectivity)
  - [Step 5: Inspect Cluster Nodes](#step-5-inspect-cluster-nodes)
  - [Step 6: Review Kubeconfig Context](#step-6-review-kubeconfig-context)
  - [Step 7: Confirm Active Context](#step-7-confirm-active-context)
  - [Step 8: Identify Pre-existing State (Error Encountered)](#step-8-identify-pre-existing-state-error-encountered)
  - [Step 9: Create the HTTPD Deployment](#step-9-create-the-httpd-deployment)
  - [Step 10: Verify Deployment Status](#step-10-verify-deployment-status)
  - [Step 11: Inspect the ReplicaSet](#step-11-inspect-the-replicaset)
  - [Step 12: Verify Running Pods](#step-12-verify-running-pods)
  - [Step 13: Describe the Deployment](#step-13-describe-the-deployment)
  - [Step 14: Describe the Pod](#step-14-describe-the-pod)
  - [Step 15: Final Validation of All Resources](#step-15-final-validation-of-all-resources)
- [Error Log and Resolution](#error-log-and-resolution)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This runbook documents the end-to-end process of deploying an **Apache HTTPD** web server as a Kubernetes `Deployment` on a single-node **K3s** cluster running on a `jump-host` control plane node. It covers environment verification, deployment creation, health validation, and post-deployment inspection using `kubectl`.

This guide follows a **Problem and Resolution** model and is written to FAANG-grade operational standards for team onboarding, incident review, and institutional knowledge preservation.

---

## Problem Statement

| Field | Detail |
|---|---|
| **Team** | Nautilus DevOps |
| **Objective** | Deploy the `httpd` application using the `httpd:latest` image on a Kubernetes cluster |
| **Deployment Name** | `httpd` |
| **Target Image** | `httpd:latest` |
| **Cluster Tool** | `kubectl` pre-configured on `jump-host` |
| **Constraint** | Image tag (`latest`) must be explicitly specified |

**Initial State:** No `httpd` deployment existed in the `default` namespace.

**Desired State:** A healthy, running `httpd` Deployment with 1 available replica, backed by a ReplicaSet and a Running Pod.

---

## Architecture

```
jump-host (Control Plane + Worker Node)
|
+-- Namespace: default
    |
    +-- Deployment: httpd
        |
        +-- ReplicaSet: httpd-6c755866c7
            |
            +-- Pod: httpd-6c755866c7-zc2js
                |
                +-- Container: httpd (image: httpd:latest)
                    Pod IP: 10.22.0.9
                    Node IP: 10.244.195.206
```

**Cluster Details:**

| Component | Value |
|---|---|
| Control Plane Endpoint | `https://127.0.0.1:6443` |
| CoreDNS | Running |
| Metrics Server | Running |
| Node | `jump-host` (control-plane, Ready) |
| Kubernetes Version | `v1.34.1+k3s1` |
| kubectl Client Version | `v1.34.1` |
| Kustomize Version | `v5.7.1` |

---

## Prerequisites

* Access to the `jump-host` jump server via SSH or terminal
* `kubectl` installed and available on `$PATH`
* A valid kubeconfig pointing to a reachable Kubernetes API server
* Internet access or a pre-pulled `httpd:latest` image in the container registry
* Permissions to create `Deployments` in the `default` namespace

---

## Environment Verification

Before any deployment action, verify the execution environment end-to-end. Skipping these steps is the leading cause of failed deployments in production environments.

---

## Resolution: Step-by-Step Deployment

---

### Step 1: Verify Host Identity

**Purpose:** Confirm you are operating on the correct host before issuing any cluster commands.

```bash
hostname
```

**Output:**

```
jump-host
```

**Screenshot:**

<img width="1030" height="475" alt="image" src="https://github.com/user-attachments/assets/fdb129ae-1cdf-4b7a-946e-9b9118655783" />

> `Terminal showing hostname output as "jump-host"`

**Validation Criteria:** The hostname must return `jump-host`. If it returns any other value, you are on the wrong machine. Stop and re-establish your session.

---

### Step 2: Confirm kubectl Availability

**Purpose:** Verify that the `kubectl` binary is installed and resolvable from `$PATH`.

```bash
which kubectl
```

**Output:**

```
/usr/bin/kubectl
```

**Screenshot:**

<img width="1030" height="475" alt="image" src="https://github.com/user-attachments/assets/fdb129ae-1cdf-4b7a-946e-9b9118655783" />

> `Terminal showing "which kubectl" output as "/usr/bin/kubectl"`

**Validation Criteria:** A valid filesystem path must be returned. An empty output or a `not found` error indicates `kubectl` is not installed or not on `$PATH`.

---

### Step 3: Validate kubectl Client Version

**Purpose:** Confirm the `kubectl` client version and ensure Kustomize is bundled correctly.

```bash
kubectl version --client
```

**Output:**

```
Client Version: v1.34.1
Kustomize Version: v5.7.1
```

**Screenshot:**

<img width="1030" height="475" alt="image" src="https://github.com/user-attachments/assets/fdb129ae-1cdf-4b7a-946e-9b9118655783" />

> `Terminal displaying kubectl client version v1.34.1 and Kustomize v5.7.1`

**Validation Criteria:** The client version must be within one minor version of the cluster server version (skew policy). A version mismatch beyond this range can cause API compatibility issues.

---

### Step 4: Verify Cluster Connectivity

**Purpose:** Confirm that `kubectl` can communicate with the Kubernetes API server and that core cluster services are running.

```bash
kubectl cluster-info
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

**Screenshot:**

<img width="1030" height="475" alt="image" src="https://github.com/user-attachments/assets/fdb129ae-1cdf-4b7a-946e-9b9118655783" />

> `Terminal output of "kubectl cluster-info" showing control plane and CoreDNS/Metrics-server status`

**Validation Criteria:** All three endpoints (control plane, CoreDNS, Metrics-server) must show a `running` status. A `connection refused` error indicates the API server is down or the kubeconfig endpoint is incorrect.

---

### Step 5: Inspect Cluster Nodes

**Purpose:** List all nodes in the cluster and verify their `Ready` status.

```bash
kubectl get nodes
```

**Output:**

```
NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   45m   v1.34.1+k3s1
```

**Screenshot:**

<img width="1030" height="475" alt="image" src="https://github.com/user-attachments/assets/fdb129ae-1cdf-4b7a-946e-9b9118655783" />

> `Terminal showing "kubectl get nodes" with jump-host in Ready status`

**Validation Criteria:** All nodes must show `STATUS: Ready`. A `NotReady` status requires immediate investigation using `kubectl describe node <node-name>` before proceeding.

---

### Step 6: Review Kubeconfig Context

**Purpose:** Inspect the active kubeconfig to confirm the correct cluster, user, and authentication credentials are bound.

```bash
kubectl config view --minify
```

**Output:**

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
users:
- name: default
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

**Screenshot:**

<img width="1030" height="743" alt="image" src="https://github.com/user-attachments/assets/e5186454-4db8-4c4c-813e-8b14128a949f" />

> `Terminal output of "kubectl config view --minify" showing the default context, cluster, and user`

**Validation Criteria:** The `server` field must point to `https://127.0.0.1:6443`. Certificate data fields should show `DATA+OMITTED` (not empty), confirming TLS credentials are present.

---

### Step 7: Confirm Active Context

**Purpose:** List all available contexts and confirm which one is currently active (marked with `*`).

```bash
kubectl config get-contexts
```

**Output:**

```
CURRENT   NAME      CLUSTER   AUTHINFO   NAMESPACE
*         default   default   default
```

**Screenshot:**

<img width="1031" height="836" alt="image" src="https://github.com/user-attachments/assets/979d65c9-fee1-4d12-a899-6ea2d77760ff" />

> `Terminal output of "kubectl config get-contexts" showing the asterisk on the "default" context`

**Validation Criteria:** The `*` must appear next to the intended context. If it is absent or on the wrong context, switch using `kubectl config use-context <context-name>`.

---

### Step 8: Identify Pre-existing State (Error Encountered)

**Purpose:** Check whether an `httpd` deployment already exists in the `default` namespace before attempting creation.

```bash
kubectl get deployment httpd
```

**Output (Error):**

```
Error from server (NotFound): deployments.apps "httpd" not found
```

**Screenshot:**

<img width="1031" height="836" alt="image" src="https://github.com/user-attachments/assets/979d65c9-fee1-4d12-a899-6ea2d77760ff" />

> `Terminal output showing the "NotFound" error for the httpd deployment`

#### Error Analysis

| Field | Detail |
|---|---|
| **Error Type** | `NotFound` (HTTP 404 equivalent) |
| **Resource** | `deployments.apps "httpd"` |
| **Namespace** | `default` |
| **Root Cause** | No `httpd` deployment existed prior to this operation |
| **Severity** | Informational - expected when starting fresh |
| **Resolution** | Proceed to create the deployment in Step 9 |

**Note:** This is an **expected and non-blocking error**. It confirms the namespace is clean and no naming conflict exists. In production environments, always check for pre-existing resources to prevent duplicate deployments or unintended overwrites.

---

### Step 9: Create the HTTPD Deployment

**Purpose:** Create a Kubernetes `Deployment` named `httpd` using the `httpd:latest` image. The image tag `latest` is explicitly specified per task requirements.

```bash
kubectl create deployment httpd --image=httpd:latest
```

**Output:**

```
deployment.apps/httpd created
```

**Screenshot Placeholder:**
> `[SCREENSHOT: Terminal showing "deployment.apps/httpd created" confirmation message]`

**What This Command Does:**

* Creates a `Deployment` object named `httpd` in the `default` namespace
* Sets the deployment selector and pod template label to `app=httpd`
* Instructs the Deployment controller to maintain 1 replica (default)
* Pulls the `httpd:latest` image from Docker Hub
* Automatically creates a backing `ReplicaSet`

---

### Step 10: Verify Deployment Status

**Purpose:** Confirm the deployment has rolled out successfully with the desired number of replicas available.

```bash
kubectl get deployment httpd
```

**Output:**

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
httpd   1/1     1            1           2m15s
```

**Screenshot Placeholder:**
> `[SCREENSHOT: Terminal output of "kubectl get deployment httpd" showing 1/1 READY and 1 AVAILABLE]`

**Column Definitions:**

| Column | Value | Meaning |
|---|---|---|
| `READY` | `1/1` | 1 replica running out of 1 desired |
| `UP-TO-DATE` | `1` | 1 replica updated to the latest pod template |
| `AVAILABLE` | `1` | 1 replica available to serve traffic |
| `AGE` | `2m15s` | Time since deployment creation |

**Validation Criteria:** `READY` must show `1/1` and `AVAILABLE` must be `1`. Any value of `0` requires investigation.

---

### Step 11: Inspect the ReplicaSet

**Purpose:** Verify that the Deployment controller automatically created a ReplicaSet and that it maintains the desired pod count.

```bash
kubectl get replicaset
```

**Output:**

```
NAME               DESIRED   CURRENT   READY   AGE
httpd-6c755866c7   1         1         1       2m28s
```

**Screenshot Placeholder:**
> `[SCREENSHOT: Terminal showing the ReplicaSet "httpd-6c755866c7" with 1/1/1 desired/current/ready]`

**Validation Criteria:** `DESIRED`, `CURRENT`, and `READY` must all be `1`. A ReplicaSet with `0` ready pods while the deployment shows available indicates an image pull failure or resource constraint.

---

### Step 12: Verify Running Pods

**Purpose:** Confirm that the Pod managed by the ReplicaSet is in `Running` status with `0` restarts.

```bash
kubectl get pods
```

**Output:**

```
NAME                     READY   STATUS    RESTARTS   AGE
httpd-6c755866c7-zc2js   1/1     Running   0          2m41s
```

**Screenshot Placeholder:**
> `[SCREENSHOT: Terminal showing Pod "httpd-6c755866c7-zc2js" in Running status with 0 restarts]`

**Pod Naming Convention:**

```
httpd               - Deployment name
6c755866c7          - ReplicaSet hash (derived from pod template spec)
zc2js               - Unique pod suffix (randomly generated)
```

**Validation Criteria:** `STATUS` must be `Running`, `READY` must be `1/1`, and `RESTARTS` must be `0`. Repeated restarts indicate a crash-looping container.

---

### Step 13: Describe the Deployment

**Purpose:** Perform a deep inspection of the Deployment object, reviewing its strategy, conditions, and event history.

```bash
kubectl describe deployment httpd
```

**Output (Key Sections):**

```
Name:                   httpd
Namespace:              default
CreationTimestamp:      Fri, 27 Mar 2026 04:33:33 +0000
Labels:                 app=httpd
Selector:               app=httpd
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
RollingUpdateStrategy:  25% max unavailable, 25% max surge

Conditions:
  Type           Status  Reason
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable

Events:
  Normal  ScalingReplicaSet  3m20s  deployment-controller  Scaled up replica set httpd-6c755866c7 from 0 to 1
```

**Screenshot Placeholder:**
> `[SCREENSHOT: Terminal output of "kubectl describe deployment httpd" showing conditions and events]`

**Key Observations:**

| Field | Value | Significance |
|---|---|---|
| `StrategyType` | `RollingUpdate` | Zero-downtime rolling updates enabled by default |
| `Max Unavailable` | `25%` | At most 0 pods unavailable during updates (rounded down from 0.25) |
| `Max Surge` | `25%` | At most 1 extra pod during updates (rounded up from 0.25) |
| `Available Condition` | `True` | Minimum replicas are available |
| `Progressing Condition` | `True` | New ReplicaSet is available |

---

### Step 14: Describe the Pod

**Purpose:** Perform a deep inspection of the running Pod, validating image identity, resource binding, container state, and event timeline.

```bash
kubectl describe pod $(kubectl get pod -l app=httpd -o jsonpath='{.items[0].metadata.name}')
```

**Output (Key Sections):**

```
Name:             httpd-6c755866c7-zc2js
Namespace:        default
Node:             jump-host/10.244.195.206
Start Time:       Fri, 27 Mar 2026 04:33:33 +0000
Labels:           app=httpd
                  pod-template-hash=6c755866c7
Status:           Running
IP:               10.22.0.9

Containers:
  httpd:
    Image:          httpd:latest
    Image ID:       docker.io/library/httpd@sha256:331548c5249bdeced0f048bc2fb8c6b6427d2ec6508bed9c1fec6c57d0b27a60
    State:          Running
    Ready:          True
    Restart Count:  0

Conditions:
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True

Events:
  Normal  Scheduled  Successfully assigned default/httpd-6c755866c7-zc2js to jump-host
  Normal  Pulling    Pulling image "httpd:latest"
  Normal  Pulled     Successfully pulled image "httpd:latest" in 3.053s (45239959 bytes)
  Normal  Created    Created container: httpd
  Normal  Started    Started container httpd
```

**Screenshot Placeholder:**
> `[SCREENSHOT: Terminal output of "kubectl describe pod" showing image ID, all conditions True, and event timeline]`

**Critical Fields Verified:**

| Field | Value | Significance |
|---|---|---|
| `Status` | `Running` | Container is active |
| `Image ID` | `sha256:331548c5...` | Immutable digest confirms exact image pulled |
| `Restart Count` | `0` | No crashes since start |
| `Image Pull Time` | `3.053s` | Image pulled fresh (not from cache) |
| `Image Size` | `45,239,959 bytes (~45 MB)` | Consistent with official httpd image |
| `All Conditions` | `True` | Pod is fully initialized, scheduled, and ready |

**Command Breakdown:**

```bash
kubectl get pod -l app=httpd -o jsonpath='{.items[0].metadata.name}'
# Dynamically resolves the pod name using the app=httpd label selector
# Avoids hardcoding the randomly generated pod suffix
```

---

### Step 15: Final Validation of All Resources

**Purpose:** Execute a single unified query to confirm all three Kubernetes objects (Pod, Deployment, ReplicaSet) are healthy and associated with the `app=httpd` label.

```bash
kubectl get all -l app=httpd
```

**Output:**

```
NAME                         READY   STATUS    RESTARTS   AGE
pod/httpd-6c755866c7-zc2js   1/1     Running   0          5m36s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/httpd   1/1     1            1           5m36s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/httpd-6c755866c7   1         1         1       5m36s
```

**Screenshot Placeholder:**
> `[SCREENSHOT: Terminal showing all three resources (pod, deployment, replicaset) healthy under "kubectl get all -l app=httpd"]`

**Final State Summary:**

| Resource | Name | Status |
|---|---|---|
| Pod | `httpd-6c755866c7-zc2js` | Running, 1/1 Ready, 0 Restarts |
| Deployment | `httpd` | 1/1 Ready, 1 Available, 1 Up-to-Date |
| ReplicaSet | `httpd-6c755866c7` | 1 Desired, 1 Current, 1 Ready |

**Deployment objective achieved.** The `httpd` application is running successfully on the Kubernetes cluster.

---

## Error Log and Resolution

| Step | Command | Error Message | Root Cause | Resolution | Severity |
|---|---|---|---|---|---|
| 8 | `kubectl get deployment httpd` | `Error from server (NotFound): deployments.apps "httpd" not found` | No `httpd` deployment existed in the `default` namespace at the time of query | Proceed with `kubectl create deployment` in Step 9 | Informational |

**No blocking or critical errors were encountered during this deployment.**

The `NotFound` error at Step 8 was an **expected pre-flight check result**, confirming a clean namespace state before resource creation.

---

## Best Practices

### 1. Always Verify Your Environment Before Acting

Run `hostname`, `kubectl config get-contexts`, and `kubectl cluster-info` before any cluster operation. A single context mismatch can result in deploying resources to the wrong cluster or namespace.

### 2. Always Specify Image Tags Explicitly

```bash
# Avoid - "latest" is implicit and non-deterministic in production
kubectl create deployment httpd --image=httpd

# Preferred for production - pin to a specific digest or semantic version
kubectl create deployment httpd --image=httpd:2.4.62-alpine
```

Using `httpd:latest` is acceptable for lab and onboarding scenarios. In production, always pin images to a specific version or SHA256 digest to guarantee reproducibility.

### 3. Use Label Selectors for Dynamic Pod Resolution

```bash
# Avoid - hardcoded pod name breaks after any pod restart
kubectl describe pod httpd-6c755866c7-zc2js

# Preferred - label selector is always correct regardless of pod suffix
kubectl describe pod $(kubectl get pod -l app=httpd -o jsonpath='{.items[0].metadata.name}')
```

### 4. Use `kubectl get all -l <label>` for Unified Resource Validation

Rather than querying Pod, Deployment, and ReplicaSet individually, use a single label-scoped query for a holistic view of resource health.

### 5. Inspect Image SHA256 Digests

When reviewing `kubectl describe pod`, always note the `Image ID` field. This SHA256 digest is the immutable fingerprint of the exact image layer pulled. It enables precise audit tracking and reproducibility verification.

### 6. Never Skip `kubectl describe` in Post-Deployment Validation

The `Events` section of `kubectl describe deployment` and `kubectl describe pod` contains the ground truth of what occurred during scheduling, image pull, and container start. It is the first place to look when a deployment behaves unexpectedly.

### 7. Define Resource Requests and Limits in Production

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

`kubectl create deployment` does not set resource requests or limits. In production, always define these to prevent resource starvation and ensure the scheduler places pods correctly.

### 8. Prefer Declarative Manifests Over Imperative Commands in Production

```bash
# Imperative (used here for speed in lab environments)
kubectl create deployment httpd --image=httpd:latest

# Declarative (preferred for production, GitOps, and auditability)
kubectl apply -f httpd-deployment.yaml
```

Declarative manifests enable version control, peer review, drift detection, and GitOps workflows.

---

## Lessons Learned

### Lesson 1: Pre-flight Checks Prevent Production Incidents

Running `kubectl get deployment httpd` before creation confirmed the namespace was clean. In environments where automation runs on shared clusters, skipping this step risks duplicate deployments, naming conflicts, or unintended resource updates.

### Lesson 2: The `NotFound` Error is a Diagnostic Signal, Not a Failure

The error `deployments.apps "httpd" not found` at Step 8 was not a problem. It was a confirmation. Teams that treat every non-zero exit code as a failure without reading the error message introduce unnecessary rollback procedures and incident noise.

### Lesson 3: K3s is a Production-Grade Single-Binary Kubernetes

K3s is not a toy cluster. It runs the full Kubernetes API surface with the same `kubectl` toolchain. What is learned here transfers directly to multi-node EKS, GKE, and AKS environments. The only difference is scale.

### Lesson 4: Image Pull Telemetry Lives in Pod Events

The `Events` section of `kubectl describe pod` revealed that `httpd:latest` was pulled fresh in `3.053s` at `45,239,959 bytes`. This confirms the image was not served from a local cache. In air-gapped or bandwidth-constrained environments, this timing data helps diagnose slow deployments.

### Lesson 5: `kubectl get all` is Scope-Limited

`kubectl get all` does not return truly all resources. It returns the most common ones (Pods, Services, Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs, CronJobs). Resources like `ConfigMaps`, `Secrets`, `PersistentVolumeClaims`, and `Ingress` are not included. For a comprehensive inventory, use `kubectl api-resources` and query each type explicitly.

### Lesson 6: Single-Node Clusters Have No Scheduling Redundancy

With only `jump-host` as the sole control-plane and worker node, there is no node affinity separation, no topology spread, and no failure domain isolation. Any node failure brings down both the control plane and all workloads simultaneously. This is acceptable for lab use but must be addressed before production promotion.

### Lesson 7: Declarative Reproducibility Matters from Day One

Even in a lab, building the habit of saving deployment definitions to YAML enables faster recovery, peer review, and environment portability. The imperative commands used here should be exported immediately:

```bash
kubectl get deployment httpd -o yaml > httpd-deployment.yaml
```

---

## References

* [Kubernetes Official Documentation](https://kubernetes.io/docs/)
* [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
* [K3s Documentation](https://docs.k3s.io/)
* [Apache HTTPD Docker Hub](https://hub.docker.com/_/httpd)
* [Kubernetes Deployment Concepts](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
* [kubectl config Contexts](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

---






<img width="1028" height="859" alt="image" src="https://github.com/user-attachments/assets/c6a184fb-d8ae-4164-bbf5-298cb7ad30f1" />
<img width="1032" height="766" alt="image" src="https://github.com/user-attachments/assets/238d1e85-aa0e-4529-a1b4-0f6565192904" />
<img width="1035" height="858" alt="image" src="https://github.com/user-attachments/assets/9a6f3a57-8729-4a7f-aa20-2b4abb123ac8" />
<img width="1028" height="864" alt="image" src="https://github.com/user-attachments/assets/76ed7ac8-81a9-4ad1-bd9d-1779f879106d" />
<img width="1034" height="859" alt="image" src="https://github.com/user-attachments/assets/585ddcb5-8263-4d63-904e-15c2fe91ba06" />
<img width="1030" height="567" alt="image" src="https://github.com/user-attachments/assets/fe979471-377e-4afe-82ee-739908da5335" />
