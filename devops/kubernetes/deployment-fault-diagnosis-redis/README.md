# Kubernetes Redis Deployment Fault Diagnosis and Repair

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Summary](#solution-summary)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Cluster Verification](#phase-1-cluster-verification)
  - [Phase 2: Container Image Correction](#phase-2-container-image-correction)
  - [Phase 3: ConfigMap Reference Repair](#phase-3-configmap-reference-repair)
  - [Phase 4: Rollout Verification](#phase-4-rollout-verification)
  - [Phase 5: Deployment and Pod State Confirmation](#phase-5-deployment-and-pod-state-confirmation)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Lessons Learned](#lessons-learned)
- [Environment Reference](#environment-reference)

---

## Overview

This document details the fault diagnosis and remediation of a broken `redis-deployment` on a live Kubernetes cluster. The deployment had been intentionally modified by a team member in a way that caused all pods to fail. Two distinct faults were identified and corrected in-place without deleting or recreating the deployment: a misconfigured container image and a broken ConfigMap reference caused by a typo in the manifest.

---

## Problem Statement

The Nautilus DevOps team reported that a previously healthy Redis application deployed on a Kubernetes cluster went down following a change introduced by a team member. The deployment, named `redis-deployment`, had its pods in a non-running state. The root cause was unknown at the time of escalation and required live investigation.

**Observed symptoms:**

* Pods under `redis-deployment` were not in `Running` state
* No explicit error message was provided; investigation was required to identify all faults

**Constraints:**

* The `kubectl` utility on the jump-host was pre-configured to communicate with the Kubernetes control plane at `https://127.0.0.1:6443`
* All remediation was performed without destroying or recreating the deployment

---

## Solution Summary

Two independent faults were identified in the `redis-deployment` manifest:

* **Fault 1:** The container image was incorrect. The `redis-container` was referencing an invalid or unavailable image, preventing pod scheduling and startup.
* **Fault 2:** A typo in the ConfigMap reference within the deployment manifest (`redis-conig` instead of `redis-config`) caused a configuration binding failure.

Both issues were resolved using targeted `kubectl` commands, and the deployment was confirmed healthy with all pods in `Running` state.

---

## Architecture Context

* **Platform:** Kubernetes (k3s cluster)
* **Control Plane Endpoint:** `https://127.0.0.1:6443`
* **Access Method:** `kubectl` via jump-host (`thor@jump-host`)
* **Workload Type:** Kubernetes `Deployment`
* **Deployment Name:** `redis-deployment`
* **Container Name:** `redis-container`
* **Corrected Image:** `redis:alpine`
* **ConfigMap Key (corrected):** `redis-config`

---

## Prerequisites

* `kubectl` configured on the jump-host with cluster access
* Appropriate RBAC permissions to patch and apply changes to `Deployment` resources in the target namespace
* Network connectivity to the container image registry (to pull `redis:alpine`)

---

## Implementation Guide

### Phase 1: Cluster Verification

Before performing any remediation, the health and reachability of the Kubernetes control plane was confirmed.

```bash
kubectl cluster-info
```

**Terminal output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

The control plane was reachable and all core system services (CoreDNS, Metrics Server) were operational. Remediation proceeded.

*Screenshot: kubectl cluster-info confirming control plane reachability at 127.0.0.1:6443*

---

### Phase 2: Container Image Correction

The first fault identified was an incorrect container image assigned to `redis-container`. The `kubectl set image` command was used to perform a surgical, in-place image update without requiring a full manifest rewrite or deployment recreation.

```bash
kubectl set image deployment/redis-deployment redis-container=redis:alpine
```

**Terminal output:**

```
deployment.apps/redis-deployment image updated
```

The `redis:alpine` image is the official, lightweight Redis image published by the Docker Official Images program. It is appropriate for production workloads where minimal image surface area is a priority.

*Screenshot: kubectl set image command confirming image updated for redis-container*

---

### Phase 3: ConfigMap Reference Repair

The second fault was a typographical error in the deployment manifest. The ConfigMap reference was misspelled as `redis-conig` instead of the correct `redis-config`. This mismatch would cause Kubernetes to fail when attempting to resolve the ConfigMap binding during pod initialization.

The manifest was extracted, the typo corrected in-flight using `sed`, and the corrected manifest was re-applied using `kubectl apply`.

```bash
kubectl get deployment redis-deployment -o yaml \
  | sed 's/redis-conig/redis-config/' \
  | kubectl apply -f -
```

**Terminal output:**

```
deployment.apps/redis-deployment configured
```

This approach is preferred over editing the manifest manually and re-applying from a file because it eliminates intermediate file management and keeps the correction atomic and auditable within the shell history.

*Screenshot: kubectl get/sed/apply pipeline correcting the redis-config typo in the deployment manifest*

---

### Phase 4: Rollout Verification

After both corrections were applied, the rollout status was monitored to confirm that Kubernetes successfully reconciled the deployment and brought all pods to a healthy state.

```bash
kubectl rollout status deployment redis-deployment
```

**Terminal output:**

```
deployment "redis-deployment" successfully rolled out
```

A successful rollout status confirms that the desired pod count was met, the new pod template was accepted, and no pods were stuck in a pending or crash loop state.

*Screenshot: kubectl rollout status confirming redis-deployment successfully rolled out*

---

### Phase 5: Deployment and Pod State Confirmation

Final verification was performed at both the pod level and the deployment level to confirm the end state matched the desired configuration.

**Pod-level verification:**

```bash
kubectl get pods
```

**Terminal output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
redis-deployment-5476b4ddd6-7tnf6   1/1     Running   0          73s
```

**Deployment-level verification:**

```bash
kubectl get deployment redis-deployment
```

**Terminal output:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
redis-deployment   1/1     1            1           8m28s
```

All availability indicators confirmed healthy state: `READY 1/1`, `UP-TO-DATE 1`, `AVAILABLE 1`. The pod reported `0` restarts, confirming the container started cleanly without crash-loop behavior.

*Screenshot: kubectl get pods showing redis-deployment pod in Running state with 0 restarts*

*Screenshot: kubectl get deployment showing redis-deployment with READY 1/1 and AVAILABLE 1*

---

## Errors and Resolutions

| Fault | Root Cause | Resolution |
|---|---|---|
| Pods not in Running state | Incorrect container image assigned to `redis-container` | Corrected image to `redis:alpine` using `kubectl set image` |
| Configuration binding failure | Typo `redis-conig` in ConfigMap reference within deployment manifest | Piped manifest through `sed` to correct the reference to `redis-config` and re-applied with `kubectl apply` |

---

## Key Decisions

**In-place remediation over redeployment**

Rather than deleting and recreating the deployment, both faults were corrected surgically using `kubectl set image` and a `kubectl get | sed | kubectl apply` pipeline. This approach preserves deployment history, avoids unnecessary downtime windows, and leaves the rollout audit trail intact.

**`sed` for manifest patching over manual file editing**

Extracting the manifest, patching it with `sed`, and piping directly into `kubectl apply` avoids creating intermediate files, reduces the risk of introducing new errors through manual editing, and keeps the operation repeatable and scriptable.

**`redis:alpine` as the corrected image**

The `alpine`-based Redis image was chosen because it is the official lightweight variant maintained by the Docker Official Images team. It carries a minimal attack surface, fast pull times, and is widely accepted in production Kubernetes environments where image hygiene is enforced.

**Rollout status gating before final verification**

`kubectl rollout status` was used as an explicit gate before running final pod and deployment state checks. This ensures the verification step reflects a stable, converged state rather than an in-progress rollout.

---

## Lessons Learned

**Typos in ConfigMap references cause silent initialization failures**

A single character difference between `redis-conig` and `redis-config` is enough to prevent a pod from mounting its configuration, causing it to fail before the application process even starts. Manifest authoring should be validated with linting tools such as `kubeval` or `kustomize build` before applying to a cluster.

**Two independent faults can exist simultaneously**

In this incident, both the image and the ConfigMap reference were broken at the same time. Fixing only one would not have resolved the issue. When diagnosing pod failures, it is important to inspect the full pod spec and describe output rather than stopping at the first fault found.

**`kubectl set image` is the fastest surgical tool for image-only corrections**

When the only required change is an image tag, `kubectl set image` is faster, less error-prone, and easier to audit than extracting and re-applying a full manifest. It should be the default choice for image hotfixes in production incidents.

**`kubectl rollout status` provides a reliable convergence signal**

Rather than polling `kubectl get pods` repeatedly, `kubectl rollout status` blocks until the deployment reaches its desired state or fails, making it a reliable checkpoint in both manual incident response and CI/CD pipelines.

**In-place repair preserves deployment history**

Deleting and recreating a deployment loses all rollout history and makes `kubectl rollout undo` unavailable. Where possible, surgical correction with apply or patch commands keeps the deployment object and its revision history intact.

---

## Environment Reference

| Property | Value |
|---|---|
| Cluster Access Host | `thor@jump-host` |
| Control Plane Endpoint | `https://127.0.0.1:6443` |
| Deployment Name | `redis-deployment` |
| Container Name | `redis-container` |
| Corrected Container Image | `redis:alpine` |
| Corrected ConfigMap Reference | `redis-config` |
| Final Pod Status | `Running` |
| Pod Restarts | `0` |
| Deployment Ready State | `1/1` |





<img width="1143" height="502" alt="image" src="https://github.com/user-attachments/assets/2e0433d6-1777-4432-b4ad-c1ec40099281" />
<img width="1143" height="465" alt="image" src="https://github.com/user-attachments/assets/3f98ced0-6aed-4948-88b6-a0b1b4df03a9" />
<img width="1145" height="499" alt="image" src="https://github.com/user-attachments/assets/885fc47a-e846-4f2f-8e06-6a85465a451d" />
<img width="1144" height="643" alt="image" src="https://github.com/user-attachments/assets/34880afb-1a9f-4ada-b844-3c26928282ce" />
<img width="1140" height="599" alt="image" src="https://github.com/user-attachments/assets/4a75a384-e94c-4f0b-ae98-a1670685e8c2" />
<img width="1142" height="597" alt="image" src="https://github.com/user-attachments/assets/3258bd92-6b02-4b35-aeac-08ce4a832cd9" />
