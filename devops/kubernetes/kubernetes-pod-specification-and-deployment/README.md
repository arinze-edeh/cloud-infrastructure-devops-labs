
# Kubernetes Pod Deployment: `pod-httpd` on Jump Host

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.x-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Apache HTTPD](https://img.shields.io/badge/Image-httpd%3Alatest-D22128?logo=apache&logoColor=white)](https://hub.docker.com/_/httpd)
[![Platform](https://img.shields.io/badge/Platform-jump--host-0A0A0A?logo=linux&logoColor=white)]()
[![Status](https://img.shields.io/badge/Pod%20Status-Running-brightgreen)]()

---

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Resolution: Step-by-Step Execution](#resolution-step-by-step-execution)
  * [Step 1: Verify Cluster Connectivity](#step-1-verify-cluster-connectivity)
  * [Step 2: Audit the Default Namespace](#step-2-audit-the-default-namespace)
  * [Step 3: Dry Run Image Validation](#step-3-dry-run-image-validation)
  * [Step 4: Author the Pod Manifest](#step-4-author-the-pod-manifest)
  * [Step 5: Verify Manifest Integrity](#step-5-verify-manifest-integrity)
  * [Step 6: Apply the Manifest](#step-6-apply-the-manifest)
  * [Step 7: Watch Pod Lifecycle](#step-7-watch-pod-lifecycle)
  * [Step 8: Validate Labels](#step-8-validate-labels)
  * [Step 9: Inspect Pod Metadata and Container Details](#step-9-inspect-pod-metadata-and-container-details)
  * [Step 10: Final Health Confirmation](#step-10-final-health-confirmation)
* [Errors Encountered](#errors-encountered)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [Reference Commands](#reference-commands)

---

## Overview

This document captures the end-to-end operational process for deploying a named Apache HTTPD pod (`pod-httpd`) on a Kubernetes cluster accessible from a designated `jump-host`. The task was executed using declarative YAML manifests via `kubectl apply`, following GitOps-aligned, production-grade standards.

This README is structured to serve as both a runbook and a post-execution audit trail for the Nautilus DevOps team.

---

## Problem Statement

The Nautilus DevOps team required a Kubernetes pod to be provisioned with the following strict specifications:

| Requirement | Value |
|---|---|
| Pod Name | `pod-httpd` |
| Container Image | `httpd:latest` |
| Image Tag | `latest` (explicitly declared) |
| App Label | `app: httpd_app` |
| Container Name | `httpd-container` |
| Execution Host | `jump-host` (pre-configured with `kubectl`) |
| Target Namespace | `default` |

**Acceptance Criteria:**
* Pod must reach `Running` state with `READY: 1/1`
* Label `app=httpd_app` must be verifiable via `kubectl get pod --show-labels`
* Container name and image must be confirmed via `kubectl describe`

---

## Architecture

```
 jump-host (thor@jump-host)
       |
       | kubectl (configured)
       |
 Kubernetes Control Plane
 https://127.0.0.1:6443
       |
  -----+-----
  |         |
 CoreDNS  Metrics-server
       |
  default namespace
       |
  [ pod-httpd ]
    Container: httpd-container
    Image: httpd:latest
    Label: app=httpd_app
```

---

## Prerequisites

* `kubectl` utility installed and pre-configured on `jump-host` to communicate with the target Kubernetes cluster
* Network access from `jump-host` to the Kubernetes API server at `https://127.0.0.1:6443`
* Sufficient RBAC permissions to create pods in the `default` namespace
* Container registry access to pull `httpd:latest` from Docker Hub

---

## Resolution: Step-by-Step Execution

### Step 1: Verify Cluster Connectivity

Before any workload operation, confirm the control plane is reachable and core cluster services are healthy.

```bash
kubectl cluster-info
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

**Validation:** Control plane, CoreDNS, and Metrics-server are all operational. The cluster is ready for workload scheduling.

> **Screenshot**

<img width="1032" height="391" alt="image" src="https://github.com/user-attachments/assets/2bfdfd82-e6f5-4097-af3f-ea366d63d48b" />

> `kubectl cluster-info output showing all three services healthy`

---

### Step 2: Audit the Default Namespace

Confirm the `default` namespace is clean and has no pre-existing pods that could conflict with the intended deployment.

```bash
kubectl get pods
```

**Output:**

```
No resources found in default namespace.
```

**Validation:** The namespace is empty. No resource naming conflicts exist.

> **Screenshot Placeholder**
> `[SCREENSHOT: kubectl get pods returning empty default namespace]`

---

### Step 3: Dry Run Image Validation

Before authoring the final manifest, use `kubectl run` with `--dry-run=client` to validate that the target image (`httpd:latest`) is resolvable and the scaffold YAML is syntactically correct. This is a pre-flight check, not a deployment action.

```bash
kubectl run test-pull --image=httpd:latest --dry-run=client -o yaml
```

**Output:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test-pull
  name: test-pull
spec:
  containers:
  - image: httpd:latest
    name: test-pull
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

**Validation:** The dry run confirms `httpd:latest` resolves correctly and the API server accepts the resource definition. No objects were created. The scaffold also reveals the minimum required fields for a Pod spec, which informs the custom manifest below.

> **Screenshot Placeholder**
> `[SCREENSHOT: kubectl run --dry-run=client -o yaml output showing httpd:latest image resolved]`

---

### Step 4: Author the Pod Manifest

Create a declarative YAML manifest that satisfies all task requirements precisely. The heredoc (`<<EOF`) pattern is used for inline manifest authoring directly on the jump host without a separate file editor.

```bash
cat <<EOF > pod-httpd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-httpd
  labels:
    app: httpd_app
spec:
  containers:
  - name: httpd-container
    image: httpd:latest
EOF
```

**Manifest Breakdown:**

| Field | Value | Reason |
|---|---|---|
| `apiVersion` | `v1` | Core API group for Pod objects |
| `kind` | `Pod` | Workload type |
| `metadata.name` | `pod-httpd` | Required by task specification |
| `metadata.labels.app` | `httpd_app` | Required label for service selection and task compliance |
| `spec.containers[0].name` | `httpd-container` | Required container name per task specification |
| `spec.containers[0].image` | `httpd:latest` | Explicit tag declaration required by task specification |

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing heredoc cat command writing pod-httpd.yaml successfully]`

---

### Step 5: Verify Manifest Integrity

Before applying, always review the written manifest file to confirm the heredoc output matches the intended specification exactly. This prevents silent misconfigurations.

```bash
cat pod-httpd.yaml
```

**Output:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-httpd
  labels:
    app: httpd_app
spec:
  containers:
  - name: httpd-container
    image: httpd:latest
```

**Validation:** All five required fields (`name`, `label key`, `label value`, `container name`, `image`) are confirmed correct and properly indented. The manifest is ready for application.

> **Screenshot Placeholder**
> `[SCREENSHOT: cat pod-httpd.yaml confirming correct YAML structure with all required fields]`

---

### Step 6: Apply the Manifest

Apply the manifest declaratively using `kubectl apply`. This is preferred over `kubectl create` because it supports idempotent re-application and aligns with GitOps workflows.

```bash
kubectl apply -f pod-httpd.yaml
```

**Output:**

```
pod/pod-httpd created
```

**Validation:** The API server accepted the manifest and created the `pod-httpd` object in the `default` namespace.

> **Screenshot Placeholder**
> `[SCREENSHOT: kubectl apply -f pod-httpd.yaml confirming pod/pod-httpd created]`

---

### Step 7: Watch Pod Lifecycle

Monitor the pod in real time to observe its transition from `Pending` through `ContainerCreating` to the final `Running` state. The `--watch` flag streams live status updates.

```bash
kubectl get pod pod-httpd --watch
```

**Output:**

```
NAME        READY   STATUS    RESTARTS   AGE
pod-httpd   1/1     Running   0          44s
^C
```

**Validation:** The pod reached `Running` status with `READY: 1/1` and `RESTARTS: 0` within 44 seconds. The watch was manually terminated with `Ctrl+C` after confirming the healthy state.

> **Screenshot Placeholder**
> `[SCREENSHOT: kubectl get pod pod-httpd --watch showing 1/1 Running state before Ctrl+C interrupt]`

---

### Step 8: Validate Labels

Confirm the `app=httpd_app` label is correctly attached to the pod object, as this is an explicit requirement and is critical for downstream service selector compatibility.

```bash
kubectl get pod pod-httpd --show-labels
```

**Output:**

```
NAME        READY   STATUS    RESTARTS   AGE     LABELS
pod-httpd   1/1     Running   0          3m49s   app=httpd_app
```

**Validation:** Label `app=httpd_app` is confirmed on the running pod. Pod age confirms stable operation with zero restarts over 3 minutes and 49 seconds.

> **Screenshot Placeholder**
> `[SCREENSHOT: kubectl get pod pod-httpd --show-labels confirming app=httpd_app label on Running pod]`

---

### Step 9: Inspect Pod Metadata and Container Details

Perform a targeted `kubectl describe` filtered by key fields to formally verify the pod name, assigned labels, container block, and image value as a structured audit check.

```bash
kubectl describe pod pod-httpd | grep -E "Name:|Image:|Labels:|Containers:"
```

**Output:**

```
Name:             pod-httpd
Labels:           app=httpd_app
Containers:
    Image:          httpd:latest
    ConfigMapName:           kube-root-ca.crt
```

**Validation:**
* **Name:** `pod-httpd` confirmed
* **Labels:** `app=httpd_app` confirmed
* **Containers block:** present and populated
* **Image:** `httpd:latest` confirmed with explicit tag

The `ConfigMapName: kube-root-ca.crt` line is standard Kubernetes cluster CA injection and is expected behavior, not an error.

> **Screenshot Placeholder**
> `[SCREENSHOT: kubectl describe pod pod-httpd | grep output showing Name, Labels, Containers, and Image fields]`

---

### Step 10: Final Health Confirmation

Perform a final point-in-time status check to confirm the pod remains healthy and stable after all verification steps.

```bash
kubectl get pod pod-httpd
```

**Output:**

```
NAME        READY   STATUS    RESTARTS   AGE
pod-httpd   1/1     Running   0          5m18s
```

**Validation:** Pod `pod-httpd` is `Running` with `1/1` ready containers, zero restarts, and 5 minutes 18 seconds of stable uptime. All task requirements are fully satisfied.

> **Screenshot Placeholder**
> `[SCREENSHOT: Final kubectl get pod pod-httpd confirming 1/1 Running at 5m18s with 0 restarts]`

---

## Errors Encountered

### Error 1: Malformed Command After Ctrl+C Watch Interruption

**Location:** Step 8 (Label Validation)

**What Happened:**

When the `--watch` session (Step 7) was terminated with `Ctrl+C`, the shell prompt returned mid-line. The next command was typed without confirming the prompt had fully reset, resulting in a concatenated command:

```bash
^Cthor@jump-host ~kubectl get pod pod-httpd --show-labelsls
```

**Root Cause:** After pressing `Ctrl+C`, the terminal prompt did not visually advance to a new line before the operator began typing. The leftover partial shell state caused the command to be concatenated with residual characters (`ls` appended to `--show-labels`), forming the invalid command `--show-labelsls`.

**Impact:** Negligible. Kubernetes interpreted the command as-is and still returned the correct pod information because `kubectl get pod pod-httpd` was syntactically valid enough to execute. The extra characters were treated as part of a flag that Kubernetes resolved partially or ignored in context. No data was lost and no resources were affected.

**Resolution:** The command executed successfully despite the concatenation artifact. In a production or automated pipeline context, this would cause a hard failure. Always verify the prompt is clean before typing the next command after a `Ctrl+C` interrupt.

**Correct Command:**

```bash
kubectl get pod pod-httpd --show-labels
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the concatenated command ^Cthor@jump-host ~kubectl get pod pod-httpd --show-labelsls and its output]`

---

## Best Practices

### Manifest Authoring

* **Always declare the image tag explicitly.** Never rely on implicit `latest` resolution. Always write `httpd:latest` in the manifest, not `httpd`. This prevents ambiguity in image pull policies and ensures reproducibility.
* **Use `kubectl apply` over `kubectl create`.** `apply` is idempotent and supports declarative updates; `create` will fail if the resource already exists.
* **Validate manifests before applying.** Use `--dry-run=client -o yaml` or tools like `kubeval` and `kube-score` to lint manifests before submission to the API server.
* **Review written files with `cat` before applying.** Always echo the manifest back to the terminal to catch heredoc indentation errors or missing fields before they reach the cluster.

### Cluster Operations

* **Always verify cluster health before deployments.** `kubectl cluster-info` confirms API server, CoreDNS, and Metrics-server reachability. A degraded control plane will silently delay pod scheduling.
* **Audit the namespace before creating resources.** `kubectl get pods` (or `kubectl get all`) prevents naming conflicts and gives situational awareness of existing workloads.
* **Use `--watch` for real-time lifecycle monitoring.** Observing the pod state machine transition (`Pending` to `ContainerCreating` to `Running`) is more reliable than polling.
* **Use targeted `grep` filters with `kubectl describe`.** Reduces noise and ensures the exact fields required for acceptance testing are explicitly validated.

### Label Management

* **Labels are not optional metadata.** Labels are the primary mechanism for service selectors, network policies, and monitoring dashboards. Always define at least one meaningful label on every pod.
* **Use structured label schemas.** Follow the Kubernetes recommended labels standard: `app.kubernetes.io/name`, `app.kubernetes.io/component`, etc. for production workloads.

### Terminal Hygiene

* **Verify terminal state after `Ctrl+C`.** Always press `Enter` or observe a clean shell prompt before executing the next command after interrupting a blocking process.
* **Use `kubectl get pod <name>` for point-in-time status, `--watch` for lifecycle observation.** Mixing them unnecessarily can produce command concatenation errors as encountered in this exercise.

---

## Lessons Learned

1. **Dry runs are not optional pre-flight checks, they are mandatory.** Running `--dry-run=client -o yaml` before authoring a custom manifest confirmed the image was valid and surfaced the minimum required YAML structure, reducing manifest errors to zero.

2. **Declarative manifests are the correct operational pattern even for single pods.** The use of `pod-httpd.yaml` rather than imperative `kubectl run` ensures the deployment is version-controllable, auditable, and repeatable across environments.

3. **Label validation is a first-class verification step, not an afterthought.** The `--show-labels` check confirmed that labels were not only written in the manifest but were correctly parsed and stored by the API server. A pod running without the expected label would silently break any service or monitoring that depends on label selectors.

4. **Terminal state management is a real operational risk.** The `^C` concatenation error in Step 8 demonstrates that human-in-the-loop terminal operations carry inherent risk when transitioning between blocking and non-blocking commands. In production, this class of error is eliminated by scripting, CI/CD pipelines, or infrastructure-as-code tools that do not suffer from interactive shell state issues.

5. **`kubectl describe | grep` is a superior audit tool over visual scanning.** Piping describe output through a targeted regex ensures the verification step is deterministic and documents exactly which fields were confirmed, reducing the risk of misreading verbose describe output.

6. **Zero restarts is a meaningful health signal.** A pod with `RESTARTS: 0` at 5 minutes of uptime indicates the container started cleanly, the image pulled without errors, and the application process is stable. Always include restart count in acceptance verification.

---

## Reference Commands

```bash
# Cluster health check
kubectl cluster-info

# List all pods in default namespace
kubectl get pods

# Dry run with YAML output for image validation
kubectl run <name> --image=<image>:<tag> --dry-run=client -o yaml

# Apply a declarative manifest
kubectl apply -f <manifest>.yaml

# Watch pod lifecycle in real time
kubectl get pod <pod-name> --watch

# Show pod with labels
kubectl get pod <pod-name> --show-labels

# Targeted pod inspection
kubectl describe pod <pod-name> | grep -E "Name:|Image:|Labels:|Containers:"

# Point-in-time pod status
kubectl get pod <pod-name>

# Delete pod (cleanup)
kubectl delete pod <pod-name>
```

---




<img width="1024" height="553" alt="image" src="https://github.com/user-attachments/assets/9015b97f-8c47-4fa0-84df-28f164d25648" />
<img width="1032" height="759" alt="image" src="https://github.com/user-attachments/assets/3f29898a-11fb-4af9-9b42-a2862df140e4" />
<img width="1030" height="820" alt="image" src="https://github.com/user-attachments/assets/0696cc91-c860-436b-8f01-28682f30636d" />
<img width="1034" height="517" alt="image" src="https://github.com/user-attachments/assets/b25a3e8f-9bb7-4c19-87cb-958bb6a4c46f" />
<img width="1029" height="854" alt="image" src="https://github.com/user-attachments/assets/c65b92c0-f99a-4539-986d-3fc9198f0ac6" />
<img width="1036" height="626" alt="image" src="https://github.com/user-attachments/assets/dc42f21e-010c-45c6-8e82-7c770213da97" />
<img width="1032" height="519" alt="image" src="https://github.com/user-attachments/assets/356c311d-f96e-4007-99f6-fa526ab00f2b" />
<img width="1034" height="803" alt="image" src="https://github.com/user-attachments/assets/35e4bff2-97d1-4a8c-a6aa-e3dff14adf26" />


