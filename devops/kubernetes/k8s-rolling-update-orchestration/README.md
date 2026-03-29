# Kubernetes Rolling Update: nginx Deployment Image Upgrade (nginx:1.16 to nginx:1.17)

---

## Table of Contents

- [Overview](#overview)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Environment Details](#environment-details)
- [Problem Statement](#problem-statement)
- [Execution Walkthrough](#execution-walkthrough)
  - [Phase 1: Cluster Health Verification](#phase-1-cluster-health-verification)
  - [Phase 2: Deployment State Inspection](#phase-2-deployment-state-inspection)
  - [Phase 3: Label Discovery and Pod Investigation](#phase-3-label-discovery-and-pod-investigation)
  - [Phase 4: ReplicaSet and Container Configuration Audit](#phase-4-replicaset-and-container-configuration-audit)
  - [Phase 5: Rolling Update Execution](#phase-5-rolling-update-execution)
  - [Phase 6: Rollout Status Monitoring](#phase-6-rollout-status-monitoring)
  - [Phase 7: Post-Update Verification](#phase-7-post-update-verification)
  - [Phase 8: Rollout History Audit](#phase-8-rollout-history-audit)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)

---

## Overview

This document captures the end-to-end execution of a Kubernetes rolling update performed on the `nginx-deployment` workload within the **Nautilus** application development cluster. The operation upgraded the container image from `nginx:1.16` to `nginx:1.17` using `kubectl set image` to trigger a zero-downtime rolling replacement of all running pods.

| Field | Value |
|---|---|
| Deployment Name | `nginx-deployment` |
| Container Name | `nginx-container` |
| Previous Image | `nginx:1.16` |
| Target Image | `nginx:1.17` |
| Replica Count | 3 |
| Namespace | `default` |
| Update Strategy | RollingUpdate (Kubernetes default) |
| Cluster Endpoint | `https://127.0.0.1:6443` |
| Executed By | `thor` on `jump-host` |

---

## Architecture Context

```
jump-host (kubectl client)
        |
        v
Kubernetes Control Plane (https://127.0.0.1:6443)
        |
        +-- nginx-deployment (3 replicas)
                |
                +-- nginx-deployment-fc677cbc9-* (OLD: nginx:1.16)  [Terminated post-update]
                +-- nginx-deployment-544f9896c8-* (NEW: nginx:1.17) [Running post-update]
```

The rolling update replaces pods controlled by the old ReplicaSet (`fc677cbc9`) with pods under a new ReplicaSet (`544f9896c8`), maintaining availability throughout.

---

## Prerequisites

- `kubectl` installed and configured on `jump-host`
- Kubeconfig pointed to the target cluster at `https://127.0.0.1:6443`
- Sufficient RBAC permissions to patch Deployments and view Pods/ReplicaSets
- Active internet or local registry access to pull `nginx:1.17`

---

## Environment Details

| Component | Detail |
|---|---|
| Host | `jump-host` |
| User | `thor` |
| Kubernetes Control Plane | `https://127.0.0.1:6443` |
| CoreDNS | Running via `kube-dns` service proxy |
| Metrics Server | Running via HTTPS proxy |
| Target Namespace | `default` |

---

## Problem Statement

The Nautilus application development team introduced updates to the nginx web server configuration and packaged them into the `nginx:1.17` image. The active deployment `nginx-deployment` was running `nginx:1.16` across 3 pods. The task required executing a rolling update to replace all pod instances with the `nginx:1.17` image while ensuring all pods remained operational post-update, with zero downtime.

---

## Execution Walkthrough

---

### Phase 1: Cluster Health Verification

Before making any changes to a live workload, cluster connectivity and control plane health were confirmed.

```bash
thor@jump-host ~$ kubectl cluster-info
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

**Observation:** The control plane, CoreDNS, and Metrics Server were all confirmed active. The cluster was healthy and safe to proceed with workload changes.

> **SCREENSHOT**

<img width="1033" height="626" alt="image" src="https://github.com/user-attachments/assets/d0261268-fb09-4f0f-bb9a-d88f604e723c" />

> *Full terminal output of `kubectl cluster-info` showing all components running*

---

### Phase 2: Deployment State Inspection

The current state of the `nginx-deployment` was inspected to confirm replica readiness before the update.

```bash
thor@jump-host ~$ kubectl get deployment nginx-deployment
```

**Output:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           10m
```

**Observation:** All 3 replicas were ready and available. The deployment was in a fully healthy state before the update.

```bash
thor@jump-host ~$ kubectl describe deployment nginx-deployment | grep -i image
```

**Output:**

```
    Image:         nginx:1.16
```

**Observation:** The active container image was confirmed as `nginx:1.16`. This was the baseline image to be replaced.

> **SCREENSHOT**

<img width="1035" height="422" alt="image" src="https://github.com/user-attachments/assets/b331eb36-daca-4425-a7bf-6f8c160109aa" />

> *Terminal output of `kubectl get deployment nginx-deployment` and `kubectl describe deployment nginx-deployment | grep -i image` showing 3/3 ready and nginx:1.16*

---

### Phase 3: Label Discovery and Pod Investigation

#### Step 3a: Initial Label Query (Incorrect Selector)

An attempt was made to list pods using the label `app=nginx-deployment`, which returned no results.

```bash
thor@jump-host ~$ kubectl get pods -l app=nginx-deployment
```

**Output:**

```
No resources found in default namespace.
```

**Problem:** The selector `app=nginx-deployment` did not match any pods. This indicated the pods were labeled differently than assumed.

> **SCREENSHOT**

<img width="1028" height="559" alt="image" src="https://github.com/user-attachments/assets/10405386-6c1c-4bd9-8b1a-06d86bff1028" />

> *Terminal showing `No resources found in default namespace.` for the incorrect label selector*

#### Step 3b: Label Discovery via `--show-labels`

To identify the correct pod labels, all pods in the namespace were listed with their full label sets.

```bash
thor@jump-host ~$ kubectl get pods --show-labels
```

**Output:**

```
NAME                               READY   STATUS    RESTARTS   AGE   LABELS
nginx-deployment-fc677cbc9-95ljq   1/1     Running   0          12m   app=nginx-app,pod-template-hash=fc677cbc9
nginx-deployment-fc677cbc9-b5c7l   1/1     Running   0          12m   app=nginx-app,pod-template-hash=fc677cbc9
nginx-deployment-fc677cbc9-cjjmr   1/1     Running   0          12m   app=nginx-app,pod-template-hash=fc677cbc9
```

**Observation:** The pods used the label `app=nginx-app`, not `app=nginx-deployment`. All 3 pods were running under ReplicaSet hash `fc677cbc9`. This discovery was critical for all subsequent pod-targeted verification commands.

> **SCREENSHOT**

<img width="1028" height="559" alt="image" src="https://github.com/user-attachments/assets/10405386-6c1c-4bd9-8b1a-06d86bff1028" />

> *Full `kubectl get pods --show-labels` output showing the correct label `app=nginx-app` and ReplicaSet hash*

---

### Phase 4: ReplicaSet and Container Configuration Audit

#### Step 4a: ReplicaSet State

```bash
thor@jump-host ~$ kubectl get replicaset
```

**Output:**

```
NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-fc677cbc9   3         3         3       13m
```

**Observation:** One ReplicaSet was active with 3 desired, current, and ready replicas.

> **SCREENSHOT**

<img width="1025" height="597" alt="image" src="https://github.com/user-attachments/assets/bca28c4f-db7a-4db2-ace3-bdc82754c68c" />

> *Terminal output of `kubectl get replicaset` showing single ReplicaSet with 3/3 ready*

#### Step 4b: Container Specification Audit

The full container block was inspected to confirm the container name and port configuration before issuing the `set image` command.

```bash
thor@jump-host ~$ kubectl describe deployment nginx-deployment | grep -A5 "Containers:"
```

**Output:**

```
  Containers:
   nginx-container:
    Image:         nginx:1.16
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
```

**Observation:** The container name was `nginx-container`. This was the exact name required for the `kubectl set image` command. The image was still `nginx:1.16` as expected.

> **SCREENSHOT**

<img width="1028" height="658" alt="image" src="https://github.com/user-attachments/assets/37b51236-dbb9-4b1e-ac95-4296044f906b" />

> *Terminal output of the container describe block confirming `nginx-container` name and `nginx:1.16` image*

---

### Phase 5: Rolling Update Execution

With the container name and current image confirmed, the rolling update was issued.

```bash
thor@jump-host ~$ kubectl set image deployment/nginx-deployment nginx-container=nginx:1.17
```

**Output:**

```
deployment.apps/nginx-deployment image updated
```

**Observation:** Kubernetes accepted the image patch and began the rolling update process. The control plane began replacing old pods with new ones running `nginx:1.17`.

> **SCREENSHOT**

<img width="1035" height="718" alt="image" src="https://github.com/user-attachments/assets/51f4329e-3084-431d-bf59-669ae00fa03b" />

> *Terminal showing the `deployment.apps/nginx-deployment image updated` confirmation message*

---

### Phase 6: Rollout Status Monitoring

The rollout was monitored in real time to confirm successful completion before proceeding to verification.

```bash
thor@jump-host ~$ kubectl rollout status deployment/nginx-deployment
```

**Output:**

```
deployment "nginx-deployment" successfully rolled out
```

**Observation:** Kubernetes confirmed the rollout completed successfully. All pods were replaced without error.

> **SCREENSHOT**

<img width="1035" height="718" alt="image" src="https://github.com/user-attachments/assets/51f4329e-3084-431d-bf59-669ae00fa03b" />

> *Terminal showing `deployment "nginx-deployment" successfully rolled out`*

---

### Phase 7: Post-Update Verification

Multiple verification commands were executed to confirm the updated state of the deployment across all layers: deployment, pods, ReplicaSets, and image pull events.

#### Step 7a: Image Verification via Deployment Describe

```bash
thor@jump-host ~$ kubectl describe deployment nginx-deployment | grep -i image
```

**Output:**

```
    Image:         nginx:1.17
```

**Observation:** The deployment spec now shows `nginx:1.17` as the active image.

> **SCREENSHOT**

<img width="1031" height="826" alt="image" src="https://github.com/user-attachments/assets/89f3ae89-ac46-4d25-831b-cbfb7fc0aa8c" />

> *Terminal showing `Image: nginx:1.17` in the describe output*

#### Step 7b: Pod Health Verification with Correct Label

```bash
thor@jump-host ~$ kubectl get pods -l app=nginx-app
```

**Output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-544f9896c8-6xmb2   1/1     Running   0          63s
nginx-deployment-544f9896c8-f5xjv   1/1     Running   0          58s
nginx-deployment-544f9896c8-mn4gl   1/1     Running   0          59s
```

**Observation:** All 3 pods were running under a new ReplicaSet hash `544f9896c8`, confirming the old pods were replaced. All pods showed `1/1 Running` with zero restarts.

> **SCREENSHOT**

<img width="1031" height="826" alt="image" src="https://github.com/user-attachments/assets/89f3ae89-ac46-4d25-831b-cbfb7fc0aa8c" />

> *Terminal showing the 3 new pods under ReplicaSet `544f9896c8` all in Running state*

#### Step 7c: ReplicaSet Transition Verification

```bash
thor@jump-host ~$ kubectl get replicaset
```

**Output:**

```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-544f9896c8   3         3         3       92s
nginx-deployment-fc677cbc9    0         0         0       17m
```

**Observation:** The new ReplicaSet `544f9896c8` was fully scaled to 3. The old ReplicaSet `fc677cbc9` was scaled down to 0 (retained for potential rollback). This is the expected behavior of a Kubernetes rolling update.

> **SCREENSHOT**

<img width="1031" height="856" alt="image" src="https://github.com/user-attachments/assets/3afdd322-efe4-4119-b7dd-b2d19addfcbe" />

> *Terminal output showing both ReplicaSets, new one at 3/3 and old one at 0/0/0*

#### Step 7d: Deep Image Pull Verification via Pod Events

```bash
thor@jump-host ~$ kubectl describe pods -l app=nginx-app | grep -i image
```

**Output:**

```
    Image:          nginx:1.17
    Image ID:       docker.io/library/nginx@sha256:6fff55753e3b34e36e24e37039ee9eae1fe38a6420d8ae16ef37c92d1eb26699
  Normal  Pulling    2m3s  kubelet            Pulling image "nginx:1.17"
  Normal  Pulled     2m    kubelet            Successfully pulled image "nginx:1.17" in 2.883s (2.883s including waiting). Image size: 51030575 bytes.
    Image:          nginx:1.17
    Image ID:       docker.io/library/nginx@sha256:6fff55753e3b34e36e24e37039ee9eae1fe38a6420d8ae16ef37c92d1eb26699
  Normal  Pulled     118s  kubelet            Container image "nginx:1.17" already present on machine
    Image:          nginx:1.17
    Image ID:       docker.io/library/nginx@sha256:6fff55753e3b34e36e24e37039ee9eae1fe38a6420d8ae16ef37c92d1eb26699
  Normal  Pulled     119s  kubelet            Container image "nginx:1.17" already present on machine
```

**Observation:**
- Pod 1 pulled `nginx:1.17` fresh from Docker Hub in 2.883 seconds (51,030,575 bytes)
- Pods 2 and 3 reused the locally cached image ("already present on machine"), reducing pull time
- All 3 pods confirmed running `nginx:1.17` with matching digest `sha256:6fff55...`
- The image digest provides cryptographic confirmation of image integrity

> **SCREENSHOT**

<img width="1026" height="833" alt="image" src="https://github.com/user-attachments/assets/4fa01b8a-6fa4-4d02-8807-c8a72d2d32ba" />

> *Full `kubectl describe pods -l app=nginx-app | grep -i image` output showing image pull events, digest, and cache reuse across all 3 pods*

---

### Phase 8: Rollout History Audit

The deployment's revision history was inspected to confirm the rollout was recorded and to validate rollback availability.

```bash
thor@jump-host ~$ kubectl rollout history deployment/nginx-deployment
```

**Output:**

```
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

**Observation:** Two revisions were recorded. Revision 1 was the original `nginx:1.16` deployment. Revision 2 was the updated `nginx:1.17` rollout. The `CHANGE-CAUSE` field was `<none>` because the `--record` flag or annotation was not used during the update. This is noted as a best practice improvement below.

> **SCREENSHOT**

<img width="1037" height="860" alt="image" src="https://github.com/user-attachments/assets/8c9f1e40-0a44-432b-8cb8-37cfbdc3e8b2" />

> *Terminal showing the two-revision rollout history with CHANGE-CAUSE fields as none*

---
 
### Phase 9: Browser-Level Service Reachability Verification
 
With all pod-level and deployment-level verification complete, the final validation confirmed that the nginx service was reachable from a browser via the exposed NodePort, proving end-to-end operational status of the updated deployment.
 
**Access URL:**
 
```
http://30008-port-ffgz6enhdsim4jwt.labs.kodekloud.com
```
 
**Observation:** The browser returned the default nginx welcome page, confirming:
 
- The nginx service was correctly exposed via NodePort `30008` through the KodeKloud lab environment
- The updated pods running `nginx:1.17` were actively serving HTTP traffic
- No application-layer disruption occurred as a result of the rolling update
- The deployment was fully operational from the end-user perspective
 
This browser-level test is the definitive proof-of-life check that goes beyond CLI verification. A pod in `Running` state at the Kubernetes layer does not guarantee the application is actually serving traffic. The successful HTTP response closes the verification loop entirely.
 
> **SCREENSHOT**

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/9430a2f1-b309-42c9-889e-44a47e3f0a21" />

> *Browser window showing `Welcome to nginx!` page at `30008-port-ffgz6enhdsim4jwt.labs.kodekloud.com`, confirming the nginx:1.17 pods are actively serving HTTP traffic via NodePort 30008*
 
---

## Errors Encountered and Resolutions

### Error 1: Incorrect Pod Label Selector Returns No Results

**Command Attempted:**

```bash
kubectl get pods -l app=nginx-deployment
```

**Output:**

```
No resources found in default namespace.
```

**Root Cause:** The pod label selector `app=nginx-deployment` was assumed based on the deployment name. However, the deployment's pod template was configured with a different label: `app=nginx-app`. This is a common operational assumption error when working with pre-existing deployments that were not created by the current operator.

**Resolution:** The correct labels were discovered by running:

```bash
kubectl get pods --show-labels
```

This revealed the actual label `app=nginx-app`, which was used for all subsequent pod-targeted commands.

**Prevention:** Always inspect pod labels using `--show-labels` or `kubectl describe deployment <name>` under the `Pod Template > Labels` section before constructing label selectors.

---

## Best Practices

### 1. Always Verify Cluster Health Before Workload Changes

Run `kubectl cluster-info` before any update operation to confirm the control plane and critical cluster services are responsive. A degraded control plane can cause incomplete rollouts.

### 2. Inspect Actual Pod Labels Before Constructing Selectors

Never assume pod labels match deployment names. Use `kubectl get pods --show-labels` or inspect the deployment's `Pod Template > Labels` section with `kubectl describe deployment <name>` before using `-l` selectors.

### 3. Confirm the Exact Container Name Before `set image`

The `kubectl set image` command requires the exact container name as defined in the pod spec, not the deployment name. Always confirm with:

```bash
kubectl describe deployment <name> | grep -A5 "Containers:"
```

### 4. Monitor Rollout Status in Real Time

Use `kubectl rollout status deployment/<name>` after every `set image` command. This blocks until the rollout completes or fails, providing a reliable gate before running post-update verification steps.

### 5. Verify Image Integrity via Digest

Post-update, confirm the running image digest using `kubectl describe pods -l <selector> | grep -i image`. The sha256 digest provides cryptographic verification that the correct image was pulled and is running.

### 6. Annotate Rollout Cause for Auditability

Use `kubectl annotate deployment/<name> kubernetes.io/change-cause="<description>"` immediately after a rollout to populate the `CHANGE-CAUSE` field in `kubectl rollout history`. Without this, revision history becomes non-auditable.

Example:

```bash
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="Upgrade nginx from 1.16 to 1.17 per Nautilus team release"
```

### 7. Validate Old ReplicaSet Retention for Rollback Readiness

After a rolling update, confirm the old ReplicaSet is retained at 0 replicas (not deleted). Kubernetes preserves it for rollback. Verify with:

```bash
kubectl get replicaset
```

If a rollback is needed:

```bash
kubectl rollout undo deployment/nginx-deployment
```

### 8. Leverage Image Caching for Multi-Pod Deployments

On single-node or small clusters, only the first pod pull fetches the image from the registry. Subsequent pods reuse the locally cached layer. This reduces rollout time and external bandwidth. This behavior was observed in this lab with pods 2 and 3 using the cached `nginx:1.17` image.

---

## Lessons Learned

### 1. Pod Labels Are Defined in the Pod Template, Not the Deployment Name

The deployment name and the pod label selector are independent constructs. The pod label (`app=nginx-app`) was configured at deployment creation time and did not match the deployment name (`nginx-deployment`). Always treat pod labels as an unknown until explicitly verified.

### 2. `kubectl get pods --show-labels` Is a Non-Negotiable Diagnostic Tool

When a pod selector returns no results, the immediate next step is `--show-labels`. This single command exposes the full label map and the ReplicaSet hash, enabling accurate targeting for all subsequent operations.

### 3. Rolling Updates Preserve the Old ReplicaSet

Kubernetes scales down the old ReplicaSet to 0 but retains it in the cluster. This is an intentional design choice supporting rollback via `kubectl rollout undo`. Understanding this behavior prevents confusion when two ReplicaSets appear in `kubectl get replicaset` output post-update.

### 4. Image Digest Verification Is a Production Standard

Verifying the sha256 image digest after a rollout ensures the correct and untampered image version is running. Digest verification is especially important in regulated environments where image provenance must be auditable.

### 5. The `--record` Flag and Change-Cause Annotation Matter in Team Environments

The `CHANGE-CAUSE` field being `<none>` in revision history creates an operational blind spot. In team or production environments, every rollout should be annotated with a meaningful change cause to support incident response and audit trails.

### 6. Pre-Update Baseline Capture Enables Faster Incident Response

Capturing the pre-update image, replica count, ReplicaSet name, and pod labels before the update creates a rollback reference that does not depend on cluster state. If a rollout leaves the cluster partially degraded, this baseline enables faster recovery.

---
