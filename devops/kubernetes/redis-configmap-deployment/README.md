# Kubernetes Redis Deployment with ConfigMap Volume Integration

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.x-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-alpine-DC382D?style=flat-square&logo=redis&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-KodeKloud%20%2F%20Nautilus-0A66C2?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-28a745?style=flat-square)

---

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Architecture and Design Intent](#architecture-and-design-intent)
* [Prerequisites](#prerequisites)
* [Task Requirements](#task-requirements)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify Cluster Connectivity](#step-1-verify-cluster-connectivity)
  * [Step 2: Create the Redis ConfigMap](#step-2-create-the-redis-configmap)
  * [Step 3: Verify the ConfigMap](#step-3-verify-the-configmap)
  * [Step 4: Create the Redis Deployment Manifest](#step-4-create-the-redis-deployment-manifest)
  * [Step 5: Apply the Deployment](#step-5-apply-the-deployment)
  * [Step 6: Verify Deployment and Pod Health](#step-6-verify-deployment-and-pod-health)
  * [Step 7: Inspect Pod Configuration](#step-7-inspect-pod-configuration)
  * [Step 8: Validate ConfigMap Volume Mount](#step-8-validate-configmap-volume-mount)
  * [Step 9: Validate Data Volume Mount](#step-9-validate-data-volume-mount)
  * [Step 10: Confirm Port Exposure](#step-10-confirm-port-exposure)
* [Validation Summary](#validation-summary)
* [Errors and Resolutions](#errors-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)

---

## Overview

This project documents the end-to-end deployment of a Redis in-memory caching service on a Kubernetes cluster for the Nautilus application development team. The implementation provisions a `ConfigMap` for Redis runtime configuration, mounts it as a volume inside the container, and deploys a single-replica Redis pod using the `redis:alpine` image with resource constraints and dual volume mounts. The entire workflow was executed from a `jump-host` with pre-configured `kubectl` access to the target cluster.

---

## Problem Statement

The Nautilus application development team identified performance degradation in one of their application services deployed on Kubernetes. After root cause analysis across multiple contributing factors, the team determined that introducing an in-memory caching layer for the database service would significantly reduce latency and improve throughput. Redis was selected as the caching solution. The initial deployment targets a test environment on Kubernetes before promotion to production.

---

## Architecture and Design Intent

The solution implements a single-replica Redis deployment backed by two volume types to separate concerns clearly:

* **ConfigMap Volume (`redis-config`):** Injects the `redis-config` ConfigMap as a file at `/redis-master`, making Redis runtime parameters externally manageable without container rebuilds. This follows the Kubernetes-native configuration injection pattern.

* **EmptyDir Volume (`data`):** Provides ephemeral storage mounted at `/redis-master-data` for Redis data operations during the pod lifecycle. This is appropriate for a test environment where data persistence across pod restarts is not required.

* **Resource Requests:** A CPU request of `1` core is declared to allow the Kubernetes scheduler to make informed placement decisions and guarantee baseline compute allocation for the container.

```
┌────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                      │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │              redis-deployment (1 replica)          │   │
│  │                                                    │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │           redis-container (redis:alpine)     │  │   │
│  │  │                                              │  │   │
│  │  │  Port: 6379                                  │  │   │
│  │  │  CPU Request: 1 core                         │  │   │
│  │  │                                              │  │   │
│  │  │  /redis-master       <-- ConfigMap Volume    │  │   │
│  │  │  /redis-master-data  <-- EmptyDir Volume     │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────┘   │
│                                                            │
│  ConfigMap: my-redis-config                                │
│    redis-config: maxmemory 2mb                             │
└────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Kubernetes Cluster | Running and accessible |
| kubectl | Configured on `jump-host` with `default` context |
| Cluster Context | `default` targeting `https://127.0.0.1:6443` |
| Image Access | `redis:alpine` pullable from Docker Hub |
| Permissions | Sufficient RBAC to create ConfigMaps and Deployments in `default` namespace |

---

## Task Requirements

The following specifications were provided and must be met precisely:

1. Create a `ConfigMap` named `my-redis-config` with a key `redis-config` containing the value `maxmemory 2mb`
2. Create a `Deployment` named `redis-deployment` using the `redis:alpine` image
3. Container name must be `redis-container` with exactly `1` replica
4. The container must request `1` CPU
5. Mount two volumes:
   * An `emptyDir` volume named `data` at path `/redis-master-data`
   * A `ConfigMap` volume named `redis-config` at path `/redis-master`
6. Expose container port `6379`
7. The deployment must reach a ready state

---

## Implementation Guide

### Step 1: Verify Cluster Connectivity

Before any resource provisioning, confirm cluster availability and the active kubectl context to prevent misrouted operations.

```bash
kubectl cluster-info
kubectl config get-contexts
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

CURRENT   NAME      CLUSTER   AUTHINFO   NAMESPACE
*         default   default   default
```

The active context is `default`, targeting the correct cluster. All subsequent operations apply to the `default` namespace.

> **Screenshot:** `01-cluster-info-and-context.png`

---

### Step 2: Create the Redis ConfigMap

Define a `ConfigMap` named `my-redis-config` using a heredoc manifest. The `redis-config` key holds the Redis runtime directive `maxmemory 2mb`, which will be projected as a file inside the container.

```bash
cat <<EOF > my-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-redis-config
data:
  redis-config: |
    maxmemory 2mb
EOF
```

Apply the manifest:

```bash
kubectl apply -f my-redis-config.yaml
```

**Output:**

```
configmap/my-redis-config created
```

> **Screenshot:** `02-configmap-create-apply.png`

---

### Step 3: Verify the ConfigMap

Confirm the ConfigMap was created and its contents are accurate before proceeding to the deployment.

```bash
kubectl get configmap my-redis-config
```

**Output:**

```
NAME              DATA   AGE
my-redis-config   1      26s
```

Inspect the ConfigMap content in detail:

```bash
kubectl describe configmap my-redis-config
```

**Output:**

```
Name:         my-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb

BinaryData
====

Events:  <none>
```

The `redis-config` key is present with the correct value `maxmemory 2mb`. The ConfigMap is ready to be referenced in the deployment volume specification.

> **Screenshot:** `03-configmap-describe-verify.png`

---

### Step 4: Create the Redis Deployment Manifest

Define the deployment manifest with all required specifications: replica count, image, container name, resource requests, port, and both volume mounts.

```bash
cat <<EOF > redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis-container
        image: redis:alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: "1"
        volumeMounts:
        - name: data
          mountPath: /redis-master-data
        - name: redis-config
          mountPath: /redis-master
      volumes:
      - name: data
        emptyDir: {}
      - name: redis-config
        configMap:
          name: my-redis-config
EOF
```

**Key manifest decisions:**

* `replicas: 1` aligns with the test environment single-instance requirement
* `resources.requests.cpu: "1"` reserves one full CPU core for scheduler placement guarantees
* `volumes[].configMap.name` references `my-redis-config`, binding the previously created ConfigMap
* `emptyDir: {}` provides ephemeral pod-scoped storage without persistence requirements

> **Screenshot:** `04-deployment-manifest-created.png`

---

### Step 5: Apply the Deployment

Submit the deployment manifest to the cluster:

```bash
kubectl apply -f redis-deployment.yaml
```

**Output:**

```
deployment.apps/redis-deployment created
```

> **Screenshot:** `05-deployment-apply.png`

---

### Step 6: Verify Deployment and Pod Health

Confirm the deployment reaches the desired ready state and the pod is running:

```bash
kubectl get deployment redis-deployment
```

**Output:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
redis-deployment   1/1     1            1           30s
```

List pods using the `app=redis` label selector:

```bash
kubectl get pods -l app=redis
```

**Output:**

```
NAME                               READY   STATUS    RESTARTS   AGE
redis-deployment-c795495f4-nks9q   1/1     Running   0          92s
```

The deployment shows `1/1` ready and the pod status is `Running` with zero restarts, confirming a healthy rollout.

> **Screenshot:** `06-deployment-and-pod-status.png`

---

### Step 7: Inspect Pod Configuration

Retrieve the full pod spec to validate all configurations were applied correctly including volumes, mounts, resources, and runtime status:

```bash
kubectl describe pod $(kubectl get pods -l app=redis -o jsonpath='{.items[0].metadata.name}')
```

**Selected output highlights:**

```
Name:             redis-deployment-c795495f4-nks9q
Namespace:        default
Status:           Running
IP:               10.22.0.9

Containers:
  redis-container:
    Image:          redis:alpine
    Port:           6379/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 12 Apr 2026 00:20:07 +0000
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        1
    Mounts:
      /redis-master from redis-config (rw)
      /redis-master-data from data (rw)

Volumes:
  data:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
  redis-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      my-redis-config
    Optional:  false

QoS Class: Burstable

Events:
  Normal  Scheduled  Successfully assigned default/redis-deployment-c795495f4-nks9q to jump-host
  Normal  Pulling    Pulling image "redis:alpine"
  Normal  Pulled     Successfully pulled image "redis:alpine" in 2.069s
  Normal  Created    Created container: redis-container
  Normal  Started    Started container redis-container
```

All volumes, mounts, resource requests, and container configuration match the declared specifications exactly.

> **Screenshot:** `07-pod-describe-full.png`

---

### Step 8: Validate ConfigMap Volume Mount

Exec into the running pod to confirm the ConfigMap data was correctly projected as a file at the expected mount path:

```bash
kubectl exec -it $(kubectl get pods -l app=redis -o jsonpath='{.items[0].metadata.name}') -- cat /redis-master/redis-config
```

**Output:**

```
maxmemory 2mb
```

The `redis-config` file at `/redis-master/redis-config` contains the exact directive defined in the ConfigMap. The volume projection is functioning correctly.

> **Screenshot:** `08-configmap-volume-file-validation.png`

---

### Step 9: Validate Data Volume Mount

Verify the `emptyDir` volume is mounted and accessible at `/redis-master-data`:

```bash
kubectl exec -it $(kubectl get pods -l app=redis -o jsonpath='{.items[0].metadata.name}') -- ls /redis-master-data
```

**Output:**

```
(empty)
```

The directory exists and is empty, which is the expected state for a freshly mounted `emptyDir` volume with no Redis data written yet.

> **Screenshot:** `09-emptydir-volume-validation.png`

---

### Step 10: Confirm Port Exposure

Extract port configuration from the pod description to validate port `6379` is declared:

```bash
kubectl describe pod $(kubectl get pods -l app=redis -o jsonpath='{.items[0].metadata.name}') | grep -i port
```

**Output:**

```
    Port:           6379/TCP
    Host Port:      0/TCP
```

Container port `6379` is correctly exposed. `Host Port: 0/TCP` is expected behavior when no `hostPort` is explicitly bound, as the port is accessible within the cluster through the pod IP and through Services.

> **Screenshot:** `10-port-validation.png`

---

## Validation Summary

| Requirement | Expected | Observed | Status |
|---|---|---|---|
| ConfigMap name | `my-redis-config` | `my-redis-config` | Passed |
| ConfigMap key | `redis-config` | `redis-config` | Passed |
| ConfigMap value | `maxmemory 2mb` | `maxmemory 2mb` | Passed |
| Deployment name | `redis-deployment` | `redis-deployment` | Passed |
| Container name | `redis-container` | `redis-container` | Passed |
| Image | `redis:alpine` | `redis:alpine` | Passed |
| Replicas | `1` | `1/1 Ready` | Passed |
| CPU Request | `1` | `cpu: 1` | Passed |
| Port exposed | `6379` | `6379/TCP` | Passed |
| ConfigMap volume mount | `/redis-master` | `/redis-master` | Passed |
| EmptyDir volume mount | `/redis-master-data` | `/redis-master-data` | Passed |
| Pod status | `Running` | `Running`, 0 restarts | Passed |
| ConfigMap file content | `maxmemory 2mb` | `maxmemory 2mb` | Passed |

All 13 validation checkpoints passed without deviation.

---

## Errors and Resolutions

No errors were encountered during this implementation. The manifest was constructed correctly on the first attempt, and the deployment reached a ready state within 30 seconds of applying the manifest.

**Potential failure modes to be aware of for similar tasks:**

| Failure Scenario | Root Cause | Resolution |
|---|---|---|
| Pod stuck in `Pending` | Insufficient node CPU to satisfy `requests.cpu: 1` | Reduce CPU request or free cluster resources |
| `MountVolume.SetUp failed` | ConfigMap name mismatch between volume reference and actual ConfigMap | Ensure `volumes[].configMap.name` exactly matches the created ConfigMap name |
| Pod in `CrashLoopBackOff` | Redis config file contains invalid directives | Validate ConfigMap content with `kubectl describe configmap` before deploying |
| `ImagePullBackOff` | `redis:alpine` not accessible from node | Verify network connectivity to Docker Hub or use a local registry mirror |

---

## Best Practices Applied

* **Declarative manifests:** All Kubernetes resources were defined in YAML manifests and applied with `kubectl apply`, enabling version control, reproducibility, and auditability.

* **Label selectors:** The deployment uses `app: redis` labels consistently across `matchLabels`, pod template metadata, and `kubectl` queries with `-l app=redis`, establishing a clean and queryable resource identity.

* **ConfigMap decoupling:** Redis configuration is externalized into a ConfigMap rather than embedded in the container image or passed as environment variables. This allows runtime parameter changes without image rebuilds.

* **Volume type selection:** `emptyDir` is used intentionally for test environment ephemeral storage, avoiding unnecessary PersistentVolume overhead. In production, this would be replaced with a `PersistentVolumeClaim`.

* **Resource requests declared:** Specifying `requests.cpu` allows the Kubernetes scheduler to make informed node placement decisions and ensures the pod receives guaranteed baseline compute allocation under the `Burstable` QoS class.

* **jsonpath pod targeting:** Using `kubectl get pods -l app=redis -o jsonpath='{.items[0].metadata.name}'` inside exec and describe commands avoids hardcoded pod names, making commands reusable across pod restarts.

* **Pre-deployment verification:** The ConfigMap was verified with both `kubectl get` and `kubectl describe` before the deployment was applied, catching any configuration errors early in the workflow.

* **Post-deployment exec validation:** In-container file inspection with `cat` and `ls` confirmed the volume projection and mount behavior beyond what the control plane reports, validating actual runtime state.

---

## Lessons Learned

* **ConfigMap volume projection creates files per key:** When a ConfigMap is mounted as a volume, each key in the `data` section becomes a file at the mount path. The key `redis-config` becomes the file `/redis-master/redis-config`, not `/redis-master`. This means the Redis process would need to be configured to read from this specific file path. For production use, the Redis startup command should explicitly reference `--include /redis-master/redis-config` or the config file path argument.

* **EmptyDir is pod-scoped, not container-scoped:** Data written to an `emptyDir` volume persists across container restarts within the same pod. However, when the pod itself is deleted or evicted, all data is lost. For a caching layer this is often acceptable, but it should be explicitly documented in production handoff materials.

* **Burstable QoS class has implications:** Setting `requests.cpu` without a corresponding `limits.cpu` places the pod in the `Burstable` QoS class. Under memory pressure, `Burstable` pods are evicted after `BestEffort` pods but before `Guaranteed` pods. For production Redis deployments, setting both requests and limits for both CPU and memory is recommended to achieve `Guaranteed` QoS and avoid unexpected eviction.

* **Host Port 0 does not mean inaccessible:** A `Host Port: 0/TCP` output from `kubectl describe` indicates that no host port binding was configured, which is the correct and expected default. The container port remains reachable within the cluster via the pod IP. A Kubernetes `Service` of type `ClusterIP` or `NodePort` would be the correct mechanism for routing traffic to this pod in a production scenario.

* **Pre-flight context verification prevents namespace collisions:** Confirming the active kubectl context before applying any manifest is a discipline that prevents accidental resource creation in the wrong cluster or namespace, particularly in environments with multiple contexts configured.

---

*Documented by Arinze Edeh | Cloud and DevOps Engineer | GitHub: [@arinze-edeh](https://github.com/arinze-edeh)*


<img width="1033" height="651" alt="image" src="https://github.com/user-attachments/assets/590d3fc8-b7fe-4f37-9b14-56b1e2f5e5ad" />
<img width="1031" height="649" alt="image" src="https://github.com/user-attachments/assets/f61ad952-786f-4165-beba-4381b32ed1c2" />
<img width="1029" height="585" alt="image" src="https://github.com/user-attachments/assets/b43cdd98-e743-434b-87a1-77c89eedde18" />
<img width="1030" height="608" alt="image" src="https://github.com/user-attachments/assets/a3eee2c7-ac06-4c01-8dcf-0abdebdf5345" />
<img width="1031" height="738" alt="image" src="https://github.com/user-attachments/assets/adae63c1-f1de-4b7a-9e17-f3c991680d22" />
<img width="1032" height="844" alt="image" src="https://github.com/user-attachments/assets/68f042de-a65f-423d-b1ef-59042873922a" />
<img width="1033" height="859" alt="image" src="https://github.com/user-attachments/assets/7da1ec9f-b1be-4d56-8fed-1e56c2038289" />
<img width="1022" height="778" alt="image" src="https://github.com/user-attachments/assets/22625e4e-8b5a-4291-847f-c9dc0c8d9926" />
<img width="1025" height="824" alt="image" src="https://github.com/user-attachments/assets/bbd75e6b-1507-4186-86cd-be5be551329b" />
<img width="1026" height="858" alt="image" src="https://github.com/user-attachments/assets/0994abf2-c77c-4478-9fdd-374d23516ade" />
<img width="1032" height="858" alt="image" src="https://github.com/user-attachments/assets/f61377b1-e790-4b2a-9680-3843219d9da5" />
<img width="1034" height="857" alt="image" src="https://github.com/user-attachments/assets/8b463095-9277-456b-9fb6-1525da04b99f" />
<img width="1036" height="345" alt="image" src="https://github.com/user-attachments/assets/ce27adb4-9b13-441d-b096-47064a01a8fc" />
<img width="1039" height="306" alt="image" src="https://github.com/user-attachments/assets/39ecdd75-7a3f-439e-8f3a-268736cbcf53" />
<img width="1038" height="361" alt="image" src="https://github.com/user-attachments/assets/6374b6c0-1115-49cb-bea0-99b933fe4797" />

