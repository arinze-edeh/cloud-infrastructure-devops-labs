# Kubernetes Pod Environment Variable Injection via Native Spec Fields

> Validating runtime configuration propagation into containerized workloads using Kubernetes-native `env` spec fields on a K3s single-node cluster.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Design Intent](#architecture-and-design-intent)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: Verify Cluster Health](#step-1-verify-cluster-health)
  - [Step 2: Author the Pod Manifest](#step-2-author-the-pod-manifest)
  - [Step 3: Apply the Manifest](#step-3-apply-the-manifest)
  - [Step 4: Confirm Pod Completion](#step-4-confirm-pod-completion)
  - [Step 5: Validate Output via Pod Logs](#step-5-validate-output-via-pod-logs)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Environment Reference](#environment-reference)

---

## Overview

This exercise provisions a Kubernetes pod that demonstrates environment variable injection at the container level using native Kubernetes `env` spec fields. The pod executes a shell command that reads injected variables and prints a concatenated greeting string, validating that runtime environment configuration is correctly propagated into containerized workloads.

The implementation reflects a foundational but operationally significant pattern: **decoupling configuration from container images**, enabling environment-specific deployments without rebuilding image artifacts. This pattern underpins twelve-factor application design and is a prerequisite skill for managing environment-specific rollouts across staging, QA, and production Kubernetes clusters.

---

## Problem Statement

The Nautilus DevOps team requires a prerequisite pod to validate that environment variables can be injected into a container and consumed at runtime before rolling out a broader greeting application. The pod must satisfy the following requirements:

- Be named `print-envars-greeting` with a container named `print-env-container`
- Use the `bash` image
- Inject three environment variables: `GREETING`, `COMPANY`, and `GROUP`
- Execute a shell command that echoes all three variables in a single output line
- Terminate cleanly with `restartPolicy: Never` to prevent crash loop conditions

---

## Architecture and Design Intent

```
┌──────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                  │
│                                                      │
│  ┌───────────────────────────────────────────────┐   │
│  │         Pod: print-envars-greeting            │   │
│  │                                               │   │
│  │  Container: print-env-container (bash)        │   │
│  │                                               │   │
│  │  ENV:                                         │   │
│  │    GREETING  = "Welcome to"                   │   │
│  │    COMPANY   = "xFusionCorp"                  │   │
│  │    GROUP     = "Datacenter"                   │   │
│  │                                               │   │
│  │  CMD: echo "$GREETING $COMPANY $GROUP"        │   │
│  │  Output: Welcome to xFusionCorp Datacenter    │   │
│  │                                               │   │
│  │  restartPolicy: Never                         │   │
│  └───────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

**Key Design Decisions:**

- **`restartPolicy: Never`:** The pod is a one-shot task, not a long-running service. Setting `Never` prevents Kubernetes from rescheduling the container after exit, which would generate spurious `CrashLoopBackOff` events for a container that completes successfully with exit code `0`.
- **`bash` image:** Chosen explicitly because it ships with `/bin/sh` and handles variable interpolation natively without additional tooling.
- **Inline `env` fields vs. ConfigMap:** Direct injection is appropriate at this scope. For multi-pod or multi-environment promotion scenarios, externalizing values into a `ConfigMap` or `Secret` is strongly preferred.
- **Double-quoted echo string:** Wrapping the echo argument in double quotes ensures all three variables are expanded by the shell in a single expression, preserving spacing and preventing word splitting on values that contain whitespace.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Kubernetes cluster | K3s single-node cluster accessible via `jump-host` |
| `kubectl` | Pre-configured on `jump-host` targeting `127.0.0.1:6443` |
| Cluster access | Control plane reachable and nodes in `Ready` state |

---

## Implementation

### Step 1: Verify Cluster Health

Confirm the control plane is reachable and the node is in a `Ready` state before proceeding. Applying a manifest against a degraded cluster can result in pending pods or silent scheduling failures that are difficult to triage.

```bash
kubectl cluster-info
kubectl get nodes
```

**Screenshot: Cluster info and node status confirming control plane reachability and node readiness**

![Cluster info and node status output](https://github.com/user-attachments/assets/25fe4798-5e1e-490c-8c7f-f0f11e72dd96)

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   59m   v1.34.1+k3s1
```

**Expected result:** The control plane URL is returned and the node shows `STATUS: Ready`. Any other node status (`NotReady`, `SchedulingDisabled`) must be resolved before proceeding.

---

### Step 2: Author the Pod Manifest

Write the pod manifest using a heredoc to eliminate editor dependency and guarantee exact field fidelity. In constrained environments without a preferred editor, heredoc is the most reliable way to write multi-line YAML without introducing tab or indentation corruption.

```bash
cat << 'EOF' > print-envars-greeting.yaml
apiVersion: v1
kind: Pod
metadata:
  name: print-envars-greeting
spec:
  restartPolicy: Never
  containers:
    - name: print-env-container
      image: bash
      env:
        - name: GREETING
          value: "Welcome to"
        - name: COMPANY
          value: "xFusionCorp"
        - name: GROUP
          value: "Datacenter"
      command: ["/bin/sh", "-c", "echo \"$GREETING $COMPANY $GROUP\""]
EOF
```

**Screenshot: Heredoc manifest written to `print-envars-greeting.yaml`, confirming successful file creation**

![Heredoc manifest written to print-envars-greeting.yaml](https://github.com/user-attachments/assets/6fe3d77c-df8c-49c9-b92b-6357441d3787)

Verify the file contents before applying to catch any field-level errors before they reach the API server:

```bash
cat print-envars-greeting.yaml
```

**Screenshot: `cat` output confirming full manifest contents are intact and correctly structured**

![cat output showing full manifest contents](https://github.com/user-attachments/assets/3ad1ddf5-8d79-4679-9ae5-4dc18611dc28)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: print-envars-greeting
spec:
  restartPolicy: Never
  containers:
    - name: print-env-container
      image: bash
      env:
        - name: GREETING
          value: "Welcome to"
        - name: COMPANY
          value: "xFusionCorp"
        - name: GROUP
          value: "Datacenter"
      command: ["/bin/sh", "-c", "echo \"$GREETING $COMPANY $GROUP\""]
```

> **Operational note:** Always inspect the manifest with `cat` or `kubectl apply --dry-run=client -f` before submitting to the API server. This catches YAML indentation errors, missing required fields, and incorrect value types before they surface as cryptic API rejection errors.

---

### Step 3: Apply the Manifest

Submit the pod definition to the Kubernetes API server. The `apply` subcommand is used rather than `create` to maintain idempotency: if the manifest is reapplied after a previous run, Kubernetes will compare the desired state and update rather than returning a conflict error.

```bash
kubectl apply -f print-envars-greeting.yaml
```

**Screenshot: `kubectl apply` confirmation showing the pod resource was accepted and created by the API server**

![kubectl apply confirmation output](https://github.com/user-attachments/assets/2313843b-3088-4275-96e8-c89d4db08477)

```
pod/print-envars-greeting created
```

**Expected result:** The API server accepts the manifest and confirms resource creation. Any rejection at this stage is a manifest schema error and must be resolved before proceeding.

---

### Step 4: Confirm Pod Completion

Watch the pod lifecycle until it reaches `Completed` status, confirming it executed successfully and exited without error. The `--watch` flag streams live status updates so you do not need to poll manually.

```bash
kubectl get pod print-envars-greeting --watch
```

**Screenshot: Pod watch output showing `Completed` status with zero restarts, confirming clean one-shot execution**

![Pod watch output showing Completed status](https://github.com/user-attachments/assets/55290e40-2ee2-47a1-93da-d14c95b22d4f)

```
NAME                    READY   STATUS      RESTARTS   AGE
print-envars-greeting   0/1     Completed   0          26s
```

**Interpreting the output:**

- **`STATUS: Completed`** confirms the container ran to successful termination. This is the authoritative signal for one-shot workloads.
- **`RESTARTS: 0`** confirms `restartPolicy: Never` prevented any rescheduling after exit.
- **`READY: 0/1`** is expected for a completed pod. Kubernetes marks a pod as not ready once its containers exit, regardless of exit code. This does not indicate a failure condition.

> **Operational note:** If the pod image is not cached on the node, the initial pull may cause a brief `ContainerCreating` phase before transitioning to `Completed`. This is normal pull latency and resolves automatically.

---

### Step 5: Validate Output via Pod Logs

Stream the pod logs to confirm the environment variables were correctly injected and the echo command produced the expected output. Log validation is the definitive end-to-end proof that the Kubernetes `env` fields propagated correctly through the container runtime to the shell environment.

```bash
kubectl logs -f print-envars-greeting
```

**Screenshot: `kubectl logs` output confirming all three environment variables were resolved and concatenated correctly at runtime**

![kubectl logs output showing greeting string](https://github.com/user-attachments/assets/59efaad6-803e-4cae-bcc8-1c955aa34b17)

```
Welcome to xFusionCorp Datacenter
```

The output confirms all three environment variables were resolved and concatenated correctly at runtime, validating the full configuration propagation chain from pod spec to container shell.

---

## Errors and Resolutions

No errors were encountered during this implementation. The manifest applied cleanly, the pod reached `Completed` status in under 30 seconds, and the log output matched the expected string exactly.

**Potential failure modes to be aware of:**

| Scenario | Symptom | Resolution |
|---|---|---|
| Image pull failure | Pod stuck in `ErrImagePull` | Verify image name, check node internet connectivity |
| YAML indentation error | API server rejects manifest | Run `kubectl apply --dry-run=client -f` before applying |
| `restartPolicy` misplaced under container | Manifest validation error, pod not created | Move `restartPolicy` to top-level `spec` field |
| Variable with spaces unquoted in YAML | Incorrect runtime value or parse error | Wrap all string values containing whitespace in double quotes |
| `CrashLoopBackOff` on a one-shot pod | `restartPolicy` not set to `Never` | Add `restartPolicy: Never` at the spec level |

---

## Best Practices Applied

- **Manifest-first workflow:** The pod was defined declaratively in a YAML file rather than imperatively with `kubectl run`, enabling version control, peer review, and reproducibility across environments.
- **Heredoc for manifest authoring:** Using `cat << 'EOF'` eliminates editor dependency in constrained environments and preserves exact formatting without risk of tab or space corruption.
- **Pre-apply manifest inspection:** Running `cat print-envars-greeting.yaml` before `kubectl apply` catches field errors before they reach the API server, reducing unnecessary API round trips and cluster event noise.
- **`restartPolicy: Never` for one-shot workloads:** Prevents Kubernetes from misclassifying a cleanly exiting container as a crash, which would trigger unnecessary `CrashLoopBackOff` state and pollute cluster event logs.
- **Shell-level variable expansion validation:** Streaming logs immediately after pod completion confirms end-to-end propagation from Kubernetes `env` fields through the container runtime to the shell environment.
- **Double-quoting the echo argument:** Ensures the three-variable expression is treated as a single shell word, preserving spaces in variable values such as `"Welcome to"` and preventing unintended word splitting.

---

## Lessons Learned

- **`READY: 0/1` on a completed pod is not a failure.** Kubernetes marks a pod as not ready once its containers exit, regardless of exit code. `STATUS: Completed` is the authoritative signal for successful one-shot workloads. Misreading `0/1` as an error is a common point of confusion for engineers new to batch-style pods.
- **`restartPolicy` placement matters.** In a pod spec, `restartPolicy` is a top-level field under `spec`, not a field under `containers`. Misplacing it under the container block will cause a manifest validation error and the pod will not be created.
- **Environment variable values with spaces require quoting in YAML.** The value `"Welcome to"` contains a space. Without explicit quoting in the YAML, some parsers may interpret this incorrectly. Always quote string values in Kubernetes manifests when they contain whitespace or special characters.
- **`kubectl logs -f` on a completed pod is safe.** The `-f` (follow) flag does not hang on a completed pod; it streams the captured output and returns immediately. This makes it a reliable pattern for validating one-shot job output without needing to know the exact completion timing.
- **Inline `env` is appropriate for single-pod scope.** For multi-pod or multi-environment deployments, externalizing variable values into a `ConfigMap` or `Secret` is the production-grade pattern. This avoids duplicating configuration across manifests and enables centralized updates without touching individual pod specs.

---

## Environment Reference

| Parameter | Value |
|---|---|
| Cluster type | K3s single-node |
| Control plane endpoint | `https://127.0.0.1:6443` |
| Access method | `kubectl` on `jump-host` |
| Pod name | `print-envars-greeting` |
| Container name | `print-env-container` |
| Container image | `bash` |
| `GREETING` value | `Welcome to` |
| `COMPANY` value | `xFusionCorp` |
| `GROUP` value | `Datacenter` |
| Restart policy | `Never` |
| Expected log output | `Welcome to xFusionCorp Datacenter` |
