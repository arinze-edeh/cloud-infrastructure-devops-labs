# Kubernetes Pod Resource Management: Enforcing CPU and Memory Limits

> **Domain:** Container Orchestration | Kubernetes | Resource Governance
> **Difficulty:** Intermediate
> **Environment:** Kubernetes v1.34.1 | kubectl | containerd runtime
> **Status:** Resolved and Verified

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Environment and Prerequisites](#environment-and-prerequisites)
- [Architecture Overview](#architecture-overview)
- [Step-by-Step Resolution](#step-by-step-resolution)
- [Validation and Verification](#validation-and-verification)
- [Errors and Observations](#errors-and-observations)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Key Commands Reference](#key-commands-reference)

---

## Problem Statement

The Nautilus DevOps team identified performance degradation in Kubernetes-hosted applications caused by the absence of resource boundaries on running workloads. Unconstrained pods compete for node CPU and memory, leading to noisy-neighbour effects, OOMKill events, and degraded cluster stability.

**Objective:** Provision a pod named `httpd-pod` running the `httpd:latest` container image with explicitly defined CPU and memory requests and limits that enforce resource governance at the scheduler and kubelet level.

**Resource Specification:**

| Parameter | Value |
|-----------|-------|
| Pod Name | `httpd-pod` |
| Container Name | `httpd-container` |
| Image | `httpd:latest` |
| Memory Request | `15Mi` |
| Memory Limit | `20Mi` |
| CPU Request | `100m` |
| CPU Limit | `100m` |

---

## Environment and Prerequisites

### Cluster Information

```bash
thor@jump-host ~$ kubectl version --client
Client Version: v1.34.1
Kustomize Version: v5.7.1

thor@jump-host ~$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

thor@jump-host ~$ kubectl config current-context
default
```

### Verified Context

- kubectl client: `v1.34.1`
- Active context: `default`
- Control plane endpoint: `https://127.0.0.1:6443`
- Metrics server: Available (enables future HPA and resource tracking)

### Namespace State Before Provisioning

```bash
thor@jump-host ~$ kubectl get pods -n default
No resources found in default namespace.
```

> The default namespace was confirmed clean prior to pod creation, eliminating pre-existing resource conflicts.

---

## Architecture Overview

```
jump-host (kubectl client)
        |
        | HTTPS :6443
        v
Kubernetes Control Plane (127.0.0.1:6443)
        |
        | Scheduler assigns pod to node
        v
Node: jump-host / 10.244.13.56
        |
        | containerd runtime
        v
Pod: httpd-pod (10.22.0.9)
  └── Container: httpd-container
        Image:   httpd:latest
        Request: 15Mi RAM | 100m CPU
        Limit:   20Mi RAM | 100m CPU
        QoS:     Burstable
```

---

## Step-by-Step Resolution

### Step 1: Verify Cluster Connectivity and Context

Before any workload provisioning, validate that the kubectl client is properly configured and the cluster is reachable.

```bash
kubectl version --client
kubectl cluster-info
kubectl config current-context
```

**Expected Output:**

```
Client Version: v1.34.1
Kubernetes control plane is running at https://127.0.0.1:6443
default
```

> **Why this matters:** Deploying to the wrong context (e.g., production vs staging) is a leading cause of accidental workload misplacement. Always confirm context before applying any manifests.

---

> **SCREENSHOT**

<img width="1030" height="518" alt="image" src="https://github.com/user-attachments/assets/bb95e1e4-d5e4-4c41-87cb-fabc6e940f88" />

> *Terminal output showing kubectl version, cluster-info, and current context confirmation.*

---

### Step 2: Confirm Namespace is Clean

```bash
kubectl get pods -n default
```

**Expected Output:**

```
No resources found in default namespace.
```

> Verifying a clean namespace prevents naming collisions and ensures the pod state observed during validation is attributable solely to this deployment.

> **SCREENSHOT**

<img width="1030" height="522" alt="image" src="https://github.com/user-attachments/assets/8d28be08-81b6-48d3-b818-324426f65188" />


---

### Step 3: Author the Pod Manifest

Use a heredoc to write the pod specification directly to disk. This approach is reproducible, auditable, and avoids manual editor errors.

```bash
cat <<'EOF' > httpd-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-pod
spec:
  containers:
    - name: httpd-container
      image: httpd:latest
      resources:
        requests:
          memory: "15Mi"
          cpu: "100m"
        limits:
          memory: "20Mi"
          cpu: "100m"
EOF
```

**Manifest breakdown:**

| Field | Value | Purpose |
|-------|-------|---------|
| `apiVersion` | `v1` | Core API group for Pod objects |
| `kind` | `Pod` | Kubernetes resource type |
| `metadata.name` | `httpd-pod` | Unique identifier in the namespace |
| `spec.containers[0].name` | `httpd-container` | Container name within the pod |
| `spec.containers[0].image` | `httpd:latest` | Official Apache HTTPD image |
| `resources.requests.memory` | `15Mi` | Guaranteed memory allocation for scheduling |
| `resources.requests.cpu` | `100m` | 0.1 vCPU guaranteed at scheduling |
| `resources.limits.memory` | `20Mi` | Hard cap; exceeding triggers OOMKill |
| `resources.limits.cpu` | `100m` | Hard CPU throttle ceiling |

---

> **SCREENSHOT**

<img width="1028" height="861" alt="image" src="https://github.com/user-attachments/assets/ab118cd3-30cc-493f-8385-28f4c2142b7d" />

> *Terminal showing the cat command output of httpd-pod.yaml confirming the manifest content.*

---

### Step 4: Validate the Manifest (Client-Side Dry Run)

Perform a client-side dry run to detect syntax errors and schema violations without making any API server calls.

```bash
kubectl apply -f httpd-pod.yaml --dry-run=client
```

**Expected Output:**

```
pod/httpd-pod created (dry run)
```

> `--dry-run=client` validates the manifest against the local OpenAPI schema embedded in kubectl. No network call is made. This is the first gate in a safe deployment pipeline.

---

### Step 5: Validate Against the API Server (Server-Side Dry Run)

Perform a server-side dry run to validate the manifest against live admission controllers, namespace quotas, and policy enforcement without persisting any resources.

```bash
kubectl apply -f httpd-pod.yaml --dry-run=server
```

**Expected Output:**

```
pod/httpd-pod created (server dry run)
```

> `--dry-run=server` exercises the full admission webhook chain. It catches issues that client-side validation cannot, such as PodSecurityAdmission rejections, LimitRange violations, and quota exhaustion. Always run both dry-run modes before applying to production-adjacent environments.

---

> **SCREENSHOT**

<img width="1031" height="742" alt="image" src="https://github.com/user-attachments/assets/22120df8-b1a6-49fe-8de8-9d59118a4f29" />

> *Terminal showing both --dry-run=client and --dry-run=server passing successfully.*

---

### Step 6: Apply the Manifest to the Cluster

With both dry-run validations passing, apply the manifest to provision the pod.

```bash
kubectl apply -f httpd-pod.yaml
```

**Expected Output:**

```
pod/httpd-pod created
```

> `kubectl apply` is idempotent and preferred over `kubectl create` for GitOps-aligned workflows. Re-running `apply` on an unchanged manifest produces no side effects.

---

> **SCREENSHOT**

<img width="1034" height="744" alt="image" src="https://github.com/user-attachments/assets/cdd6cc1a-84f9-4137-bf5a-69e4a683df87" />

> *Caption: Terminal showing kubectl apply -f httpd-pod.yaml with the "pod/httpd-pod created" confirmation.*

---

### Step 7: Watch Pod Reach Running State

Monitor the pod lifecycle in real time until it reaches `Running` status.

```bash
kubectl get pod httpd-pod --watch
```

**Expected Output:**

```
NAME        READY   STATUS    RESTARTS   AGE
httpd-pod   1/1     Running   0          19s
```

Interrupt the watch with `Ctrl+C` once the pod is confirmed running.

> The pod transitioned from `Pending` to `Running` in approximately 19 seconds. The dominant factor was image pull latency (3.026 seconds per kubelet events), with the remainder attributable to container initialization.

---

> **SCREENSHOT**

<img width="1026" height="865" alt="image" src="https://github.com/user-attachments/assets/c2bdd40a-fcb7-44ef-a95c-54dca2c8f587" />

> *Terminal showing kubectl get pod --watch output with pod status transitioning to 1/1 Running.*

---

### Step 8: Confirm Pod Status After Stabilization

Verify the pod remains in a stable running state after the watch is dismissed.

```bash
kubectl get pod httpd-pod
```

**Expected Output:**

```
NAME        READY   STATUS    RESTARTS   AGE
httpd-pod   1/1     Running   0          4m15s
```

> `RESTARTS: 0` confirms no crashloops or OOMKill events occurred. The memory limit of 20Mi was sufficient for the Apache HTTPD process at idle.

> **SCREENSHOT**

<img width="1028" height="595" alt="image" src="https://github.com/user-attachments/assets/c154f4d6-81c2-4003-abb2-73c5f3dc5d75" />

---

## Validation and Verification

### Verify Container Name via JSONPath

```bash
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].name}'
```

**Output:**

```
httpd-container
```

### Verify Container Image via JSONPath

```bash
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].image}'
```

**Output:**

```
httpd:latest
```

### Verify Resource Requests via JSONPath

```bash
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].resources.requests}'
```

**Output:**

```json
{"cpu":"100m","memory":"15Mi"}
```

### Verify Resource Limits via JSONPath

```bash
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].resources.limits}'
```

**Output:**

```json
{"cpu":"100m","memory":"20Mi"}
```

---

> **SCREENSHOT PLACEHOLDER**
> `[screenshot-06-jsonpath-validation.png]`
> *Caption: Terminal showing all four JSONPath queries with their respective outputs confirming container name, image, requests, and limits.*

---

### Full Pod Inspection via kubectl describe

```bash
kubectl describe pod httpd-pod
```

**Key fields from describe output:**

```
Name:             httpd-pod
Namespace:        default
Node:             jump-host/10.244.13.56
Status:           Running
IP:               10.22.0.9

Containers:
  httpd-container:
    Image:          httpd:latest
    Image ID:       docker.io/library/httpd@sha256:331548c5249bde...
    State:          Running
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     100m
      memory:  20Mi
    Requests:
      cpu:        100m
      memory:     15Mi

Conditions:
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True

QoS Class: Burstable

Events:
  Normal  Scheduled  Successfully assigned default/httpd-pod to jump-host
  Normal  Pulling    Pulling image "httpd:latest"
  Normal  Pulled     Successfully pulled image "httpd:latest" in 3.026s
  Normal  Created    Created container: httpd-container
  Normal  Started    Started container httpd-container
```

---

> **SCREENSHOT PLACEHOLDER**
> `[screenshot-07-describe-pod.png]`
> *Caption: Full kubectl describe pod httpd-pod output showing all container specs, resource limits, conditions, and events.*

---

### QoS Class Explanation

The pod was assigned `QoS Class: Burstable` because:

- CPU and memory **requests** are set and non-zero
- CPU and memory **limits** are set but the limits **do not equal** the requests for memory (15Mi request vs 20Mi limit)

| QoS Class | Condition |
|-----------|-----------|
| `Guaranteed` | requests == limits for ALL resources |
| `Burstable` | At least one resource has requests set; requests != limits |
| `BestEffort` | No requests or limits defined |

> For stricter workloads, setting requests equal to limits (e.g., both memory at `20Mi`) promotes the pod to `Guaranteed` QoS, giving it the lowest eviction priority under node memory pressure.

---

## Errors and Observations

### Observation 1: Terminal Prompt Corruption After Ctrl+C

**What happened:** After interrupting `kubectl get pod httpd-pod --watch` with `Ctrl+C`, the terminal prompt partially merged with the next command typed:

```
^Cthor@jump-host ~kubectl get pod httpd-podod
```

**Root cause:** The `--watch` flag places the terminal in a streaming read loop. Pressing `Ctrl+C` sends SIGINT mid-render, which can corrupt the displayed prompt buffer. This is a cosmetic terminal rendering artefact only. The underlying kubectl process terminated cleanly.

**Impact:** None. The subsequent command executed correctly and returned valid output.

**Mitigation:** After interrupting a `--watch` command, press `Enter` once to force a clean prompt render before typing the next command.

---

### Observation 2: JSONPath Output Truncation in Terminal Log

**What happened:** The terminal transcript shows JSONPath queries with the command and prompt partially overlapping the output:

```
{"cpu":"100m","memkubectl get pod httpd-pod...
```

**Root cause:** This is a terminal multiplexer or shell history replay artefact in the captured transcript. The actual kubectl output was correct as evidenced by the verified resource values in `kubectl describe`.

**Impact:** None. Functional verification via `kubectl describe` confirmed all resource values were applied correctly.

---

## Best Practices

### 1. Always Define Both Requests and Limits

Never deploy pods without `resources.requests` and `resources.limits`. Pods without requests bypass the scheduler's bin-packing logic, and pods without limits become unbounded consumers that can destabilize entire nodes.

```yaml
resources:
  requests:
    memory: "15Mi"
    cpu: "100m"
  limits:
    memory: "20Mi"
    cpu: "100m"
```

### 2. Use a Two-Phase Dry Run Pipeline

Adopt a mandatory `--dry-run=client` followed by `--dry-run=server` before every `kubectl apply` in non-development environments. This catches schema errors locally and policy/quota violations server-side before any state change occurs.

```bash
kubectl apply -f manifest.yaml --dry-run=client
kubectl apply -f manifest.yaml --dry-run=server
kubectl apply -f manifest.yaml
```

### 3. Prefer kubectl apply Over kubectl create

`kubectl apply` is declarative and idempotent. `kubectl create` is imperative and will error if the resource already exists. For GitOps and CI/CD pipelines, `apply` is the correct verb.

### 4. Use JSONPath for Targeted Verification

Rather than grepping `kubectl describe` output, use structured JSONPath queries for precise, scriptable validation:

```bash
# Verify resource limits programmatically
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].resources.limits}'
```

This pattern integrates directly into CI validation scripts and smoke test pipelines.

### 5. Pin Image Tags in Production

`httpd:latest` is appropriate for lab and testing contexts. In production, always pin to an immutable digest:

```yaml
image: httpd@sha256:331548c5249bdeced0f048bc2fb8c6b6427d2ec6508bed9c1fec6c57d0b27a60
```

This prevents unexpected behaviour from upstream image updates and ensures reproducible deployments.

### 6. Monitor QoS Class and Size Limits Appropriately

Validate that your memory limit leaves sufficient headroom for the application's actual working set. A limit too close to actual usage risks OOMKill under load spikes. A 25-30% headroom above the observed peak RSS is a reasonable starting point before tuning with Vertical Pod Autoscaler (VPA) data.

### 7. Validate the Namespace Before Provisioning

Always run `kubectl get pods -n <namespace>` before applying new manifests. Pre-existing pods with the same name in a `Terminating` state can cause apply to block or behave unexpectedly.

### 8. Use Heredoc for Reproducible Manifest Creation

The `cat <<'EOF' > file.yaml` heredoc pattern eliminates editor-introduced whitespace or encoding issues and is directly replayable in automation pipelines.

---

## Lessons Learned

**L1: Resource limits are not optional in shared clusters.**
Without limits, a single runaway process can exhaust node memory, triggering cascading OOMKills across co-located pods. Treat resource boundaries as a hard organisational policy enforced via LimitRange objects at the namespace level.

**L2: Server-side dry run is a different validation surface than client-side.**
Client-side dry run validates schema. Server-side dry run validates admission policy. Both are necessary. Skipping server-side dry run means your manifest can pass local validation and still be rejected by webhook-enforced policies at apply time.

**L3: QoS class is an emergent property, not an explicit configuration.**
The QoS class (`Guaranteed`, `Burstable`, `BestEffort`) is computed by the kubelet from your requests and limits configuration. Understanding QoS class is essential for predicting eviction order under node memory pressure. For critical workloads, design resource specs to achieve `Guaranteed` QoS.

**L4: kubectl describe is the authoritative source of applied state.**
JSONPath queries against the spec confirm what was submitted. `kubectl describe` surfaces the live, kubelet-interpreted state including admission-mutated values, injected defaults, and runtime conditions. Always cross-reference both.

**L5: Image pull time is a latency factor in pod startup SLAs.**
The httpd:latest image (45MB) pulled in 3.026 seconds. In production, pre-pull critical images using DaemonSet-managed image caches or configure `imagePullPolicy: IfNotPresent` with pinned digests to eliminate pull latency from pod startup time.

**L6: Terminal artefacts in captured sessions are cosmetic only.**
Prompt corruption after `Ctrl+C` on watch commands is a known terminal rendering behaviour. It does not indicate command failure. Develop the habit of pressing Enter after any interrupted streaming command before continuing.

---

## Key Commands Reference

```bash
# Cluster diagnostics
kubectl version --client
kubectl cluster-info
kubectl config current-context
kubectl get pods -n default

# Manifest management
cat <<'EOF' > httpd-pod.yaml
# ... manifest content ...
EOF

# Validation pipeline
kubectl apply -f httpd-pod.yaml --dry-run=client
kubectl apply -f httpd-pod.yaml --dry-run=server
kubectl apply -f httpd-pod.yaml

# Runtime observation
kubectl get pod httpd-pod --watch
kubectl get pod httpd-pod

# Targeted field verification
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].name}'
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].image}'
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].resources.requests}'
kubectl get pod httpd-pod -o jsonpath='{.spec.containers[0].resources.limits}'

# Full pod inspection
kubectl describe pod httpd-pod
```

---

## Related Resources

- [Kubernetes Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [LimitRange: Default Limits for Namespaces](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---









<img width="1031" height="611" alt="image" src="https://github.com/user-attachments/assets/6340598c-7dce-4b89-a746-482f1fe6f227" />
<img width="1025" height="625" alt="image" src="https://github.com/user-attachments/assets/d7f3569a-267b-4edb-9f6e-a4161fde7785" />
<img width="1029" height="648" alt="image" src="https://github.com/user-attachments/assets/f4b12228-744c-41e9-a267-63f0b6e474d5" />
<img width="1032" height="863" alt="image" src="https://github.com/user-attachments/assets/e2ea2782-3cbe-4c71-9caf-d279358ac0f5" />
<img width="1031" height="857" alt="image" src="https://github.com/user-attachments/assets/fba7626f-3480-45b7-bd39-75270a93fb0b" />

