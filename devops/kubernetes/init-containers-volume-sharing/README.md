# Kubernetes Init Containers: Shared Volume Pre-Initialization Pattern

> **Platform:** Kubernetes (k3s v1.34.1) | **Namespace:** `default`

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: Verify Cluster Connectivity and Context](#step-1-verify-cluster-connectivity-and-context)
  - [Step 2: Inspect Available Namespaces](#step-2-inspect-available-namespaces)
  - [Step 3: Author the Deployment Manifest](#step-3-author-the-deployment-manifest)
  - [Step 4: Validate the Manifest with Dry Run](#step-4-validate-the-manifest-with-dry-run)
  - [Step 5: Apply the Deployment](#step-5-apply-the-deployment)
  - [Step 6: Confirm Rollout Success](#step-6-confirm-rollout-success)
  - [Step 7: Verify Deployment Availability](#step-7-verify-deployment-availability)
  - [Step 8: Retrieve the Running Pod Name](#step-8-retrieve-the-running-pod-name)
  - [Step 9: Inspect Init Container Logs](#step-9-inspect-init-container-logs)
  - [Step 10: Inspect Main Container Logs](#step-10-inspect-main-container-logs)
  - [Step 11: Describe Pod and Verify Init Container State](#step-11-describe-pod-and-verify-init-container-state)
  - [Step 12: Programmatic Spec Verification via JSONPath](#step-12-programmatic-spec-verification-via-jsonpath)
- [Manifest Reference](#manifest-reference)
- [Verification Summary](#verification-summary)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Errors and Resolutions](#errors-and-resolutions)
- [Repository Structure](#repository-structure)

---

## Overview

This task demonstrates the **init container pattern** in Kubernetes, where a dedicated initialization container runs to completion before the main application container starts. A shared `emptyDir` volume serves as the inter-container communication channel, allowing the init container to stage data that the main container consumes at runtime.

This pattern is a foundational building block for production workloads that require bootstrapping configuration, pre-populating shared state, or performing readiness checks before the primary process begins.

---

## Problem Statement

The xFusionCorp Industries DevOps team identified a class of applications that require pre-runtime configuration changes that cannot be baked into container images. These changes must occur dynamically at pod startup, within the Kubernetes scheduling lifecycle, using only standard Kubernetes primitives.

**Requirements defined by the Nautilus DevOps team:**

| Parameter | Value |
|---|---|
| Deployment Name | `ic-deploy-devops` |
| Deployment Label (`app`) | `ic-devops` |
| Replicas | `1` |
| Init Container Name | `ic-msg-devops` |
| Init Container Image | `ubuntu:latest` |
| Init Container Command | Write `Init Done - Welcome to xFusionCorp Industries` to `/ic/beta` |
| Main Container Name | `ic-main-devops` |
| Main Container Image | `ubuntu:latest` |
| Main Container Command | Continuously `cat /ic/beta` every 5 seconds |
| Shared Volume Name | `ic-volume-devops` |
| Volume Type | `emptyDir` |
| Volume Mount Path | `/ic` (both containers) |

---

## Solution Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                          Pod Lifecycle                       │
│                                                             │
│  ┌──────────────────────┐        ┌───────────────────────┐  │
│  │   Init Container     │        │    Main Container     │  │
│  │   ic-msg-devops      │──────> │    ic-main-devops     │  │
│  │                      │        │                       │  │
│  │  ubuntu:latest       │        │  ubuntu:latest        │  │
│  │                      │        │                       │  │
│  │  Writes:             │        │  Reads:               │  │
│  │  "Init Done -        │        │  cat /ic/beta         │  │
│  │   Welcome to         │        │  (every 5 seconds)    │  │
│  │   xFusionCorp"       │        │                       │  │
│  │   to /ic/beta        │        │                       │  │
│  │                      │        │                       │  │
│  │  State: Completed    │        │  State: Running       │  │
│  └──────────┬───────────┘        └──────────┬────────────┘  │
│             │                               │               │
│             └──────────┬────────────────────┘               │
│                        │                                     │
│             ┌──────────▼────────────┐                        │
│             │  emptyDir Volume      │                        │
│             │  ic-volume-devops     │                        │
│             │  Mounted at: /ic      │                        │
│             └───────────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

**Data flow:** The init container runs first, writes the initialization message to `/ic/beta` on the shared `emptyDir` volume, and exits with code `0`. Only after this successful completion does Kubernetes start the main container, which continuously reads and prints the file content to stdout.

---

## Prerequisites

* `kubectl` configured and pointed at the target cluster
* Access to the `jump-host` node (acts as both jump host and control-plane in this single-node k3s setup)
* Network access to `docker.io` for pulling `ubuntu:latest`

---

## Implementation

### Step 1: Verify Cluster Connectivity and Context

Confirm the Kubernetes API server is reachable and identify which context is active before making any changes.

```bash
kubectl cluster-info
kubectl config current-context
kubectl get nodes
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

default

NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   57m   v1.34.1+k3s1
```

> **Screenshot:** 

<img width="1036" height="633" alt="image" src="https://github.com/user-attachments/assets/13f54b63-769a-469e-a3f0-9f18ef1f53d4" />

The cluster is healthy, the active context is `default`, and the single-node control-plane (`jump-host`) is in `Ready` state running k3s `v1.34.1`.

---

### Step 2: Inspect Available Namespaces

Verify which namespaces exist to confirm the deployment target before applying resources.

```bash
kubectl get namespaces
```

**Output:**

```
NAME              STATUS   AGE
default           Active   58m
kube-node-lease   Active   58m
kube-public       Active   58m
kube-system       Active   58m
```

> **Screenshot:** 

<img width="1036" height="633" alt="image" src="https://github.com/user-attachments/assets/13f54b63-769a-469e-a3f0-9f18ef1f53d4" />

The `default` namespace is active. No custom namespaces are required for this task; the deployment targets `default` implicitly.

---

### Step 3: Author the Deployment Manifest

Compose the full Deployment manifest inline using a heredoc. This captures the complete desired state in a single atomic operation and leaves a file artifact on disk for audit and repeatability.

```bash
cat <<'EOF' > ic-deploy-devops.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ic-deploy-devops
  labels:
    app: ic-devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ic-devops
  template:
    metadata:
      labels:
        app: ic-devops
    spec:
      initContainers:
      - name: ic-msg-devops
        image: ubuntu:latest
        command: ['/bin/bash', '-c', 'echo Init Done - Welcome to xFusionCorp Industries > /ic/beta']
        volumeMounts:
        - name: ic-volume-devops
          mountPath: /ic
      containers:
      - name: ic-main-devops
        image: ubuntu:latest
        command: ['/bin/bash', '-c', 'while true; do cat /ic/beta; sleep 5; done']
        volumeMounts:
        - name: ic-volume-devops
          mountPath: /ic
      volumes:
      - name: ic-volume-devops
        emptyDir: {}
EOF
```

Verify the file was written correctly by printing it back to stdout:

```bash
cat ic-deploy-devops.yaml
```

> **Screenshots:** 


<img width="1031" height="855" alt="image" src="https://github.com/user-attachments/assets/229c1b1d-8b56-4b1e-a296-4afbdbe7ef18" />
<img width="1028" height="862" alt="image" src="https://github.com/user-attachments/assets/d1890b12-de85-4c64-b4e1-1bd8fc482c41" />

**Key manifest decisions:**

* `emptyDir: {}` allocates ephemeral node-local storage scoped to the pod lifetime. Data written by the init container is immediately available to the main container because both mount the same volume object.
* The init container command uses shell redirection (`>`) to write the exact message to `/ic/beta`.
* The main container command runs an infinite loop, reading `/ic/beta` every 5 seconds, demonstrating continuous consumption of init-staged data.

---

### Step 4: Validate the Manifest with Dry Run

Before submitting to the API server, perform a client-side dry run to catch schema validation errors without creating any cluster resources.

```bash
kubectl apply -f ic-deploy-devops.yaml --dry-run=client
```

**Output:**

```
deployment.apps/ic-deploy-devops created (dry run)
```

> **Screenshot:** 

<img width="1029" height="723" alt="image" src="https://github.com/user-attachments/assets/0a69ab64-0593-4b25-8feb-7a72484d6708" />

The `(dry run)` suffix confirms the manifest passed client-side schema validation. No resources were created at this stage.

---

### Step 5: Apply the Deployment

Submit the manifest to the Kubernetes API server to create the Deployment resource.

```bash
kubectl apply -f ic-deploy-devops.yaml
```

**Output:**

```
deployment.apps/ic-deploy-devops created
```

> **Screenshot:** 

<img width="1031" height="761" alt="image" src="https://github.com/user-attachments/assets/66d375c4-3e1e-4bb6-a697-bb55c93afcf8" />

The `created` response confirms the API server accepted and persisted the Deployment object. The ReplicaSet controller and scheduler will now act on this desired state.

---

### Step 6: Confirm Rollout Success

Block and watch the rollout until all replicas reach the `Available` condition or a timeout occurs.

```bash
kubectl rollout status deployment/ic-deploy-devops
```

**Output:**

```
deployment "ic-deploy-devops" successfully rolled out
```

> **Screenshot:** 

<img width="1032" height="863" alt="image" src="https://github.com/user-attachments/assets/aea77b00-f105-41c2-b1f1-d7b2644a3ee7" />

The deployment controller confirmed all replicas successfully transitioned to the `Available` state. This command is blocking and exits with code `0` on success, making it suitable for CI/CD pipeline gates.

---

### Step 7: Verify Deployment Availability

Confirm the Deployment reports the correct replica counts across READY, UP-TO-DATE, and AVAILABLE columns.

```bash
kubectl get deployment ic-deploy-devops
```

**Output:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
ic-deploy-devops   1/1     1            1           2m31s
```

> **Screenshot:** 

<img width="1032" height="863" alt="image" src="https://github.com/user-attachments/assets/aea77b00-f105-41c2-b1f1-d7b2644a3ee7" />

All three replica count columns report `1`, confirming the single replica is healthy and serving.

---

### Step 8: Retrieve the Running Pod Name

Use a label selector with JSONPath to dynamically capture the pod name into a shell variable, avoiding hardcoded names that would break across restarts or re-deploys.

```bash
POD_NAME=$(kubectl get pods -l app=ic-devops -o jsonpath='{.items[0].metadata.name}')
echo $POD_NAME
```

**Output:**

```
ic-deploy-devops-7cffc48877-qjtfj
```

> **Screenshot:** 

<img width="1030" height="602" alt="image" src="https://github.com/user-attachments/assets/fbc0cffc-0327-49b4-95a9-020c41a94165" />

The pod name follows the standard Kubernetes naming convention: `<deployment-name>-<replicaset-hash>-<pod-hash>`. Storing it in `$POD_NAME` enables all subsequent log and describe commands to remain dynamic.

---

### Step 9: Inspect Init Container Logs

Query logs specifically from the init container. Since the init container ran a simple `echo` redirect (no stdout output) and exited, the log stream is expected to be empty.

```bash
kubectl logs $POD_NAME -c ic-msg-devops
```

**Output:**

```
(empty)
```

> **Screenshot:** 

<img width="1024" height="335" alt="image" src="https://github.com/user-attachments/assets/f6f47aec-c082-4411-9199-029120bfd0d1" />

The empty log output is correct and expected. The init container used shell redirection (`> /ic/beta`) to write to the file, not `echo` to stdout. There is nothing for the Kubernetes log driver to capture.

---

### Step 10: Inspect Main Container Logs

Query logs from the main container to confirm it is successfully reading the file staged by the init container.

```bash
kubectl logs $POD_NAME -c ic-main-devops
```

**Output (excerpt):**

```
Init Done - Welcome to xFusionCorp Industries
Init Done - Welcome to xFusionCorp Industries
Init Done - Welcome to xFusionCorp Industries
...
```

> **Screenshots:** 

<img width="1036" height="864" alt="image" src="https://github.com/user-attachments/assets/84bc4138-d945-4225-90e8-ffdec7642d48" />
<img width="1051" height="867" alt="image" src="https://github.com/user-attachments/assets/ae29cd5a-1bec-45d7-964e-380f6b89e23e" />

The main container is printing `Init Done - Welcome to xFusionCorp Industries` on a 5-second interval, confirming:

1. The init container successfully wrote the message to `/ic/beta` before exiting.
2. The `emptyDir` volume correctly persisted data between the init and main container lifecycles within the same pod.
3. The main container's `while true; do cat /ic/beta; sleep 5; done` loop is functioning as specified.

---

### Step 11: Describe Pod and Verify Init Container State

Run `kubectl describe` and filter for the Init Containers section to confirm the init container completed cleanly with exit code `0`.

```bash
kubectl describe pod $POD_NAME | grep -A 15 "Init Containers:"
```

**Output:**

```
Init Containers:
  ic-msg-devops:
    Container ID:  containerd://655e5443eafb1f7c9882ec1ed4d4592a06b64cd02fa83a823e6ce3b394f6da02
    Image:         ubuntu:latest
    Image ID:      docker.io/library/ubuntu@sha256:84e77dee7d1bc93fb029a45e3c6cb9d8aa4831ccfcc7103d36e876938d28895b
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/bash
      -c
      echo Init Done - Welcome to xFusionCorp Industries > /ic/beta
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 08 Apr 2026 22:37:27 +0000
      Finished:     Wed, 08 Apr 2026 22:37:27 +0000
```

> **Screenshot:** 

<img width="1037" height="490" alt="image" src="https://github.com/user-attachments/assets/c7857523-f2ab-4f53-b54b-e9c314816020" />

Critical observations:

* **State:** `Terminated` with **Reason:** `Completed` confirms the init container ran to successful completion.
* **Exit Code:** `0` confirms no errors occurred during file write.
* The init container started and finished at the same second, reflecting the near-instant nature of the `echo` and redirect operation.
* The image is pinned to a specific digest (`sha256:84e77dee...`), confirming the exact image layer pulled from Docker Hub.

---

### Step 12: Programmatic Spec Verification via JSONPath

Run a series of targeted JSONPath queries against the live Deployment object to programmatically confirm every key spec field matches the original requirements. This pattern is suitable for automated compliance checks in CI pipelines.

```bash
# Deployment name
kubectl get deployment ic-deploy-devops \
  -o jsonpath='{.metadata.name}'

# Replica count
kubectl get deployment ic-deploy-devops \
  -o jsonpath='{.spec.replicas}'

# Deployment-level label
kubectl get deployment ic-deploy-devops \
  -o jsonpath='{.metadata.labels.app}'

# Pod template label
kubectl get deployment ic-deploy-devops \
  -o jsonpath='{.spec.template.metadata.labels.app}'

# Init container name
kubectl get deployment ic-deploy-devops \
  -o jsonpath='{.spec.template.spec.initContainers[0].name}'

# Main container name
kubectl get deployment ic-deploy-devops \
  -o jsonpath='{.spec.template.spec.containers[0].name}'

# Volume name
kubectl get deployment ic-deploy-devops \
  -o jsonpath='{.spec.template.spec.volumes[0].name}'

# Volume type (emptyDir)
kubectl get deployment ic-deploy-devops \
  -o jsonpath='{.spec.template.spec.volumes[0].emptyDir}'
```

**Outputs:**

| Query | Expected | Actual |
|---|---|---|
| `metadata.name` | `ic-deploy-devops` | `ic-deploy-devops` |
| `spec.replicas` | `1` | `1` |
| `metadata.labels.app` | `ic-devops` | `ic-devops` |
| `spec.template.metadata.labels.app` | `ic-devops` | `ic-devops` |
| `initContainers[0].name` | `ic-msg-devops` | `ic-msg-devops` |
| `containers[0].name` | `ic-main-devops` | `ic-main-devops` |
| `volumes[0].name` | `ic-volume-devops` | `ic-volume-devops` |
| `volumes[0].emptyDir` | `{}` | `{}` |

> **Screenshot:** 

<img width="1033" height="856" alt="image" src="https://github.com/user-attachments/assets/7112e2cb-5d3e-46a2-a5f2-c20f329fa568" />

All eight fields match the specification exactly. The empty `{}` for `emptyDir` confirms the volume type is set with default parameters (no size limit, no memory-backed mode), which is correct per the task requirements.

---

## Manifest Reference

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ic-deploy-devops
  labels:
    app: ic-devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ic-devops
  template:
    metadata:
      labels:
        app: ic-devops
    spec:
      initContainers:
      - name: ic-msg-devops
        image: ubuntu:latest
        command: ['/bin/bash', '-c', 'echo Init Done - Welcome to xFusionCorp Industries > /ic/beta']
        volumeMounts:
        - name: ic-volume-devops
          mountPath: /ic
      containers:
      - name: ic-main-devops
        image: ubuntu:latest
        command: ['/bin/bash', '-c', 'while true; do cat /ic/beta; sleep 5; done']
        volumeMounts:
        - name: ic-volume-devops
          mountPath: /ic
      volumes:
      - name: ic-volume-devops
        emptyDir: {}
```

---

## Verification Summary

| Check | Command | Expected Result | Status |
|---|---|---|---|
| Cluster reachable | `kubectl cluster-info` | Control plane URL returned | PASS |
| Active context | `kubectl config current-context` | `default` | PASS |
| Node ready | `kubectl get nodes` | `jump-host` Ready | PASS |
| Deployment created | `kubectl apply -f ic-deploy-devops.yaml` | `created` | PASS |
| Rollout complete | `kubectl rollout status` | `successfully rolled out` | PASS |
| Replicas available | `kubectl get deployment` | `1/1` | PASS |
| Init container exited cleanly | `kubectl describe pod` | `Exit Code: 0`, `Reason: Completed` | PASS |
| Main container reading file | `kubectl logs -c ic-main-devops` | Repeating message in stdout | PASS |
| All spec fields match | JSONPath queries | All 8 fields confirmed | PASS |

---

## Best Practices Applied

* **Dry run before apply:** `--dry-run=client` was used to validate the manifest schema before submitting to the API server, reducing the risk of malformed resources reaching the cluster state.

* **Declarative manifest management:** The Deployment was defined in a YAML file (`ic-deploy-devops.yaml`) rather than using imperative `kubectl run` commands, enabling version control, reproducibility, and team review.

* **Dynamic pod name resolution:** `kubectl get pods -l app=ic-devops -o jsonpath='{.items[0].metadata.name}'` was used instead of hardcoding the pod name, making the verification steps resilient to pod restarts and re-schedules.

* **Container-specific log targeting:** `kubectl logs $POD_NAME -c <container-name>` was used explicitly to disambiguate between the init container and main container log streams in a multi-container pod.

* **JSONPath programmatic verification:** Rather than only visual inspection, every critical spec field was queried individually via JSONPath against the live API object, producing machine-readable output suitable for automated compliance validation.

* **`rollout status` as a blocking gate:** Using `kubectl rollout status` ensures the verification pipeline only proceeds once the deployment has fully converged, rather than polling or sleeping for an arbitrary duration.

* **Selector label consistency:** The `selector.matchLabels.app` value, the `metadata.labels.app` on the Deployment, and the `spec.template.metadata.labels.app` on the pod template all use the same `ic-devops` value, maintaining correct label hierarchy and ReplicaSet ownership.

---

## Lessons Learned

**Init container stdout vs. file redirection:** The init container produced no output in `kubectl logs` because it used shell redirection (`> /ic/beta`) to write to a file rather than printing to stdout. This is a common point of confusion when debugging init containers. Always clarify whether the init container's work product is a file, a network call, or stdout before interpreting empty logs as a failure signal.

**emptyDir lifetime is pod-scoped:** The `emptyDir` volume lives and dies with the pod. If the pod is deleted or rescheduled, all data written to the volume is lost. For production use cases requiring durable init data, consider using `ConfigMap` volumes, `Secret` volumes, or a `PersistentVolumeClaim` instead.

**Init container ordering is strictly sequential:** Kubernetes guarantees that init containers run in the order they are listed in the spec, each running to completion before the next starts. The main containers only start after all init containers have exited with code `0`. This sequencing guarantee is what makes the pattern safe for bootstrapping shared state.

**`ubuntu:latest` is suitable for lab environments only:** Using `latest` tags in production is an anti-pattern because the resolved image can change without any manifest change, breaking reproducibility and potentially introducing breaking changes. Production workloads should pin images to a specific digest or immutable tag.

**`kubectl describe` is more diagnostic than `kubectl get`:** While `kubectl get` provides availability and readiness status, `kubectl describe` exposes the full event timeline, container states, exit codes, and resource conditions. It should always be the first tool used when investigating pod lifecycle issues.

---

## Errors and Resolutions

No errors were encountered during this implementation. The following edge cases were anticipated and avoided through deliberate practice:

| Potential Error | Root Cause | Prevention Applied |
|---|---|---|
| `ImagePullBackOff` | No internet access to `docker.io` on cluster nodes | Confirmed node network access before applying the manifest |
| Init container in `Error` state | Shell command syntax error in the `command` array | Tested the `echo ... > /ic/beta` redirection pattern locally before embedding in the manifest |
| Main container `CrashLoopBackOff` | File `/ic/beta` missing because volume mount path was incorrect | Verified both containers mount `ic-volume-devops` at `/ic` before applying |
| Empty main container logs despite running | Wrong container targeted with `kubectl logs` | Used explicit `-c ic-main-devops` flag to select the correct container |

---
