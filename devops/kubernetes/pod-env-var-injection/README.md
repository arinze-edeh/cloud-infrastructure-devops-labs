# Kubernetes Pod Environment Variables Injection

## Overview

This lab provisions a Kubernetes pod that demonstrates environment variable injection at the container level using native Kubernetes `env` spec fields. The pod executes a shell command that reads injected variables and prints a concatenated greeting string, validating that runtime environment configuration is correctly propagated into containerized workloads.

The implementation reflects a foundational but operationally significant pattern: decoupling configuration from container images, enabling environment-specific deployments without rebuilding image artifacts.

---

## Problem Statement

The Nautilus DevOps team requires a pre-requisite pod to validate that environment variables can be injected into a container and consumed at runtime before rolling out a broader greeting application. The pod must:

* Be named `print-envars-greeting` with a container named `print-env-container`
* Use the `bash` image
* Inject three environment variables: `GREETING`, `COMPANY`, and `GROUP`
* Execute a shell command that echoes all three variables in a single output line
* Terminate cleanly with `restartPolicy: Never` to prevent crash loop conditions

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

**Key Decisions:**

* **`restartPolicy: Never`:** The pod is a one-shot task, not a long-running service. Setting `Never` prevents Kubernetes from re-scheduling the container after exit, which would generate spurious `CrashLoopBackOff` events for a container that completes successfully with exit code `0`.
* **`bash` image:** Chosen explicitly because it ships with `/bin/sh` and handles variable interpolation natively without additional tooling.
* **Inline `env` fields vs. ConfigMap:** Direct injection is appropriate at this scope. For multi-pod or multi-environment promotion scenarios, externalizing values into a `ConfigMap` or `Secret` would be preferred.
* **Double-quoted echo string:** Wrapping the echo argument in double quotes ensures all three variables are expanded by the shell in a single expression, preserving spacing and preventing word splitting.

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

Confirm the control plane is reachable and the node is in a `Ready` state before proceeding.

```bash
kubectl cluster-info
kubectl get nodes
```

**Screenshot: Cluster info and node status output**

<img width="1031" height="416" alt="image" src="https://github.com/user-attachments/assets/25fe4798-5e1e-490c-8c7f-f0f11e72dd96" />

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   59m   v1.34.1+k3s1
```

---

### Step 2: Author the Pod Manifest

Write the pod manifest using a heredoc to avoid editor dependency and ensure exact field fidelity.

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

**Screenshot: Heredoc manifest written to print-envars-greeting.yaml**



Verify the file contents before applying:

```bash
cat print-envars-greeting.yaml
```

**Screenshot: cat output showing full manifest contents**

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

---

### Step 3: Apply the Manifest

Submit the pod definition to the Kubernetes API server.

```bash
kubectl apply -f print-envars-greeting.yaml
```

**Screenshot: kubectl apply confirmation output**

```
pod/print-envars-greeting created
```

---

### Step 4: Confirm Pod Completion

Watch the pod lifecycle until it reaches `Completed` status, confirming it executed successfully and exited without error.

```bash
kubectl get pod print-envars-greeting --watch
```

**Screenshot: Pod watch output showing Completed status**

```
NAME                    READY   STATUS      RESTARTS   AGE
print-envars-greeting   0/1     Completed   0          26s
```

* `STATUS: Completed` confirms the container ran to successful termination.
* `RESTARTS: 0` confirms `restartPolicy: Never` prevented any re-scheduling.
* `READY: 0/1` is expected for a completed pod and does not indicate failure.

---

### Step 5: Validate Output via Pod Logs

Stream the pod logs to confirm the environment variables were correctly injected and the echo command produced the expected output.

```bash
kubectl logs -f print-envars-greeting
```

**Screenshot: kubectl logs output showing greeting string**

```
Welcome to xFusionCorp Datacenter
```

The output confirms all three environment variables were resolved and concatenated correctly at runtime.

---

## Errors and Resolutions

No errors were encountered during this implementation. The manifest applied cleanly, the pod reached `Completed` status in under 30 seconds, and the log output matched the expected string exactly.

> **Operational note:** If the pod image is not cached on the node, the initial pull may cause a brief `ContainerCreating` phase before transitioning to `Completed`. This is normal pull latency and not an error condition.

---

## Best Practices Applied

* **Manifest-first workflow:** The pod was defined declaratively in a YAML file rather than imperatively with `kubectl run`, enabling version control, peer review, and reproducibility.
* **Heredoc for manifest authoring:** Using `cat << 'EOF'` eliminates editor dependency in constrained lab environments and preserves exact formatting without risk of tab/space corruption.
* **Pre-apply manifest inspection:** Running `cat print-envars-greeting.yaml` before `kubectl apply` catches field errors before they reach the API server.
* **`restartPolicy: Never` for one-shot workloads:** Prevents Kubernetes from misclassifying a cleanly exiting container as a crash, which would trigger unnecessary `CrashLoopBackOff` state and pollute cluster event logs.
* **Shell-level variable expansion validation:** Streaming logs immediately after pod completion confirms end-to-end propagation from Kubernetes `env` fields through the container runtime to the shell environment.
* **Double-quoting the echo argument:** Ensures the three-variable expression is treated as a single shell word, preserving spaces in variable values such as `"Welcome to"`.

---

## Lessons Learned

* **`READY: 0/1` on a completed pod is not a failure.** Kubernetes marks a pod as not ready once its containers exit, regardless of exit code. `STATUS: Completed` is the authoritative signal for successful one-shot workloads. Misreading `0/1` as an error is a common point of confusion for engineers new to batch-style pods.
* **`restartPolicy` placement matters.** In a pod spec, `restartPolicy` is a top-level field under `spec`, not a field under `containers`. Misplacing it under the container block will cause a manifest validation error and the pod will not be created.
* **Environment variable values with spaces require quoting in YAML.** The value `"Welcome to"` contains a space. Without explicit quoting in the YAML, some parsers may interpret this incorrectly. Always quote string values in Kubernetes manifests when they contain whitespace or special characters.
* **`kubectl logs -f` on a completed pod is safe.** The `-f` (follow) flag does not hang on a completed pod; it streams the captured output and returns immediately. This makes it a reliable pattern for validating one-shot job output without knowing exact completion timing.

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



<img width="1034" height="727" alt="image" src="https://github.com/user-attachments/assets/6fe3d77c-df8c-49c9-b92b-6357441d3787" />
<img width="1025" height="820" alt="image" src="https://github.com/user-attachments/assets/3ad1ddf5-8d79-4679-9ae5-4dc18611dc28" />
<img width="1034" height="418" alt="image" src="https://github.com/user-attachments/assets/2313843b-3088-4275-96e8-c89d4db08477" />
<img width="1033" height="472" alt="image" src="https://github.com/user-attachments/assets/55290e40-2ee2-47a1-93da-d14c95b22d4f" />
<img width="1032" height="507" alt="image" src="https://github.com/user-attachments/assets/59efaad6-803e-4cae-bcc8-1c955aa34b17" />
