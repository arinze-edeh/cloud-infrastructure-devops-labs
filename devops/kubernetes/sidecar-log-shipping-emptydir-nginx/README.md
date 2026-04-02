# Kubernetes Multi-Container Pod with Sidecar Log Shipping Pattern

## Table of Contents

* [Project Overview](#project-overview)
* [Problem Statement](#problem-statement)
* [Architecture and Solution Design](#architecture-and-solution-design)
* [Prerequisites](#prerequisites)
* [Environment Specifications](#environment-specifications)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify Cluster Readiness](#step-1-verify-cluster-readiness)
  * [Step 2: Author the Pod Manifest](#step-2-author-the-pod-manifest)
  * [Step 3: Validate the Manifest (Dry Run)](#step-3-validate-the-manifest-dry-run)
  * [Step 4: Apply the Manifest](#step-4-apply-the-manifest)
  * [Step 5: Confirm Pod Status](#step-5-confirm-pod-status)
  * [Step 6: Monitor Pod Stability](#step-6-monitor-pod-stability)
  * [Step 7: Inspect Pod Details](#step-7-inspect-pod-details)
* [Verification and Validation](#verification-and-validation)
* [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Reference Architecture](#reference-architecture)

---

## Project Overview

This project implements a **multi-container Pod** on a K3s Kubernetes cluster using the **Sidecar pattern**. The workload runs an NGINX web server alongside a dedicated Ubuntu-based sidecar container that continuously reads and ships the web server access and error logs via a shared ephemeral volume.

| Attribute | Value |
|---|---|
| Platform | Kubernetes (K3s v1.34.1) |
| Node | jump-host (control-plane) |
| Pod Name | webserver |
| Primary Container | nginx-container (nginx:latest) |
| Sidecar Container | sidecar-container (ubuntu:latest) |
| Shared Volume | shared-logs (emptyDir) |
| Volume Mount Path | /var/log/nginx |

---

## Problem Statement

The Nautilus development team requires access to the last 24 hours of NGINX access and error logs for issue tracing and debugging. The logs are not critical enough to warrant a persistent volume. However, they must be forwarded to a log-aggregation service in near real-time.

Embedding log-shipping logic directly into the NGINX container would violate the **separation of concerns** principle. NGINX should do one thing and do it well: serve web pages. A second, purpose-built container is needed to handle log shipping independently.

---

## Architecture and Solution Design

The solution implements the **Sidecar Container pattern**, one of the foundational multi-container Pod patterns in Kubernetes.

```
+-------------------------------------------------------+
|                   Pod: webserver                      |
|                                                       |
|  +-----------------+        +---------------------+  |
|  | nginx-container |        | sidecar-container   |  |
|  |                 |        |                     |  |
|  | nginx:latest    |        | ubuntu:latest       |  |
|  |                 |        |                     |  |
|  | Writes logs to  |        | Reads logs from     |  |
|  | /var/log/nginx  |        | /var/log/nginx      |  |
|  +--------+--------+        +---------+-----------+  |
|           |                           |               |
|           +--------+   +--------------+               |
|                    |   |                              |
|           +--------v---v--------+                     |
|           |  emptyDir Volume    |                     |
|           |  shared-logs        |                     |
|           |  /var/log/nginx     |                     |
|           +--------------------+                     |
+-------------------------------------------------------+
```

**Key Design Decisions:**

* **emptyDir volume** is used because logs do not require persistence beyond the Pod's lifetime.
* **Both containers mount the same volume at the same path** (`/var/log/nginx`), enabling the sidecar to read files written by NGINX without any network transport.
* **The sidecar runs a polling loop** (`while true; do cat ...; sleep 30; done`) to periodically ship logs, simulating a log-forwarding agent.
* **Containers share the Pod's network namespace and localhost**, enabling future extension to a local log-aggregation endpoint.

---

## Prerequisites

* `kubectl` configured and pointing to an active Kubernetes cluster
* Cluster node in `Ready` state
* Internet access from the cluster node (for image pulls)
* Sufficient permissions to create Pods in the `default` namespace

---

## Environment Specifications

| Component | Detail |
|---|---|
| Kubernetes Distribution | K3s v1.34.1+k3s1 |
| Control Plane Host | jump-host |
| Control Plane Endpoint | https://127.0.0.1:6443 |
| Node IP | 10.244.164.13 |
| Pod IP (assigned) | 10.22.0.9 |
| Namespace | default |
| NGINX Image Digest | sha256:7150b3a3...751c18 |
| Ubuntu Image Digest | sha256:186072bb...feef0c |

---

## Implementation Guide

### Step 1: Verify Cluster Readiness

Before deploying any workload, confirm that the cluster node is healthy and the control plane components are reachable.

```bash
kubectl get nodes
```

**Expected output:**

```
NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   31m   v1.34.1+k3s1
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

> **Screenshot: `kubectl get nodes` and `kubectl cluster-info` output confirming cluster health**

---

### Step 2: Author the Pod Manifest

Write the Pod manifest using a heredoc directly in the terminal to avoid file transfer overhead and ensure exact reproducibility.

```bash
cat << 'EOF' > webserver-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  volumes:
    - name: shared-logs
      emptyDir: {}
  containers:
    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx

    - name: sidecar-container
      image: ubuntu:latest
      command: ["sh", "-c", "while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
EOF
```

Verify the written file contents before proceeding:

```bash
cat webserver-pod.yaml
```

**Expected output:** The manifest is printed in full with all fields intact, confirming no heredoc truncation or encoding issues.

> **Screenshot: `cat webserver-pod.yaml` output confirming complete and correctly formatted manifest**

**Manifest Breakdown:**

| Field | Value | Purpose |
|---|---|---|
| `spec.volumes[0].name` | shared-logs | Volume identifier referenced by both containers |
| `spec.volumes[0].emptyDir` | `{}` | Ephemeral volume, lifecycle tied to the Pod |
| `nginx-container.image` | nginx:latest | Official NGINX web server image |
| `nginx-container.volumeMounts.mountPath` | /var/log/nginx | Default NGINX log directory |
| `sidecar-container.image` | ubuntu:latest | General-purpose Linux image for the log-shipping agent |
| `sidecar-container.command` | `sh -c while true; do cat ...` | Polling loop reading both access.log and error.log every 30 seconds |
| `sidecar-container.volumeMounts.mountPath` | /var/log/nginx | Same path as NGINX, enabling shared file access |

---

### Step 3: Validate the Manifest (Dry Run)

Run a client-side dry run to validate the manifest against the Kubernetes API schema without submitting the object to the cluster.

```bash
kubectl apply --dry-run=client -f webserver-pod.yaml
```

**Expected output:**

```
pod/webserver created (dry run)
```

> **Screenshot: `kubectl apply --dry-run=client` output confirming schema validation passed**

The `--dry-run=client` flag is a critical pre-flight check. It catches YAML syntax errors, missing required fields, and invalid API versions before any cluster-side processing occurs.

---

### Step 4: Apply the Manifest

Submit the validated manifest to the Kubernetes API server to create the Pod.

```bash
kubectl apply -f webserver-pod.yaml
```

**Expected output:**

```
pod/webserver created
```

> **Screenshot: `kubectl apply -f webserver-pod.yaml` output confirming Pod creation**

---

### Step 5: Confirm Pod Status

Check that both containers within the Pod have started successfully.

```bash
kubectl get pod webserver
```

**Expected output:**

```
NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          41s
```

The `2/2` in the `READY` column confirms that both `nginx-container` and `sidecar-container` are running and healthy. A `0` restart count confirms neither container has crashed.

> **Screenshot: `kubectl get pod webserver` output showing 2/2 Running with 0 restarts**

---

### Step 6: Monitor Pod Stability

Use the watch flag to monitor the Pod in real-time and confirm it remains stable over time without unexpected restarts or state transitions.

```bash
kubectl get pod webserver -w
```

**Expected output:**

```
NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          76s
```

The Pod remained in `Running` state with `0` restarts throughout the watch period, confirming stable container behavior. Press `Ctrl+C` to exit the watch.

> **Screenshot: `kubectl get pod webserver -w` output showing sustained Running status**

Optionally, retrieve extended node-level placement details:

```bash
kubectl get pod webserver -o wide
```

**Expected output:**

```
NAME        READY   STATUS    RESTARTS   AGE    IP          NODE        NOMINATED NODE   READINESS GATES
webserver   2/2     Running   0          2m4s   10.22.0.9   jump-host   <none>           <none>
```

> **Screenshot: `kubectl get pod webserver -o wide` output showing Pod IP and node assignment**

---

### Step 7: Inspect Pod Details

Run a full describe to confirm all container states, volume mounts, image digests, and events are as expected.

```bash
kubectl describe pod webserver
```

**Key sections from the output:**

**Container: nginx-container**

```
Image:          nginx:latest
Image ID:       docker.io/library/nginx@sha256:7150b3a39203cb5bee612ff4a9d18774f8c7caf6399d6e8985e97e28eb751c18
State:          Running
  Started:      Thu, 02 Apr 2026 05:58:12 +0000
Ready:          True
Restart Count:  0
Mounts:
  /var/log/nginx from shared-logs (rw)
```

**Container: sidecar-container**

```
Image:         ubuntu:latest
Image ID:      docker.io/library/ubuntu@sha256:186072bba1b2f436cbb91ef2567abca677337cfc786c86e107d25b7072feef0c
Command:
  sh
  -c
  while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done
State:          Running
  Started:      Thu, 02 Apr 2026 05:58:13 +0000
Ready:          True
Restart Count:  0
Mounts:
  /var/log/nginx from shared-logs (rw)
```

**Volume:**

```
shared-logs:
  Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
  Medium:
  SizeLimit:  <unset>
```

**QoS Class:** `BestEffort`

**Pod Conditions (all True):**

| Condition | Status |
|---|---|
| PodReadyToStartContainers | True |
| Initialized | True |
| Ready | True |
| ContainersReady | True |
| PodScheduled | True |

**Event Timeline:**

| Time | Event | Detail |
|---|---|---|
| 2m43s | Scheduled | Pod assigned to jump-host by default-scheduler |
| 2m43s | Pulling | Pulling nginx:latest |
| 2m39s | Pulled | nginx:latest pulled in 3.225s (62,958,873 bytes) |
| 2m39s | Created | nginx-container created |
| 2m39s | Started | nginx-container started |
| 2m39s | Pulling | Pulling ubuntu:latest |
| 2m38s | Pulled | ubuntu:latest pulled in 1.402s (29,741,401 bytes) |
| 2m38s | Created | sidecar-container created |
| 2m38s | Started | sidecar-container started |

> **Screenshot: `kubectl describe pod webserver` full output including container states, volume mounts, and event log**

---

## Verification and Validation

The following criteria confirm a successful deployment:

* `kubectl get pod webserver` reports `2/2 Running` with `0` restarts
* `kubectl describe pod webserver` shows both containers in `State: Running` with `Ready: True`
* Both containers mount the `shared-logs` emptyDir volume at `/var/log/nginx` in `rw` mode
* All five Pod conditions report `True`
* The event log shows sequential image pull, create, and start events with no errors
* The sidecar command matches exactly: `sh -c while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done`

---

## Errors Encountered and Resolutions

### Minor: Typo in `kubectl get` Command After Exiting Watch

**Observed:** After pressing `Ctrl+C` to exit the watch, the command `kubectl get pod webserver -o wide` was accidentally entered as `kubectl get pod webserver -o widede` (extra characters appended to the flag value).

**Symptom:** The flag was not recognized by `kubectl`, resulting in an error or malformed output.

**Root Cause:** Manual typing error immediately after the watch exit interrupt.

**Resolution:** Re-entered the command correctly with the `-o wide` flag:

```bash
kubectl get pod webserver -o wide
```

**Lesson:** Use shell history (`Up Arrow`) to recall and re-execute commands rather than retyping them manually, especially following interrupt sequences like `Ctrl+C`.

---

## Best Practices Applied

* **Manifest-first approach:** The Pod was defined entirely as a declarative YAML manifest rather than using imperative `kubectl run` flags. This ensures the configuration is version-controllable and reproducible.

* **Heredoc for in-terminal authoring:** The `cat << 'EOF' > file.yaml` heredoc pattern was used to write the manifest directly in the terminal. Single-quoting `'EOF'` prevents shell variable expansion inside the block, ensuring literal content is written as authored.

* **Dry-run validation before apply:** `kubectl apply --dry-run=client` was executed before the live apply. This is a mandatory pre-flight step in production workflows to catch schema violations without side effects.

* **Separation of concerns via Sidecar pattern:** NGINX is responsible exclusively for serving web content. The sidecar container handles log reading and shipping. Neither container is aware of the other's internal implementation.

* **emptyDir for ephemeral shared state:** Using `emptyDir` is the correct volume type when shared storage is needed only for the duration of the Pod's lifetime and persistence is not required. It is lightweight, requires no provisioner, and is automatically cleaned up when the Pod is deleted.

* **Explicit image tags:** Both containers use the `:latest` tag as specified by the task requirements. In production environments outside of controlled lab exercises, explicit version-pinned tags (e.g., `nginx:1.27.0`) should be used to ensure deployment reproducibility and avoid unexpected behavior from upstream image updates.

* **Watch-based stability check:** `kubectl get pod -w` was used to observe the Pod in real-time following deployment. This is preferable to a one-time status check because it surfaces transient failures such as `CrashLoopBackOff` or `OOMKilled` that may not appear immediately.

* **Describe-based deep validation:** `kubectl describe pod` was used as the final validation step. The Events section provides a precise chronological record of scheduler decisions, image pulls, and container lifecycle transitions, which is essential for audit trails and post-deployment reviews.

---

## Lessons Learned

* **emptyDir is the correct primitive for intra-Pod file sharing.** Containers within the same Pod share network and IPC namespaces but have isolated filesystems. The only supported mechanism for sharing files between containers in the same Pod is a volume. emptyDir is the simplest and most appropriate choice when the data does not need to survive a Pod restart.

* **Container start order within a Pod is not guaranteed.** Both `nginx-container` and `sidecar-container` start in rapid succession (within 1 second of each other, as shown in the event log). The sidecar's polling loop gracefully handles the scenario where log files do not yet exist, because `cat` on a non-existent file produces a non-fatal error that the shell loop continues past.

* **The QoS class BestEffort has resource scheduling implications.** Because neither container in this Pod defines CPU or memory `requests` or `limits`, Kubernetes assigns the Pod a `BestEffort` QoS class. In resource-constrained environments, BestEffort Pods are the first to be evicted. For production log-shipping workloads, resource requests and limits should be defined to achieve `Burstable` or `Guaranteed` QoS class.

* **Image pull time affects Pod readiness sequencing.** The NGINX image (62.9 MB) took 3.225 seconds to pull, while the Ubuntu image (29.7 MB) took only 1.402 seconds. Pulling order follows manifest declaration order. In latency-sensitive deployments, pre-pulling images or using leaner base images (such as `nginx:alpine` or `ubuntu:22.04-minimal`) reduces startup time.

* **Client-side dry-run does not validate against admission controllers.** `--dry-run=client` validates only against the local Kubernetes schema. It does not exercise server-side admission webhooks or policy engines (such as OPA Gatekeeper or Kyverno). In environments with admission control policies, `--dry-run=server` should be used for a more faithful pre-flight check.

---

## Reference Architecture

**Kubernetes Documentation:**
* [Pods: Multi-Container Pods](https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers)
* [emptyDir Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
* [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
* [kubectl apply](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply)
* [kubectl describe](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#describe)

**Images Used:**
* [nginx - Docker Hub](https://hub.docker.com/_/nginx)
* [ubuntu - Docker Hub](https://hub.docker.com/_/ubuntu)

  
<img width="1021" height="591" alt="image" src="https://github.com/user-attachments/assets/486d49b2-fe1c-4ef8-9a0c-c05370a52835" />
<img width="1023" height="857" alt="image" src="https://github.com/user-attachments/assets/932a0f78-45d4-4c22-8a25-510a44bf4a83" />
<img width="1024" height="852" alt="image" src="https://github.com/user-attachments/assets/09a4dfc2-9c27-4ec8-bf2a-bf6edae05523" />
<img width="1021" height="614" alt="image" src="https://github.com/user-attachments/assets/5637bd89-0e43-461c-9f7c-4f3484073e86" />
<img width="1024" height="653" alt="image" src="https://github.com/user-attachments/assets/530ae11b-f02b-4fb5-a3bf-a1a28a506806" />
<img width="1019" height="731" alt="image" src="https://github.com/user-attachments/assets/a8ac7483-ee7f-4e1a-a297-7298a22a1dfa" />
<img width="1020" height="859" alt="image" src="https://github.com/user-attachments/assets/5a366468-19c2-47ca-9d37-91707059e771" />
<img width="1022" height="499" alt="image" src="https://github.com/user-attachments/assets/accefa75-0352-4d24-a54c-e4533adda2f0" />
<img width="1017" height="846" alt="image" src="https://github.com/user-attachments/assets/4f27e50a-72fe-4f07-a6af-865750b8ce6d" />
<img width="1018" height="857" alt="image" src="https://github.com/user-attachments/assets/861ebc13-b2ad-4c1c-9a35-10be02cf0854" />
<img width="1024" height="855" alt="image" src="https://github.com/user-attachments/assets/2606534e-61ae-4e16-8a66-77a191da85a7" />
