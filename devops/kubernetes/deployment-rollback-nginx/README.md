# Kubernetes Deployment Rollback: Reverting a Faulty Release Using kubectl rollout undo

## Table of Contents

* [Project Overview](#project-overview)
* [Problem Statement](#problem-statement)
* [Architecture and Environment](#architecture-and-environment)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify Cluster Connectivity and Context](#step-1-verify-cluster-connectivity-and-context)
  * [Step 2: Confirm the Existing Deployment State](#step-2-confirm-the-existing-deployment-state)
  * [Step 3: Inspect the Full Deployment Configuration](#step-3-inspect-the-full-deployment-configuration)
  * [Step 4: Examine the Rollout History](#step-4-examine-the-rollout-history)
  * [Step 5: Inspect Individual Revisions](#step-5-inspect-individual-revisions)
  * [Step 6: Execute the Rollback](#step-6-execute-the-rollback)
  * [Step 7: Confirm Rollout Completion](#step-7-confirm-rollout-completion)
  * [Step 8: Verify the Updated Rollout History](#step-8-verify-the-updated-rollout-history)
  * [Step 9: Confirm the Active Image Version](#step-9-confirm-the-active-image-version)
  * [Step 10: Validate Pod Health and Image Consistency](#step-10-validate-pod-health-and-image-consistency)
* [Errors Encountered](#errors-encountered)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Key Commands Reference](#key-commands-reference)

---

## Project Overview

This project documents the end-to-end execution of a Kubernetes deployment rollback on a live K3s cluster. The task involved reverting the `nginx-deployment` from a recently promoted release (running `nginx:stable`) back to its previously validated revision (running `nginx:1.16`) using native Kubernetes rollout primitives via `kubectl`.

The lab simulates a real-world production incident response scenario in which a customer-facing bug is introduced by a new deployment, requiring an immediate, controlled rollback without downtime.

---

## Problem Statement

The Nautilus DevOps team promoted a new application release to the `nginx-deployment` in the `default` namespace. Shortly after the release, a customer reported a regression bug directly attributable to the new image version. The team required an immediate rollback to the previous stable revision to restore service integrity.

**Objective:** Roll back `nginx-deployment` to its prior revision using `kubectl rollout undo`, and validate that all pods are running the correct image version post-rollback.

---

## Architecture and Environment

| Component | Detail |
|---|---|
| Cluster Type | K3s (Lightweight Kubernetes) |
| Control Plane Endpoint | `https://127.0.0.1:6443` |
| kubectl Context | `default` |
| Target Namespace | `default` |
| Deployment Name | `nginx-deployment` |
| Replica Count | 3 |
| Update Strategy | RollingUpdate (25% max unavailable, 25% max surge) |
| Image Before Rollback | `nginx:stable` (Revision 2) |
| Image After Rollback | `nginx:1.16` (Revision 3, re-pinned to prior state) |
| Jump Host | `jump-host` |

---

## Prerequisites

* `kubectl` configured and authenticated against the target cluster
* Sufficient RBAC permissions to read and manage `Deployment` and `ReplicaSet` resources in the `default` namespace
* The target deployment (`nginx-deployment`) must exist and have at least two recorded revisions in its rollout history

---

## Implementation Guide

### Step 1: Verify Cluster Connectivity and Context

Before performing any operation, confirm that `kubectl` is communicating with the correct cluster and that the control plane is reachable.

```bash
kubectl cluster-info
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

Next, confirm the active kubeconfig context to rule out cross-cluster operation risk.

```bash
kubectl config current-context
```

**Output:**

```
default
```

> Screenshot:

<img width="1034" height="538" alt="image" src="https://github.com/user-attachments/assets/1bd060fc-3636-4fff-8ed5-1f89b1949202" />

> Terminal showing `kubectl cluster-info` and `kubectl config current-context` outputs confirming cluster connectivity and active context.

---

### Step 2: Confirm the Existing Deployment State

Verify that `nginx-deployment` exists in the `default` namespace and that all replicas are in a healthy state prior to the rollback.

```bash
kubectl get namespaces
```

**Output:**

```
NAME              STATUS   AGE
default           Active   3h7m
kube-node-lease   Active   3h7m
kube-public       Active   3h7m
kube-system       Active   3h7m
```

```bash
kubectl get deployment nginx-deployment
```

**Output:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           15m
```

All three replicas are `READY` and `AVAILABLE`, confirming the deployment is stable before intervention.

> Screenshot:

<img width="1036" height="330" alt="image" src="https://github.com/user-attachments/assets/16a499d1-c25b-44fc-b917-e3fdab07689b" />

> Terminal output showing `kubectl get namespaces` and `kubectl get deployment nginx-deployment` with all replicas in the ready state.

---

### Step 3: Inspect the Full Deployment Configuration

Retrieve the full deployment spec to understand the current image, update strategy, labels, replica set lineage, and event history.

```bash
kubectl describe deployment nginx-deployment
```

**Key fields from the output:**

| Field | Value |
|---|---|
| Image | `nginx:stable` |
| Replicas | `3 desired / 3 updated / 3 total / 3 available` |
| Strategy | `RollingUpdate` (25% max unavailable, 25% max surge) |
| Revision | `2` |
| Change Cause (Rev 2) | `kubectl set image deployment nginx-deployment nginx-container=nginx:stable --record=true` |
| Old ReplicaSet | `nginx-deployment-fc677cbc9` (0/0 replicas) |
| New ReplicaSet | `nginx-deployment-6c744d9dd6` (3/3 replicas) |

The event log confirmed a successful prior rolling update from `nginx:1.16` to `nginx:stable`:

```
ScalingReplicaSet: Scaled up nginx-deployment-6c744d9dd6 from 0 to 1
ScalingReplicaSet: Scaled down nginx-deployment-fc677cbc9 from 3 to 2
...
ScalingReplicaSet: Scaled down nginx-deployment-fc677cbc9 from 1 to 0
```

> Screenshots:

<img width="1028" height="828" alt="image" src="https://github.com/user-attachments/assets/81177fb1-599a-43a4-8154-1a1583ca3274" />
<img width="1033" height="860" alt="image" src="https://github.com/user-attachments/assets/d16075fa-3fa4-416e-8268-ad8854332926" />

> Terminal showing the full `kubectl describe deployment nginx-deployment` output, including image, strategy, ReplicaSet names, and event timeline.

---

### Step 4: Examine the Rollout History

List all recorded revisions for `nginx-deployment` to identify the target revision for rollback.

```bash
kubectl rollout history deployment/nginx-deployment
```

**Output:**

```
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment nginx-deployment nginx-container=nginx:stable --record=true
```

Two revisions exist:

* **Revision 1:** The initial deployment (no change cause recorded)
* **Revision 2:** The faulty release promoted using `--record=true`, running `nginx:stable`

The rollback target is **Revision 1**, which ran `nginx:1.16`.

> Screenshot:

<img width="1029" height="856" alt="image" src="https://github.com/user-attachments/assets/42f6997a-6339-47f1-a874-1671dd3310e3" />

> Terminal showing `kubectl rollout history deployment/nginx-deployment` with both revisions listed.

---

### Step 5: Inspect Individual Revisions

Before executing the rollback, inspect both revisions explicitly to confirm the exact image versions and pod template configuration associated with each.

**Inspect Revision 1:**

```bash
kubectl rollout history deployment/nginx-deployment --revision=1
```

**Output:**

```
Pod Template:
  Labels:  app=nginx-app
           pod-template-hash=fc677cbc9
  Containers:
   nginx-container:
    Image: nginx:1.16
```

**Inspect Revision 2:**

```bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

**Output:**

```
Pod Template:
  Annotations: kubernetes.io/change-cause: kubectl set image deployment nginx-deployment nginx-container=nginx:stable --record=true
  Containers:
   nginx-container:
    Image: nginx:stable
```

This confirms:

* Revision 1 corresponds to `nginx:1.16`, the previously validated stable image
* Revision 2 corresponds to `nginx:stable`, the current faulty release

> Screenshot:

<img width="1031" height="855" alt="image" src="https://github.com/user-attachments/assets/f48432c3-20b6-4725-8ae1-ce14ef7144b8" />

> Terminal showing `--revision=1` and `--revision=2` inspection outputs side by side, clearly contrasting `nginx:1.16` versus `nginx:stable`.

---

### Step 6: Execute the Rollback

Initiate the rollback to the immediately preceding revision (Revision 1) using `kubectl rollout undo`. Without a `--to-revision` flag, Kubernetes defaults to rolling back to the previous revision.

```bash
kubectl rollout undo deployment/nginx-deployment
```

**Output:**

```
deployment.apps/nginx-deployment rolled back
```

Kubernetes immediately begins terminating pods running `nginx:stable` and replacing them with pods running `nginx:1.16`, using the same RollingUpdate strategy (25% max unavailable, 25% max surge) to maintain availability throughout the transition.

> Screenshot:

<img width="1036" height="428" alt="image" src="https://github.com/user-attachments/assets/b3fbd6c3-6a93-4002-92b3-eeb4ec2d6f75" />

> Terminal showing the `kubectl rollout undo deployment/nginx-deployment` command and the `rolled back` confirmation message.

---

### Step 7: Confirm Rollout Completion

Monitor the rollout in real time until all replicas are updated and available.

```bash
kubectl rollout status deployment/nginx-deployment
```

**Output:**

```
deployment "nginx-deployment" successfully rolled out
```

This confirms the rollback completed without errors and all desired replicas are healthy.

> Screenshot Placeholder: Terminal showing `kubectl rollout status` output with the `successfully rolled out` confirmation.

---

### Step 8: Verify the Updated Rollout History

After the rollback, inspect the rollout history to confirm the revision sequence.

```bash
kubectl rollout history deployment/nginx-deployment
```

**Output:**

```
REVISION  CHANGE-CAUSE
2         kubectl set image deployment nginx-deployment nginx-container=nginx:stable --record=true
3         <none>
```

**Important behavioral note:** Kubernetes does not restore Revision 1 in place. Instead, it creates a new Revision 3, which is functionally identical to the old Revision 1 (running `nginx:1.16`). Revision 1 is consumed and removed from the history. The faulty Revision 2 remains in history for audit purposes.

> Screenshot Placeholder: Terminal showing updated rollout history with Revision 2 and the new Revision 3 listed.

---

### Step 9: Confirm the Active Image Version

Use `kubectl describe` with a `grep` filter to quickly extract the image currently active in the deployment template.

```bash
kubectl describe deployment nginx-deployment | grep Image
```

**Output:**

```
Image: nginx:1.16
```

The deployment is now running `nginx:1.16`, confirming the rollback was applied correctly to the deployment spec.

> Screenshot Placeholder: Terminal showing the `grep Image` output confirming `nginx:1.16` as the active container image.

---

### Step 10: Validate Pod Health and Image Consistency

List all pods associated with the deployment to confirm they are running, healthy, and pinned to the correct image.

**List pods:**

```bash
kubectl get pods -l app=nginx-app
```

**Output:**

```
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-fc677cbc9-d5d45   1/1     Running   0          3m13s
nginx-deployment-fc677cbc9-svrms   1/1     Running   0          3m12s
nginx-deployment-fc677cbc9-tm6b9   1/1     Running   0          3m14s
```

All three pods are in `Running` status with `0` restarts and are hashed under `fc677cbc9`, which matches the original Revision 1 ReplicaSet.

**Verify image version across all pods using JSONPath:**

```bash
kubectl get pods -l app=nginx-app -o jsonpath="{.items[*].spec.containers[*].image}"
```

**Output:**

```
nginx:1.16 nginx:1.16 nginx:1.16
```

All three pods are uniformly running `nginx:1.16`. The rollback is complete and fully validated.

> Screenshot Placeholder: Terminal showing `kubectl get pods` with all three pods in Running status, followed by the JSONPath output confirming `nginx:1.16` across all pod instances.

---

## Errors Encountered

No errors were encountered during this implementation. The rollback executed cleanly via `kubectl rollout undo` and completed without manual intervention.

**Note on `--record=true` deprecation:** The `kubernetes.io/change-cause` annotation on Revision 2 was produced by the `--record=true` flag used during the prior image update. This flag has been deprecated in recent Kubernetes versions. The recommended modern approach is to annotate the deployment directly after applying changes:

```bash
kubectl annotate deployment nginx-deployment kubernetes.io/change-cause="<description of change>"
```

This was not a blocking error in this lab but is a production-relevant consideration when interpreting rollout history.

---

## Best Practices Applied

* **Pre-rollback state validation:** The deployment's replica health and current image were confirmed before executing the rollback, ensuring a known baseline and avoiding compounding an already degraded state.

* **Explicit revision inspection before action:** Both `--revision=1` and `--revision=2` were inspected to confirm image identities before issuing `kubectl rollout undo`, eliminating ambiguity about what would be restored.

* **Rollout status monitoring:** `kubectl rollout status` was used immediately after the rollback to block on completion and confirm a clean transition rather than relying on eventual consistency.

* **JSONPath-based image verification:** A JSONPath query across all pod specs was used to confirm image uniformity across all replicas, which is more reliable than checking a single pod or the deployment object alone.

* **Label-based pod filtering:** Pods were queried using the `app=nginx-app` label selector rather than listing all pods, providing precise scoping aligned with the deployment's own selector definition.

* **No downtime rollback:** The rollback leveraged the existing `RollingUpdate` strategy (25% max unavailable, 25% max surge), ensuring that at least 2 of 3 replicas remained available throughout the transition.

---

## Lessons Learned

* **Kubernetes rollback creates a new revision, not a restore:** When `kubectl rollout undo` is executed, Kubernetes does not reinstate the old revision in its original position in history. It creates a new revision (Revision 3 in this case) that mirrors the pod template of the prior revision. The consumed revision (Revision 1) is removed from the visible history. This is critical to understand when reasoning about auditability and when planning to roll back to a specific older revision using `--to-revision`.

* **Revision 2 remains in history for audit:** Even after rolling back away from the faulty release, Revision 2 with its change-cause annotation (`nginx:stable`) persists in the rollout history. This supports post-incident review and root cause analysis without requiring external tooling.

* **The `--record` flag is deprecated but still functional:** In this environment, `--record=true` produced the change-cause annotation on Revision 2. Teams maintaining clusters on Kubernetes 1.25 and above should migrate to `kubectl annotate` for change-cause documentation to avoid relying on a deprecated flag that may be removed in future releases.

* **ReplicaSet hash confirms rollback identity:** After the rollback, the pod name prefix `nginx-deployment-fc677cbc9-*` matched the hash of the original Revision 1 ReplicaSet. Kubernetes reuses the existing ReplicaSet from the prior revision rather than creating a new one, which is an efficient mechanism that accelerates the rollback and preserves previously cached image layers on the nodes.

* **JSONPath is the most reliable image verification method:** Filtering `kubectl describe` output with `grep` is convenient for quick checks, but it reflects the deployment template spec, not the actual running pod state. Using `-o jsonpath="{.items[*].spec.containers[*].image}"` against the live pod list provides authoritative confirmation that every pod instance is running the intended image.

---

## Key Commands Reference

| Command | Purpose |
|---|---|
| `kubectl cluster-info` | Verify control plane reachability |
| `kubectl config current-context` | Confirm active kubeconfig context |
| `kubectl get deployment nginx-deployment` | Check deployment replica status |
| `kubectl describe deployment nginx-deployment` | Inspect full deployment spec and events |
| `kubectl rollout history deployment/nginx-deployment` | List all recorded revisions |
| `kubectl rollout history deployment/nginx-deployment --revision=N` | Inspect a specific revision's pod template |
| `kubectl rollout undo deployment/nginx-deployment` | Roll back to the previous revision |
| `kubectl rollout status deployment/nginx-deployment` | Monitor rollout completion in real time |
| `kubectl describe deployment nginx-deployment \| grep Image` | Quick-check the active image in the deployment template |
| `kubectl get pods -l app=nginx-app` | List all pods associated with the deployment |
| `kubectl get pods -l app=nginx-app -o jsonpath="{.items[*].spec.containers[*].image}"` | Verify image version uniformly across all running pods |









<img width="1035" height="465" alt="image" src="https://github.com/user-attachments/assets/8004757f-2f37-4b08-8e32-2df869b01c0a" />
<img width="1025" height="576" alt="image" src="https://github.com/user-attachments/assets/2d2dc728-201d-4209-9966-e6c817671027" />
<img width="1031" height="621" alt="image" src="https://github.com/user-attachments/assets/b88aec87-4e8e-411d-8070-1e40fb2588b8" />
<img width="1027" height="717" alt="image" src="https://github.com/user-attachments/assets/24c3a701-3b69-4e82-ae36-940878e80556" />
<img width="1038" height="748" alt="image" src="https://github.com/user-attachments/assets/a4b3c77e-1bb8-45b2-a3a4-1c7df8493d49" />
