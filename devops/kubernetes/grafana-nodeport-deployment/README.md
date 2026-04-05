# Grafana Deployment on Kubernetes with NodePort Exposure

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Environment Verification](#environment-verification)
- [Implementation](#implementation)
  - [Step 1: Author the Grafana Deployment Manifest](#step-1-author-the-grafana-deployment-manifest)
  - [Step 2: Author the Grafana Service Manifest](#step-2-author-the-grafana-service-manifest)
  - [Step 3: Apply the Deployment Manifest](#step-3-apply-the-deployment-manifest)
  - [Step 4: Apply the Service Manifest](#step-4-apply-the-service-manifest)
  - [Step 5: Validate Deployment Rollout](#step-5-validate-deployment-rollout)
  - [Step 6: Verify Deployment and Pod Health](#step-6-verify-deployment-and-pod-health)
  - [Step 7: Verify Service Configuration](#step-7-verify-service-configuration)
  - [Step 8: Retrieve Node IP for Connectivity Test](#step-8-retrieve-node-ip-for-connectivity-test)
  - [Step 9: Confirm Grafana Login Page Reachability](#step-9-confirm-grafana-login-page-reachability)
- [Validation Summary](#validation-summary)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This project documents the end-to-end deployment of a Grafana monitoring instance on a live Kubernetes cluster using declarative YAML manifests. The objective was to provision a `Deployment` and a `NodePort` `Service` to expose the Grafana login interface on a fixed external port, satisfying an operational requirement raised by the Nautilus DevOps team.

The implementation follows a manifest-first, apply-and-validate workflow consistent with production Kubernetes engineering practices.

---

## Problem Statement

The Nautilus DevOps team required a centralized analytics and monitoring tool deployable on the existing Kubernetes cluster. The constraints were explicit:

* The `Deployment` must be named `grafana-deployment-nautilus`
* A `NodePort` type `Service` must expose the application on port `32000`
* No internal Grafana configuration changes were required; the acceptance criterion was a reachable Grafana login page

---

## Solution Architecture

| Component | Kind | Name | Image | Port |
|---|---|---|---|---|
| Application workload | `Deployment` | `grafana-deployment-nautilus` | `grafana/grafana:latest` | `3000` (container) |
| External exposure | `Service` | `grafana-service-nautilus` | N/A | `32000` (node) -> `3000` (pod) |

**Traffic flow:**

```
External Client --> Node IP:32000 --> Service (NodePort) --> Pod:3000 (Grafana)
```

---

## Prerequisites

* `kubectl` configured and authenticated against the target cluster
* Network access to the node IP on the target `nodePort`
* Cluster with at least one schedulable node in `Ready` state

---

## Environment Verification

Before applying any manifests, the cluster state was confirmed to be clean and operational.

```bash
kubectl cluster-info
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

```bash
kubectl get nodes
```

**Output:**

```
NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   96m   v1.34.1+k3s1
```

```bash
kubectl get deployments
kubectl get services
kubectl get pods
```

**Output:** No existing workloads in the `default` namespace, confirming a clean deployment surface.

> Screenshot: Clean cluster state showing no existing deployments, services, or pods in the default namespace

<img width="1029" height="534" alt="image" src="https://github.com/user-attachments/assets/82f42cdb-9d2e-4fab-96fb-9f98f93ebe7f" />

---

## Implementation

### Step 1: Author the Grafana Deployment Manifest

A `Deployment` manifest was written inline using a heredoc and written to `grafana-deployment.yaml`. The specification uses a `matchLabels` selector aligned with the pod template label, one replica, and exposes container port `3000` which is Grafana's default HTTP listener.

```bash
cat <<EOF > grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment-nautilus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
EOF
```

The manifest was verified before application:

```bash
cat grafana-deployment.yaml
```

> Screenshot: Terminal output confirming the contents of `grafana-deployment.yaml`

<img width="1033" height="862" alt="image" src="https://github.com/user-attachments/assets/5c4fa96c-164c-487e-ac98-456e70e00f95" />

---

### Step 2: Author the Grafana Service Manifest

A `NodePort` `Service` manifest was written to `grafana-service.yaml`. The `selector` matches the `app: grafana` label from the `Deployment` pod template, ensuring correct endpoint registration. Port `32000` was specified as the `nodePort` per the task requirement, forwarding to the pod's port `3000`.

```bash
cat <<EOF > grafana-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-service-nautilus
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 32000
EOF
```

The manifest was verified before application:

```bash
cat grafana-service.yaml
```

> Screenshot: Terminal output confirming the contents of `grafana-service.yaml`

<img width="1037" height="556" alt="image" src="https://github.com/user-attachments/assets/ef4bce03-92ea-42ce-bf37-bc5635c5011f" />

---

### Step 3: Apply the Deployment Manifest

```bash
kubectl apply -f grafana-deployment.yaml
```

**Output:**

```
deployment.apps/grafana-deployment-nautilus created
```

---

### Step 4: Apply the Service Manifest

```bash
kubectl apply -f grafana-service.yaml
```

**Output:**

```
service/grafana-service-nautilus created
```

---

### Step 5: Validate Deployment Rollout

The rollout was confirmed to have completed successfully before proceeding:

```bash
kubectl rollout status deployment/grafana-deployment-nautilus
```

**Output:**

```
deployment "grafana-deployment-nautilus" successfully rolled out
```

> Screenshot: Rollout status confirming all replicas have reached the desired state

<img width="1034" height="400" alt="image" src="https://github.com/user-attachments/assets/b8403584-9b7e-4cc8-b3fe-14226a993e48" />

---

### Step 6: Verify Deployment and Pod Health

```bash
kubectl get deployment grafana-deployment-nautilus
```

**Output:**

```
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
grafana-deployment-nautilus   1/1     1            1           2m58s
```

```bash
kubectl get pods -l app=grafana
```

**Output:**

```
NAME                                          READY   STATUS    RESTARTS   AGE
grafana-deployment-nautilus-567857c7d-tnfb4   1/1     Running   0          3m11s
```

> Screenshot: Pod in `Running` state with `1/1` containers ready and zero restarts

<img width="1035" height="577" alt="image" src="https://github.com/user-attachments/assets/ad5f5a5d-e9ce-4ac5-bc40-d094b833e626" />

---

### Step 7: Verify Service Configuration

```bash
kubectl get service grafana-service-nautilus
```

**Output:**

```
NAME                       TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
grafana-service-nautilus   NodePort   10.43.3.156   <none>        3000:32000/TCP   2m41s
```

The `PORT(S)` column confirms `3000:32000/TCP`, meaning traffic arriving at the node on port `32000` is forwarded to the pod on port `3000`.

> Screenshot: Service listing showing correct NodePort mapping `3000:32000/TCP`

---

### Step 8: Retrieve Node IP for Connectivity Test

```bash
kubectl get nodes -o wide
```

**Output:**

```
NAME        STATUS   ROLES           AGE    VERSION        INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
jump-host   Ready    control-plane   107m   v1.34.1+k3s1   10.244.234.211   <none>        Alpine Linux v3.16   6.8.0-90-generic   containerd://1.6.8
```

The node's `INTERNAL-IP` is `10.244.234.211`, used as the target for the connectivity test.

---

### Step 9: Confirm Grafana Login Page Reachability

```bash
curl -I http://10.244.234.211:32000
```

**Output:**

```
HTTP/1.1 302 Found
Cache-Control: no-store
Content-Type: text/html; charset=utf-8
Location: /login
X-Content-Type-Options: nosniff
X-Frame-Options: deny
X-Xss-Protection: 1; mode=block
Date: Sun, 05 Apr 2026 00:58:08 GMT
```

The `302 Found` redirect to `/login` confirms that the Grafana application is running and its login page is reachable at `http://10.244.234.211:32000`. The response headers also reflect correct security defaults set by Grafana out of the box.

> Screenshot: `curl -I` response showing `HTTP/1.1 302 Found` with `Location: /login`

---

## Validation Summary

| Check | Command | Expected Result | Status |
|---|---|---|---|
| Cluster operational | `kubectl cluster-info` | Control plane URL returned | Passed |
| Node ready | `kubectl get nodes` | `Ready` status | Passed |
| Deployment rollout | `kubectl rollout status` | `successfully rolled out` | Passed |
| Pod running | `kubectl get pods -l app=grafana` | `Running`, `1/1`, `0` restarts | Passed |
| Service port mapping | `kubectl get service` | `3000:32000/TCP` | Passed |
| HTTP reachability | `curl -I http://10.244.234.211:32000` | `302 Found`, `Location: /login` | Passed |

---

## Best Practices Applied

* **Declarative manifests over imperative commands:** Both the `Deployment` and `Service` were authored as YAML files rather than using `kubectl run` or `kubectl expose`. This enables version control, peer review, and reproducible deployments across environments.

* **Label-selector alignment:** The `selector.matchLabels` in the `Deployment` and the `selector` in the `Service` both reference `app: grafana`, ensuring Kubernetes endpoint registration is consistent and unambiguous.

* **Rollout status verification before connectivity test:** `kubectl rollout status` was used to confirm replica convergence before attempting the `curl` probe, preventing false-negative results from hitting the service before the pod was ready.

* **Using `-l` label filtering for pod inspection:** `kubectl get pods -l app=grafana` was used rather than listing all pods, which is more reliable at scale and demonstrates understanding of label-based resource selection.

* **HTTP header inspection with `curl -I`:** Rather than attempting a full browser-based login test, a lightweight `curl -I` probe was used to confirm HTTP reachability. The `302` redirect to `/login` is the definitive indicator that Grafana is running correctly without requiring any UI interaction.

* **Manifest verification before apply:** Both YAML files were printed with `cat` immediately after creation to confirm heredoc fidelity before `kubectl apply`, catching any shell escaping issues before they reached the cluster.

---

## Lessons Learned

* **`302 Found` is the correct expected response for Grafana root path:** Grafana redirects unauthenticated requests from `/` to `/login`. A test expecting `200 OK` at the root would fail; `302 Found` with `Location: /login` is the definitive proof of a healthy deployment. This is an important distinction when scripting health checks.

* **`INTERNAL-IP` is required when `EXTERNAL-IP` is `<none>`:** In environments without a cloud load balancer or external IP assignment (such as this k3s single-node setup), `kubectl get nodes -o wide` is the correct method to retrieve the routable node IP. Attempting to use `localhost` or `127.0.0.1` from outside the node would fail.

* **k3s on Alpine Linux with `containerd` behaves identically to production Kubernetes for workload testing:** The runtime and distribution differences are transparent at the `kubectl` and manifest layer. Manifests authored here are fully portable to any conformant Kubernetes distribution.

* **`nodePort` values below `30000` or above `32767` are rejected by the Kubernetes API by default:** The assigned port `32000` falls within the valid default `NodePort` range (`30000-32767`). Using a value outside this range without custom API server configuration would result in a validation error at apply time.

---

## References

* [Grafana Docker Image (Docker Hub)](https://hub.docker.com/r/grafana/grafana)
* [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [Kubernetes Services (NodePort)](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
* [kubectl rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)

<img width="1034" height="785" alt="image" src="https://github.com/user-attachments/assets/4a8be44a-3802-465d-9c70-a9c28c634e1d" />

<img width="1032" height="684" alt="image" src="https://github.com/user-attachments/assets/aa303f3e-4e1a-4d59-9544-10ac9ee4a157" />

<img width="1031" height="595" alt="image" src="https://github.com/user-attachments/assets/94540ee9-0b34-4603-9775-e94f2aad44b3" />
<img width="1035" height="368" alt="image" src="https://github.com/user-attachments/assets/e56a3a2f-a812-48ec-9dbe-0fd2b61577ba" />


<img width="1199" height="626" alt="image" src="https://github.com/user-attachments/assets/ab3597ac-61ba-4dff-b23d-7256b8a78ed4" />
<img width="1199" height="814" alt="image" src="https://github.com/user-attachments/assets/3e2934e6-c8ee-462f-b525-77e8509d2820" />
<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/25f4ff4a-02ae-4226-8702-0598945d40de" />
