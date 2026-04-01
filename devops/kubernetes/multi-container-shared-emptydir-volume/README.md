# Kubernetes Shared Volume Between Containers in a Pod

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify Cluster Health](#step-1-verify-cluster-health)
  - [Step 2: Author the Pod Manifest](#step-2-author-the-pod-manifest)
  - [Step 3: Deploy the Pod](#step-3-deploy-the-pod)
  - [Step 4: Confirm Pod Readiness](#step-4-confirm-pod-readiness)
  - [Step 5: Inspect Pod Configuration](#step-5-inspect-pod-configuration)
  - [Step 6: Write Test Data from Container 1](#step-6-write-test-data-from-container-1)
  - [Step 7: Verify Data Visibility from Container 2](#step-7-verify-data-visibility-from-container-2)
- [Manifest Reference](#manifest-reference)
- [Verification Summary](#verification-summary)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)

---

## Overview

This document details the end-to-end implementation of a Kubernetes pod that runs two containers sharing a single `emptyDir` volume. The exercise is part of the **xFusionCorp Industries** infrastructure lab series and demonstrates inter-container data sharing within the same pod lifecycle.

| Field | Value |
|---|---|
| **Platform** | Kubernetes (k3s v1.34.1) |
| **Cluster Host** | `jump-host` |
| **Pod Name** | `volume-share-xfusion` |
| **Container Image** | `fedora:latest` |
| **Volume Type** | `emptyDir` |
| **Volume Name** | `volume-share` |
| **Container 1 Mount Path** | `/tmp/blog` |
| **Container 2 Mount Path** | `/tmp/apps` |

---

## Problem Statement

In cloud-native applications, multiple containers within the same pod frequently need to exchange temporary data, for example a sidecar log processor reading files written by an application container. Kubernetes supports this pattern natively through shared volumes defined at the pod spec level.

The requirement for this lab is:

1. Create a pod named `volume-share-xfusion` containing two containers.
2. Both containers use the `fedora:latest` image and run a `sleep` command to remain alive.
3. Both containers mount the same `emptyDir` volume named `volume-share`, each at a different path.
4. A file written under the mount path of the first container must be readable from the mount path of the second container.

---

## Solution Architecture

```
Pod: volume-share-xfusion
+--------------------------------------------+
|                                            |
|  Container 1                Container 2    |
|  volume-container-xfusion-1  volume-container-xfusion-2  |
|  /tmp/blog  <----+     +---> /tmp/apps     |
|                  |     |                   |
|            Volume: volume-share            |
|              (emptyDir)                    |
+--------------------------------------------+
```

Both containers share the same underlying ephemeral directory on the node. Writes made by one container are immediately visible to the other because they resolve to the same inode tree on the host, scoped to the pod lifetime.

---

## Prerequisites

* `kubectl` configured and pointing to the target cluster.
* Cluster node in `Ready` state.
* Network access to pull `fedora:latest` from Docker Hub.

---

## Implementation Guide

### Step 1: Verify Cluster Health

Before any deployment, confirm that the control plane and the single node are operational.

```bash
kubectl get nodes
```

**Expected output:**

```
NAME        STATUS   ROLES           AGE    VERSION
jump-host   Ready    control-plane   151m   v1.34.1+k3s1
```

```bash
kubectl cluster-info
```

**Expected output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

> **Screenshot: Cluster health verification output showing node Ready status and control plane endpoints**

<img width="1030" height="423" alt="image" src="https://github.com/user-attachments/assets/f5f2a9a5-2f9b-4034-a2dd-365ca93ce067" />

---

### Step 2: Author the Pod Manifest

Write the pod manifest to disk using a heredoc. This approach keeps the manifest self-contained in the shell session and eliminates dependency on an external editor.

```bash
cat <<'EOF' > volume-share-xfusion.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-share-xfusion
spec:
  containers:
  - name: volume-container-xfusion-1
    image: fedora:latest
    command: ["sleep", "10000"]
    volumeMounts:
    - name: volume-share
      mountPath: /tmp/blog

  - name: volume-container-xfusion-2
    image: fedora:latest
    command: ["sleep", "10000"]
    volumeMounts:
    - name: volume-share
      mountPath: /tmp/apps

  volumes:
  - name: volume-share
    emptyDir: {}
EOF
```

Confirm the file was written correctly before applying:

```bash
cat volume-share-xfusion.yaml
```

> **Screenshots: Terminal output showing the heredoc written to volume-share-xfusion.yaml and the confirmed cat output**

<img width="1030" height="717" alt="image" src="https://github.com/user-attachments/assets/12087503-b04f-4ee2-993a-8c45992a07b8" />
<img width="1036" height="865" alt="image" src="https://github.com/user-attachments/assets/0bb3e0b6-6880-4deb-aa25-a2df7f18dacb" />

---

### Step 3: Deploy the Pod

Apply the manifest to the cluster using `kubectl apply`. This is preferred over `kubectl create` because it is idempotent and safe to re-run.

```bash
kubectl apply -f volume-share-xfusion.yaml
```

**Expected output:**

```
pod/volume-share-xfusion created
```

---

### Step 4: Confirm Pod Readiness

Watch the pod until both containers report `Running` and the `READY` column shows `2/2`.

```bash
kubectl get pod volume-share-xfusion --watch
```

**Expected output (once stable):**

```
NAME                   READY   STATUS    RESTARTS   AGE
volume-share-xfusion   2/2     Running   0          90s
```

Press `Ctrl+C` to exit the watch loop once `2/2 Running` is confirmed. Verify once more without the watch flag:

```bash
kubectl get pod volume-share-xfusion
```

**Expected output:**

```
NAME                   READY   STATUS    RESTARTS   AGE
volume-share-xfusion   2/2     Running   0          2m42s
```

> **Screenshot Placeholder: kubectl get pod output showing 2/2 Running state with zero restarts**

---

### Step 5: Inspect Pod Configuration

Run `kubectl describe` to validate volume mounts, container states, and volume definitions before performing the data-sharing test.

```bash
kubectl describe pod volume-share-xfusion
```

Key sections to verify in the output:

**Container 1 mount confirmation:**
```
Mounts:
  /tmp/blog from volume-share (rw)
```

**Container 2 mount confirmation:**
```
Mounts:
  /tmp/apps from volume-share (rw)
```

**Volume definition confirmation:**
```
Volumes:
  volume-share:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
```

> **Screenshot Placeholder: kubectl describe pod output highlighting volume mount paths for both containers and the emptyDir volume definition**

---

### Step 6: Write Test Data from Container 1

Exec into `volume-container-xfusion-1` and write a test file under the mounted path `/tmp/blog`. The `-c` flag explicitly targets the named container within the multi-container pod.

```bash
kubectl exec -it volume-share-xfusion -c volume-container-xfusion-1 -- /bin/bash
```

Inside the container shell, write the file and confirm the content:

```bash
echo "Welcome to xFusionCorp Industries" > /tmp/blog/blog.txt
cat /tmp/blog/blog.txt
```

**Expected output:**

```
Welcome to xFusionCorp Industries
```

Exit the container shell:

```bash
exit
```

> **Screenshot Placeholder: Interactive exec session into volume-container-xfusion-1 showing the echo write and cat read of blog.txt**

---

### Step 7: Verify Data Visibility from Container 2

Without entering an interactive shell, directly exec a `cat` command against `volume-container-xfusion-2` to confirm the file written by Container 1 is readable at `/tmp/apps/blog.txt`.

```bash
kubectl exec -it volume-share-xfusion -c volume-container-xfusion-2 -- cat /tmp/apps/blog.txt
```

**Expected output:**

```
Welcome to xFusionCorp Industries
```

This output confirms the `emptyDir` volume is correctly shared between both containers. The file was written to `/tmp/blog/blog.txt` in Container 1 and is visible at `/tmp/apps/blog.txt` in Container 2 because both paths resolve to the same underlying volume.

> **Screenshot Placeholder: kubectl exec output from volume-container-xfusion-2 displaying the contents of blog.txt confirming cross-container volume sharing**

---

## Manifest Reference

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-share-xfusion
spec:
  containers:
  - name: volume-container-xfusion-1
    image: fedora:latest
    command: ["sleep", "10000"]
    volumeMounts:
    - name: volume-share
      mountPath: /tmp/blog

  - name: volume-container-xfusion-2
    image: fedora:latest
    command: ["sleep", "10000"]
    volumeMounts:
    - name: volume-share
      mountPath: /tmp/apps

  volumes:
  - name: volume-share
    emptyDir: {}
```

---

## Verification Summary

| Check | Command | Expected Result |
|---|---|---|
| Node health | `kubectl get nodes` | `Ready` |
| Pod running | `kubectl get pod volume-share-xfusion` | `2/2 Running` |
| Volume mounts | `kubectl describe pod volume-share-xfusion` | Both containers show `volume-share` mounted |
| Write from Container 1 | `exec -c volume-container-xfusion-1 -- cat /tmp/blog/blog.txt` | `Welcome to xFusionCorp Industries` |
| Read from Container 2 | `exec -c volume-container-xfusion-2 -- cat /tmp/apps/blog.txt` | `Welcome to xFusionCorp Industries` |

---

## Best Practices Applied

* **Declarative manifest over imperative commands.** Using `kubectl apply -f` with a YAML file ensures the configuration is version-controllable and reproducible.

* **Explicit container targeting with `-c`.** In multi-container pods, always supply the `-c <container-name>` flag to `kubectl exec`. Omitting it causes Kubernetes to default to the first container alphabetically, which can lead to confusing results and is not deterministic across spec reorderings.

* **Heredoc for inline manifest authoring.** Using `cat <<'EOF' > file.yaml` with a quoted delimiter (`'EOF'`) prevents shell variable expansion inside the heredoc body, which protects YAML templating characters from unintended substitution.

* **`--watch` for readiness gating.** Using `kubectl get pod --watch` instead of polling manually ensures the operator only proceeds once the pod has fully transitioned to `Running`, avoiding premature exec commands against containers that have not finished initializing.

* **`kubectl describe` for pre-flight validation.** Inspecting the pod with `describe` before running exec workloads confirms that volume mounts were applied correctly and surfaces any admission or scheduling events that could indicate a misconfiguration.

* **Non-persistent volume scoped to pod lifetime.** Using `emptyDir` is appropriate when data only needs to survive for the duration of the pod. It avoids the operational overhead of `PersistentVolumeClaims` for temporary inter-container communication patterns.

---

## Lessons Learned

* **`emptyDir` is node-local and ephemeral.** If the pod is deleted and recreated, the volume and all its contents are destroyed. This volume type is not suitable for any data that needs to survive a pod restart or rescheduling event.

* **Mount paths are independent per container.** Each container defines its own `mountPath`, meaning the same volume can appear under a completely different directory path in each container. The volume name (`volume-share`) is the binding key, not the path.

* **Image pull time affects readiness timing.** In this run, `fedora:latest` was pulled from Docker Hub in under 3 seconds for the first container because it was already cached on the node after the first pull. The second container pull completed in 311ms. In environments without a local registry or pull-through cache, this step can be a significant bottleneck.

* **Multi-container pods require explicit container selection for exec.** Running `kubectl exec` without `-c` on a multi-container pod will target the first container listed in the spec. Explicitly naming the container eliminates ambiguity and is required for reliable automation.

* **`sleep 10000` as a container entrypoint is a lab pattern only.** In production, containers should run meaningful workloads with proper liveness and readiness probes. Using `sleep` as the sole command means Kubernetes has no health signal beyond the process being alive.

---

## Troubleshooting

**Pod stuck in `ContainerCreating`**

Run `kubectl describe pod volume-share-xfusion` and inspect the Events section. The most common causes are image pull failures (check network access to Docker Hub) or insufficient node resources.

**`exec` returns `error: unable to upgrade connection`**

This typically indicates a network policy or firewall rule blocking the WebSocket upgrade between the `kubectl` client and the API server. Verify the control plane is reachable and the kubeconfig context is correct.

**File not visible in Container 2**

Confirm both containers reference the same volume name (`volume-share`) in their `volumeMounts` entries. A typo in the volume name will result in Kubernetes creating a separate, unnamed volume instead of sharing the declared one, and the pod may fail to schedule.

**`/bin/bash` not found in container**

If the image variant in use does not include bash, substitute `/bin/sh`. The `fedora:latest` image ships with bash, but this is relevant when adapting this pattern to minimal base images such as `alpine` or `distroless`.




<img width="1025" height="626" alt="image" src="https://github.com/user-attachments/assets/afeed81a-c2e2-4842-8bbc-7daab94bc771" />
<img width="1035" height="669" alt="image" src="https://github.com/user-attachments/assets/ef1e0173-4980-4bac-ab45-cbf50ae2a6fe" />
<img width="1028" height="734" alt="image" src="https://github.com/user-attachments/assets/a38990c7-1e59-4da1-b299-fd61b7110181" />
<img width="1033" height="864" alt="image" src="https://github.com/user-attachments/assets/7088c1d9-1c02-4fb0-836a-3f64ea0106b5" />
<img width="1031" height="866" alt="image" src="https://github.com/user-attachments/assets/b9042c8f-0085-4b5a-b778-eb7f8a2ef9cf" />
<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/1d011f0c-13e0-4f8a-a251-d8a8cd790f38" />
<img width="1033" height="324" alt="image" src="https://github.com/user-attachments/assets/69d76203-712d-4985-ae7e-e51e42a3769f" />
<img width="1032" height="342" alt="image" src="https://github.com/user-attachments/assets/da44d6b2-8ef1-44fd-9292-1222a1751011" />
<img width="1032" height="382" alt="image" src="https://github.com/user-attachments/assets/f39d9c41-21ec-40c7-9966-2d1ed3ffe80b" />
<img width="1034" height="419" alt="image" src="https://github.com/user-attachments/assets/0ece8fdd-cec1-4e49-a739-cc5913809b32" />
<img width="1028" height="465" alt="image" src="https://github.com/user-attachments/assets/fb580837-e769-4f43-8598-bea3094fa8db" />
