# Kubernetes Multi-Tier Guestbook Application Deployment

## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Problem Statement](#problem-statement)
* [Solution Summary](#solution-summary)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify Cluster State](#step-1-verify-cluster-state)
  * [Step 2: Deploy Redis Master](#step-2-deploy-redis-master)
  * [Step 3: Expose Redis Master via Service](#step-3-expose-redis-master-via-service)
  * [Step 4: Deploy Redis Slave](#step-4-deploy-redis-slave)
  * [Step 5: Expose Redis Slave via Service](#step-5-expose-redis-slave-via-service)
  * [Step 6: Deploy Frontend Application](#step-6-deploy-frontend-application)
  * [Step 7: Expose Frontend via NodePort Service](#step-7-expose-frontend-via-nodeport-service)
  * [Step 8: Final Validation](#step-8-final-validation)
* [Resource Summary](#resource-summary)
* [Best Practices Applied](#best-practices-applied)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)

---

## Overview

This lab documents the end-to-end deployment of a multi-tier **Guestbook** application on a production Kubernetes cluster. The application is a classic PHP-Redis guestbook system that demonstrates enterprise patterns for deploying distributed, stateful, multi-component workloads on Kubernetes using declarative manifests applied via `kubectl`.

The implementation covers a full three-tier architecture: a Redis master for write operations, Redis replicas for read scaling, and a PHP frontend served via a NodePort service to allow external access through the cluster node.

| Attribute | Value |
|---|---|
| Platform | KodeKloud / Nautilus Kubernetes |
| Cluster Node | `jump-host` (control-plane) |
| Kubernetes Version | v1.34.1+k3s1 |
| Namespace | `default` |
| Deployment Method | Declarative YAML via `kubectl apply` |
| Frontend Access | NodePort `30009` |

---

## Architecture

```
                    External Traffic
                          |
                     NodePort :30009
                          |
              +-----------+-----------+
              |   frontend Service    |
              |   (NodePort, :80)     |
              +-----------+-----------+
                          |
          +---------------+---------------+
          |               |               |
    frontend Pod    frontend Pod    frontend Pod
    (php-redis)     (php-redis)     (php-redis)
          |               |               |
          +---------------+---------------+
                  |               |
    +-------------+    +----------+----------+
    |  redis-master |  |   redis-slave Svc   |
    |  Service:6379 |  |   (ClusterIP:6379)  |
    +------+--------+  +----------+----------+
           |                      |
    +------+-------+   +----------+----------+
    | redis-master  |   | redis-slave Pod x2  |
    | Pod (1 rep)   |   | (slave-redis-devops)|
    +---------------+   +---------------------+
```

**Data Flow:**
* Write requests from the PHP frontend are routed to `redis-master` (port 6379).
* Read requests are handled by `redis-slave` replicas, which synchronize from the master via DNS-based `GET_HOSTS_FROM=dns` environment variable resolution.
* All inter-service communication uses Kubernetes internal ClusterIP DNS resolution.

---

## Problem Statement

The Nautilus Application development team completed development of a PHP-based Guestbook application. The DevOps team is responsible for deploying this application onto the production Kubernetes cluster with the following requirements:

**Back-End Tier:**
* Redis Master Deployment with 1 replica, container name `master-redis-devops`, image `redis`, CPU request `100m`, Memory request `100Mi`, containerPort `6379`.
* Redis Master ClusterIP Service exposing port `6379`.
* Redis Slave Deployment with 2 replicas, container name `slave-redis-devops`, image `gcr.io/google_samples/gb-redisslave:v3`, CPU request `100m`, Memory request `100Mi`, env `GET_HOSTS_FROM=dns`, containerPort `6379`.
* Redis Slave ClusterIP Service exposing port `6379`.

**Front-End Tier:**
* Frontend Deployment with 3 replicas, container name `php-redis-devops`, image `gcr.io/google-samples/gb-frontend@sha256:a908df8486ff66f2c4daa0d3d8a2fa09846a1fc8efd65649c0109695c7c5cbff`, CPU request `100m`, Memory request `100Mi`, env `GET_HOSTS_FROM=dns`, containerPort `80`.
* Frontend NodePort Service with port `80`, nodePort `30009`.

---

## Solution Summary

All six Kubernetes resources (3 Deployments and 3 Services) were created using inline heredoc YAML manifests applied with `kubectl apply -f -`. Each deployment was validated with `kubectl rollout status` before proceeding to the next component, enforcing a controlled, sequential rollout that mirrors a production deployment pipeline.

---

## Prerequisites

* `kubectl` configured and connected to the target cluster (pre-configured on `jump-host`).
* Cluster access confirmed with `kubectl cluster-info`.
* Sufficient cluster resources to schedule 6 pods (1 redis-master + 2 redis-slave + 3 frontend).
* Network access to `gcr.io` and Docker Hub from cluster nodes for image pulls.

---

## Implementation Guide

### Step 1: Verify Cluster State

Before deploying any workloads, confirm the cluster is healthy and the correct namespaces are available.

```bash
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   41m   v1.34.1+k3s1

NAME              STATUS   AGE
default           Active   41m
kube-node-lease   Active   41m
kube-public       Active   41m
kube-system       Active   41m
```

> **Screenshot:** 

<img width="1038" height="564" alt="image" src="https://github.com/user-attachments/assets/83ee1d9a-d511-47d4-a61f-326bea5cad89" />

**Validation criteria:**
* Control plane is reachable.
* `jump-host` node shows `Ready` status.
* `default` namespace is active and available for workload deployment.

---

### Step 2: Deploy Redis Master

Create the Redis master Deployment. This single-replica deployment serves as the primary write node for all guestbook entries.

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master
  labels:
    app: guestbook
    tier: backend
    role: master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
      tier: backend
      role: master
  template:
    metadata:
      labels:
        app: guestbook
        tier: backend
        role: master
    spec:
      containers:
      - name: master-redis-devops
        image: redis
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
EOF
```

Confirm the rollout completes successfully before proceeding:

```bash
kubectl rollout status deployment/redis-master
kubectl get pods -l role=master
```

**Output:**

```
deployment "redis-master" successfully rolled out

NAME                            READY   STATUS    RESTARTS   AGE
redis-master-57996cc7cc-2zvwb   1/1     Running   0          42s
```

> **Screenshot:** 

<img width="1036" height="786" alt="image" src="https://github.com/user-attachments/assets/0c27fd88-c819-41f9-bb9d-e502dcdda3dc" />

---

### Step 3: Expose Redis Master via Service

Create a ClusterIP Service to allow the Redis slave and frontend pods to communicate with the Redis master using DNS-based service discovery.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: guestbook
    tier: backend
    role: master
spec:
  selector:
    app: guestbook
    tier: backend
    role: master
  ports:
  - port: 6379
    targetPort: 6379
EOF
```

Validate the Service and confirm endpoints are populated:

```bash
kubectl get svc redis-master
kubectl describe svc redis-master
```

**Output:**

```
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
redis-master   ClusterIP   10.43.244.32   <none>        6379/TCP   72s

Name:              redis-master
Selector:          app=guestbook,role=master,tier=backend
Type:              ClusterIP
IP:                10.43.244.32
Port:              <unset>  6379/TCP
TargetPort:        6379/TCP
Endpoints:         10.22.0.9:6379
```

> **Screenshot:** 

<img width="1032" height="816" alt="image" src="https://github.com/user-attachments/assets/d6849cdb-bc4f-4674-bfae-5625e766ea5a" />

The `Endpoints` field confirms the Service is correctly resolving to the running Redis master pod.

---

### Step 4: Deploy Redis Slave

Create the Redis slave Deployment with 2 replicas. Slave pods connect to the master using DNS-based host resolution through the `GET_HOSTS_FROM=dns` environment variable, which instructs the slave to resolve the master hostname via Kubernetes CoreDNS.

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-slave
  labels:
    app: guestbook
    tier: backend
    role: slave
spec:
  replicas: 2
  selector:
    matchLabels:
      app: guestbook
      tier: backend
      role: slave
  template:
    metadata:
      labels:
        app: guestbook
        tier: backend
        role: slave
    spec:
      containers:
      - name: slave-redis-devops
        image: gcr.io/google_samples/gb-redisslave:v3
        ports:
        - containerPort: 6379
        env:
        - name: GET_HOSTS_FROM
          value: "dns"
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
EOF
```

Confirm both replicas are running:

```bash
kubectl rollout status deployment/redis-slave
kubectl get pods -l role=slave
```

**Output:**

```
deployment "redis-slave" successfully rolled out

NAME                          READY   STATUS    RESTARTS   AGE
redis-slave-cbbd7b9cd-b9bmk   1/1     Running   0          40s
redis-slave-cbbd7b9cd-tc8xg   1/1     Running   0          40s
```

> **Screenshot:** 

<img width="1023" height="854" alt="image" src="https://github.com/user-attachments/assets/1e781915-201b-458d-be1e-1360717a3a93" />

---

### Step 5: Expose Redis Slave via Service

Create a ClusterIP Service for the Redis slave tier. The frontend reads from this service, distributing read load across the two slave replicas.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: guestbook
    tier: backend
    role: slave
spec:
  selector:
    app: guestbook
    tier: backend
    role: slave
  ports:
  - port: 6379
    targetPort: 6379
EOF
```

Validate the Service and confirm both slave pod endpoints are registered:

```bash
kubectl get svc redis-slave
kubectl describe svc redis-slave
```

**Output:**

```
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
redis-slave   ClusterIP   10.43.245.45   <none>        6379/TCP   33s

Name:              redis-slave
Selector:          app=guestbook,role=slave,tier=backend
Type:              ClusterIP
IP:                10.43.245.45
Endpoints:         10.22.0.10:6379,10.22.0.11:6379
```

> **Screenshot:** 

<img width="1032" height="821" alt="image" src="https://github.com/user-attachments/assets/0ad8f279-3b85-42f8-9b7f-75e89196f8bc" />

Two distinct endpoints confirm both slave pods are healthy and registered with the Service.

---

### Step 6: Deploy Frontend Application

Create the PHP-Redis frontend Deployment with 3 replicas. The frontend uses the `GET_HOSTS_FROM=dns` environment variable to dynamically resolve the Redis master and slave hostnames via CoreDNS at runtime. The image is pinned to a specific SHA256 digest for reproducibility and supply chain integrity.

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis-devops
        image: gcr.io/google-samples/gb-frontend@sha256:a908df8486ff66f2c4daa0d3d8a2fa09846a1fc8efd65649c0109695c7c5cbff
        ports:
        - containerPort: 80
        env:
        - name: GET_HOSTS_FROM
          value: "dns"
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
EOF
```

Confirm all 3 frontend replicas are running:

```bash
kubectl rollout status deployment/frontend
kubectl get pods -l tier=frontend
```

**Output:**

```
deployment "frontend" successfully rolled out

NAME                        READY   STATUS    RESTARTS   AGE
frontend-6dbf5877db-7whrl   1/1     Running   0          34s
frontend-6dbf5877db-r5wgc   1/1     Running   0          34s
frontend-6dbf5877db-vsp7m   1/1     Running   0          34s
```

> **Screenshot:** 

<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/af9c8c4c-a11a-4257-9fa3-8e7fbde1bfde" />

---

### Step 7: Expose Frontend via NodePort Service

Create a NodePort Service to expose the frontend application on port `30009` of the cluster node, making the guestbook accessible from outside the cluster.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  type: NodePort
  selector:
    app: guestbook
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30009
EOF
```

Validate the Service and confirm all three frontend pod endpoints are registered:

```bash
kubectl get svc frontend
kubectl describe svc frontend
```

**Output:**

```
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
frontend   NodePort   10.43.50.132   <none>        80:30009/TCP   29s

Name:              frontend
Selector:          app=guestbook,tier=frontend
Type:              NodePort
IP:                10.43.50.132
Port:              <unset>  80/TCP
TargetPort:        80/TCP
NodePort:          <unset>  30009/TCP
Endpoints:         10.22.0.12:80,10.22.0.13:80,10.22.0.14:80
```

> **Screenshot:** 

<img width="1031" height="859" alt="image" src="https://github.com/user-attachments/assets/4b8c9c3c-c7ac-42e3-b902-69fbe1c8a18c" />

Three endpoints confirm all frontend replicas are reachable through the NodePort.

---

### Step 8: Final Validation

Perform a comprehensive cluster-wide inspection to confirm all Deployments, Pods, and Services are in the expected state.

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get svc
```

**Deployments:**

```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
frontend       3/3     3            3           3m8s
redis-master   1/1     1            1           11m
redis-slave    2/2     2            2           6m51s
```

**Pods (with node and IP assignments):**

```
NAME                            READY   STATUS    RESTARTS   AGE     IP           NODE
frontend-6dbf5877db-7whrl       1/1     Running   0          3m40s   10.22.0.14   jump-host
frontend-6dbf5877db-r5wgc       1/1     Running   0          3m40s   10.22.0.12   jump-host
frontend-6dbf5877db-vsp7m       1/1     Running   0          3m40s   10.22.0.13   jump-host
redis-master-57996cc7cc-2zvwb   1/1     Running   0          11m     10.22.0.9    jump-host
redis-slave-cbbd7b9cd-b9bmk     1/1     Running   0          7m23s   10.22.0.11   jump-host
redis-slave-cbbd7b9cd-tc8xg     1/1     Running   0          7m23s   10.22.0.10   jump-host
```

**Services:**

```
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
frontend       NodePort    10.43.50.132   <none>        80:30009/TCP   2m36s
kubernetes     ClusterIP   10.43.0.1      <none>        443/TCP        55m
redis-master   ClusterIP   10.43.244.32   <none>        6379/TCP       10m
redis-slave    ClusterIP   10.43.245.45   <none>        6379/TCP       6m10s
```

> **Screenshot:** 

<img width="1029" height="860" alt="image" src="https://github.com/user-attachments/assets/994831a5-a3bc-4129-ae35-add41e9fcfed" />

All 6 pods are in `Running` state with `0` restarts. All 3 deployments report full availability. The guestbook application is accessible via the node's external IP on port `30009`.

---

## Resource Summary

| Resource | Kind | Replicas | Image | Port |
|---|---|---|---|---|
| `redis-master` | Deployment | 1 | `redis` | 6379 |
| `redis-master` | Service (ClusterIP) | N/A | N/A | 6379 |
| `redis-slave` | Deployment | 2 | `gcr.io/google_samples/gb-redisslave:v3` | 6379 |
| `redis-slave` | Service (ClusterIP) | N/A | N/A | 6379 |
| `frontend` | Deployment | 3 | `gcr.io/google-samples/gb-frontend@sha256:...` | 80 |
| `frontend` | Service (NodePort) | N/A | N/A | 80:30009 |

---

## Best Practices Applied

* **Declarative configuration:** All resources are applied via YAML manifests using `kubectl apply`, enabling idempotent, version-controlled deployments that can be re-applied safely.

* **Sequential rollout validation:** Each deployment was confirmed with `kubectl rollout status` before proceeding to the next component. This prevents downstream failures caused by unready dependencies (for example, the slave Service being created before its pods are healthy).

* **Resource requests on all containers:** CPU and memory requests are specified for every container (`100m` CPU, `100Mi` memory). This allows the Kubernetes scheduler to make informed placement decisions and prevents resource contention in shared cluster environments.

* **Label-based selector consistency:** All Deployments use `matchLabels` that align precisely with their corresponding Service selectors. Mismatched labels are one of the most common causes of Pods being unreachable through their Service.

* **DNS-based host resolution:** The `GET_HOSTS_FROM=dns` environment variable delegates hostname resolution to CoreDNS rather than hardcoding IP addresses. This is the correct pattern for Kubernetes-native inter-service communication and supports pod rescheduling without configuration changes.

* **Image digest pinning for the frontend:** The frontend container references a specific SHA256 image digest rather than a mutable tag. This eliminates the risk of silent image drift between deployments and satisfies supply chain security requirements.

* **Endpoint verification post-service creation:** After each Service was applied, `kubectl describe svc` was used to confirm the `Endpoints` field was populated. An empty Endpoints field indicates a label selector mismatch and must be caught before the dependent tier is deployed.

* **NodePort assignment for external access:** Using a fixed NodePort (`30009`) rather than a randomly assigned port ensures that the application URL is deterministic and can be shared with stakeholders without additional discovery steps.

---

## Errors and Resolutions

No errors were encountered during this implementation. All resources applied cleanly and all pods reached `Running` state without restarts. The following known failure modes were proactively avoided through careful manifest review before applying each resource.

| Potential Failure | Root Cause | Prevention Applied |
|---|---|---|
| Service with empty `Endpoints` | Label selector mismatch between Service and Deployment pod template | Verified `selector` labels matched `template.metadata.labels` before applying |
| Pod stuck in `ImagePullBackOff` | Incorrect or inaccessible image reference | Used well-known public images; pinned frontend to SHA256 digest |
| Slave pods unable to connect to master | `GET_HOSTS_FROM` env var missing or misconfigured | Confirmed `value: "dns"` set correctly on both slave and frontend containers |
| NodePort conflict | Port `30009` already in use by another Service | Confirmed no existing Services on that port before applying |
| Redis slave not replicating | Master Service created after slave deployment | Ensured master Service was fully ready before deploying slave tier |

---

## Lessons Learned

* **Order of resource creation matters in multi-tier deployments.** Creating the Redis master Service before the slave Deployment ensures the slave's DNS resolution of `redis-master` succeeds on startup. Reversing this order can cause slave pods to fail initialization if the DNS record does not yet exist.

* **Always validate `Endpoints` after creating a Service.** The absence of endpoints is not surfaced as an error by Kubernetes; it silently results in connection refused errors at the application layer. Making `kubectl describe svc` part of the deployment checklist catches this immediately.

* **Image digest pinning is a production requirement, not an optimization.** Mutable tags like `latest` or `v1` can silently pull different images across deployments, introducing unreproducible behavior. The frontend SHA256 digest used here is the correct pattern for any workload where image consistency is required.

* **Resource requests enable predictable scheduling.** Without resource requests, the scheduler may co-locate pods in ways that cause CPU throttling or OOM conditions under load. Even modest requests like `100m` CPU and `100Mi` memory give the scheduler the data it needs to make better decisions.

* **`GET_HOSTS_FROM=dns` is the Kubernetes-idiomatic approach for service discovery.** Hardcoding environment-based hostnames (the alternative pattern in the Redis slave image) ties configuration to specific cluster environments. DNS-based resolution via CoreDNS adapts automatically as pods are rescheduled.

* **`kubectl rollout status` provides a synchronous gate for deployment pipelines.** In CI/CD contexts, this command blocks until the rollout is complete or times out, making it the correct mechanism for enforcing sequenced multi-tier deployments in automated pipelines.

---
