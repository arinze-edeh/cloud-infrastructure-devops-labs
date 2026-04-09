# Kubernetes Secrets Management: Secure License Key Storage and Pod Mounting

> **Project:** Nautilus DevOps Platform | **Cluster:** Kubernetes (kubectl via jump-host) | **Namespace:** `default`

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify the Secret Key File](#step-1-verify-the-secret-key-file)
  - [Step 2: Create a Kubernetes Generic Secret](#step-2-create-a-kubernetes-generic-secret)
  - [Step 3: Verify the Secret Object](#step-3-verify-the-secret-object)
  - [Step 4: Author the Pod Manifest](#step-4-author-the-pod-manifest)
  - [Step 5: Deploy the Pod](#step-5-deploy-the-pod)
  - [Step 6: Confirm Pod is Running](#step-6-confirm-pod-is-running)
  - [Step 7: Exec into the Container and Validate the Secret](#step-7-exec-into-the-container-and-validate-the-secret)
- [File Reference](#file-reference)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)

---

## Overview

This document details the end-to-end implementation of a Kubernetes Secrets workflow for the Nautilus DevOps team. The objective is to securely store a licence/password key file as a Kubernetes `Secret` object and consume that secret within a running container via a mounted volume. The entire workflow is executed from a pre-configured `jump-host` using `kubectl`.

---

## Problem Statement

The Nautilus DevOps team is deploying licence-based tooling into a Kubernetes cluster. Licence keys must **not** be baked into container images or passed as plain-text environment variables. Instead, they must be stored using Kubernetes-native secret management so that:

* Sensitive data is encoded and access-controlled within the cluster.
* Secrets can be injected into pods at runtime without rebuilding images.
* The operational surface for credential leakage is minimised.

---

## Solution Architecture

```
jump-host
  |
  |-- /opt/news.txt  (plain-text licence key: "5ecur3")
  |
  kubectl create secret generic news --from-file=news.txt=/opt/news.txt
  |
  Kubernetes API Server
       |
       |-- Secret: news  (Opaque, base64-encoded)
       |
       Pod: secret-nautilus
         |-- Container: secret-container-nautilus (fedora:latest)
               |-- Volume Mount: /opt/cluster  <-- Secret "news" projected here
                     |-- /opt/cluster/news.txt  (decrypted at runtime by kubelet)
```

**Secret consumption method:** Volume mount (preferred over `env` injection for file-based secrets, as it supports automatic rotation without pod restarts).

---

## Prerequisites

| Requirement | Detail |
|---|---|
| `kubectl` | Configured on `jump-host` and pointing to the target cluster |
| Cluster access | `default` namespace with permissions to create Secrets and Pods |
| Source file | `/opt/news.txt` present and readable on `jump-host` |
| Container registry access | Cluster nodes must be able to pull `fedora:latest` |

---

## Implementation Guide

### Step 1: Verify the Secret Key File

Before creating the Kubernetes Secret, confirm that the source file exists and contains the expected licence key.

```bash
ls -la /opt/news.txt
cat /opt/news.txt
```

**Observed output:**

```
-rw-r--r--    1 root     root             7 Apr  9 00:57 /opt/news.txt
5ecur3
```

> **Screenshot:**

<img width="1078" height="486" alt="image" src="https://github.com/user-attachments/assets/609a672a-25d5-453f-864d-16109683a846" />

>Terminal output showing file metadata and licence key content.


The file is 7 bytes (6 characters + newline), owned by `root`, and world-readable. This confirms it is accessible for ingestion by `kubectl`.

---

### Step 2: Create a Kubernetes Generic Secret

Use `kubectl create secret generic` with the `--from-file` flag to create a Secret named `news` directly from the file on disk. The key name inside the Secret is explicitly set to `news.txt` to mirror the original filename.

```bash
kubectl create secret generic news \
  --from-file=news.txt=/opt/news.txt
```

**Observed output:**

```
secret/news created
```

> **Screenshot:**

<img width="1081" height="541" alt="image" src="https://github.com/user-attachments/assets/f0ee3e8c-7ee0-4603-8143-032c998f889e" />

>Terminal output confirming successful secret creation.

Using `--from-file=<key>=<path>` syntax ensures the key name within the Secret's `data` map is exactly `news.txt`, which determines the filename that will appear in the mounted volume.

---

### Step 3: Verify the Secret Object

Confirm the Secret was created correctly in the `default` namespace.

```bash
kubectl get secret news
```

```
NAME   TYPE     DATA   AGE
news   Opaque   1      32s
```

```bash
kubectl describe secret news
```

```
Name:         news
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
news.txt:  7 bytes
```

> **Screenshot:** 

<img width="1083" height="652" alt="image" src="https://github.com/user-attachments/assets/eba58b37-94ce-411d-9873-8c53dda4ec5f" />

>Output of `kubectl describe secret news` confirming the data key and byte size.

The Secret type is `Opaque`, which is correct for arbitrary user-defined data. The `DATA` section shows one key (`news.txt`) containing 7 bytes, matching the source file exactly. Note that `kubectl describe` intentionally redacts the actual value; the raw base64 value is only accessible via `kubectl get secret news -o jsonpath='{.data.news\.txt}'`.

---

### Step 4: Author the Pod Manifest

Write the Pod specification to `secret-nautilus.yaml`. The spec defines a single container that runs indefinitely using the `sleep infinity` command and mounts the `news` Secret at `/opt/cluster` inside the container.

```bash
cat <<EOF > secret-nautilus.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-nautilus
spec:
  containers:
  - name: secret-container-nautilus
    image: fedora:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: secret-volume
      mountPath: /opt/cluster
  volumes:
  - name: secret-volume
    secret:
      secretName: news
EOF
```

**Resulting manifest (`secret-nautilus.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-nautilus
spec:
  containers:
  - name: secret-container-nautilus
    image: fedora:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: secret-volume
      mountPath: /opt/cluster
  volumes:
  - name: secret-volume
    secret:
      secretName: news
```

> **Screenshot:** 

<img width="1082" height="771" alt="image" src="https://github.com/user-attachments/assets/ed76ddd3-79fe-4f33-a052-82f2de4dcb2a" />

>Terminal output showing the heredoc command and the resulting YAML content.

**Key manifest decisions:**

* `image: fedora:latest` with explicit `:latest` tag, as required by the task specification.
* `command: ["sleep", "infinity"]` keeps the container in a perpetual running state, enabling exec access for validation.
* The `volumes[].secret.secretName` value `news` must exactly match the Secret object name created in Step 2.
* `mountPath: /opt/cluster` is the target directory inside the container where each Secret key becomes a file.

---

### Step 5: Deploy the Pod

Apply the manifest to the cluster.

```bash
kubectl apply -f secret-nautilus.yaml
```

**Observed output:**

```
pod/secret-nautilus created
```

> **Screenshot:** 

<img width="1089" height="737" alt="image" src="https://github.com/user-attachments/assets/df5a392d-7b9e-4bdf-a1d6-6c952b4bc266" />

>Terminal output confirming pod creation.

---

### Step 6: Confirm Pod is Running

Watch the pod status until it transitions to `Running`.

```bash
kubectl get pod secret-nautilus -w
```

**Observed output:**

```
NAME              READY   STATUS    RESTARTS   AGE
secret-nautilus   1/1     Running   0          30s
```

> **Screenshot:** `step6-pod-running.png` - Terminal output showing pod in `1/1 Running` state with zero restarts.

The `1/1` readiness ratio confirms that the single container within the pod is healthy. The `0` restart count confirms a clean startup with no crash loops.

---

### Step 7: Exec into the Container and Validate the Secret

Exec into the running container using the specific container name and read the mounted secret file.

```bash
kubectl exec -it secret-nautilus \
  -c secret-container-nautilus \
  -- cat /opt/cluster/news.txt
```

**Observed output:**

```
5ecur3
```

> **Screenshot:** `step7-validate-secret.png` - Terminal output confirming the licence key value is accessible at the expected path inside the container.

The value `5ecur3` matches the original content of `/opt/news.txt` on the jump-host, confirming that:

1. The Secret was correctly encoded during creation.
2. The kubelet correctly decoded and projected the Secret data into the container's filesystem at `/opt/cluster/news.txt`.
3. The volume mount configuration in the Pod spec is correct.

---

## File Reference

| File | Location | Purpose |
|---|---|---|
| `news.txt` | `/opt/news.txt` on jump-host | Source licence key file |
| `secret-nautilus.yaml` | `~/secret-nautilus.yaml` on jump-host | Pod manifest for deployment |

---

## Best Practices Applied

* **File-based secret mounting over environment variables:** Mounting secrets as volumes is preferred for file-based credentials. It supports secret rotation (Kubernetes can update the projected file without restarting the pod when `subPath` is not used) and avoids exposing secrets in the process environment table.

* **Explicit key naming with `--from-file=<key>=<path>`:** Using `--from-file=news.txt=/opt/news.txt` instead of `--from-file=/opt/news.txt` explicitly controls the key name in the Secret's `data` map, ensuring the projected filename in the container is deterministic and matches the original filename.

* **Dedicated volume reference by name:** The `volumeMounts[].name` and `volumes[].name` values use the logical name `secret-volume`, decoupling the mount configuration from the Secret name. This makes it straightforward to swap in a different Secret without rewriting the volume mount block.

* **`sleep infinity` for long-lived utility containers:** Using `sleep infinity` as the container command is a clean, zero-dependency method to keep a container alive for inspection or validation tasks without installing any additional tooling.

* **Targeted exec with `-c` flag:** Specifying `-c secret-container-nautilus` in the `kubectl exec` command is a best practice even for single-container pods. It is mandatory for multi-container pods and makes scripts forward-compatible.

* **Namespace awareness:** All operations were performed within the `default` namespace as confirmed by `kubectl describe secret`. In production, secrets and workloads should be placed in a dedicated namespace with appropriate RBAC policies limiting access to the Secret to only the service accounts that require it.

---

## Lessons Learned

* **`kubectl describe` redacts secret values by design.** The `Data` section shows key names and byte sizes but never the actual values. Use `kubectl get secret <name> -o jsonpath='{.data.<key>}'` and pipe to `base64 --decode` if you need to inspect a stored value during debugging. Never print decoded secrets in shared terminal sessions.

* **Image pull latency for `latest` tags.** The `fedora:latest` image may not be cached on cluster nodes, causing the pod to stay in `ContainerCreating` status for a period. Using `kubectl get pod -w` to watch the transition is more reliable than polling with a fixed sleep in CI scripts.

* **Secret key names become filenames.** The key name in the Secret's `data` map directly determines the filename created inside the mounted directory. Naming the key `news.txt` (rather than e.g. `news`) ensures the file appears as `/opt/cluster/news.txt` inside the container, which is important for applications that expect a specific filename.

* **`apply` vs `create` for idempotency.** Using `kubectl apply -f` for the Pod manifest (rather than `kubectl create -f`) is a habit that pays off in iterative workflows. For immutable objects like Pods, `apply` will report a conflict on re-run rather than creating a duplicate, making errors easier to detect.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `Error from server (NotFound): secrets "news" not found` during pod scheduling | Secret was not created or was created in a different namespace | Verify with `kubectl get secret news -n <namespace>` and ensure the pod manifest specifies the same namespace |
| Pod stuck in `ContainerCreating` | Image pull in progress or image pull error | Check `kubectl describe pod secret-nautilus` for `Events` section; inspect `ImagePullBackOff` messages |
| `/opt/cluster/news.txt: No such file or directory` inside container | `secretName` in the volume spec does not match the actual Secret name | Cross-check `volumes[].secret.secretName` in the manifest against `kubectl get secret` output |
| Exec command returns empty output | Secret was created with an empty source file | Verify `cat /opt/news.txt` on the jump-host and recreate the secret if the file was empty |
| `Error: container not found` on exec | Container name mismatch | Use `kubectl get pod secret-nautilus -o jsonpath='{.spec.containers[*].name}'` to confirm the exact container name |







<img width="1079" height="859" alt="image" src="https://github.com/user-attachments/assets/8d46c966-a4fd-48a6-b583-2ae7f1230f18" />
<img width="1083" height="529" alt="image" src="https://github.com/user-attachments/assets/492f6c11-c269-4482-83be-4cbce88f4528" />


