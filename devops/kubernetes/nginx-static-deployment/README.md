# Kubernetes Nginx Deployment with NodePort Service

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34.1-326CE5?style=flat&logo=kubernetes&logoColor=white)
![K3s](https://img.shields.io/badge/Distribution-K3s-FFC61C?style=flat&logo=k3s&logoColor=black)
![Nginx](https://img.shields.io/badge/Nginx-latest-009639?style=flat&logo=nginx&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen?style=flat)

---

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Solution Architecture](#solution-architecture)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1 - Verify Cluster Connectivity](#step-1---verify-cluster-connectivity)
  * [Step 2 - Create the Nginx Deployment](#step-2---create-the-nginx-deployment)
  * [Step 3 - Verify Deployment Rollout](#step-3---verify-deployment-rollout)
  * [Step 4 - Create the NodePort Service](#step-4---create-the-nodeport-service)
  * [Step 5 - Verify Service Provisioning](#step-5---verify-service-provisioning)
  * [Step 6 - Validate Full Stack Configuration](#step-6---validate-full-stack-configuration)
* [Manifest Reference](#manifest-reference)
* [Verification Summary](#verification-summary)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Troubleshooting](#troubleshooting)

---

## Overview

This repository documents the end-to-end deployment of a highly available, scalable static website on a Kubernetes cluster using Nginx. The implementation provisions a `Deployment` with multiple replicas to ensure fault tolerance and a `NodePort` type `Service` to expose the application externally on a fixed port.

All steps were executed via `kubectl` on a K3s-backed single-node cluster using the `jump-host` as the control plane entry point.

---

## Problem Statement

The Nautilus team requires a static website to be deployed on Kubernetes with the following non-negotiable constraints:

* **High Availability** - The application must tolerate individual pod failures without downtime.
* **Scalability** - The deployment must support horizontal scaling through replica management.
* **External Accessibility** - The application must be reachable from outside the cluster on a deterministic port.

---

## Solution Architecture

```
                        +---------------------------+
                        |      jump-host (node)     |
                        |   control-plane / worker  |
                        +---------------------------+
                                    |
                     NodePort: 30011 (external access)
                                    |
                        +-----------+-----------+
                        |    nginx-service      |
                        |    (NodePort / :80)   |
                        +-----------+-----------+
                                    |
              +---------------------+---------------------+
              |                     |                     |
   +----------+--------+  +---------+---------+  +-------+-----------+
   | nginx-deployment  |  | nginx-deployment  |  | nginx-deployment  |
   |  replica pod 1    |  |  replica pod 2    |  |  replica pod 3    |
   | nginx:latest      |  | nginx:latest      |  | nginx:latest      |
   +-------------------+  +-------------------+  +-------------------+
```

| Resource | Name | Key Configuration |
|---|---|---|
| Deployment | `nginx-deployment` | 3 replicas, image `nginx:latest` |
| Container | `nginx-container` | Port 80, image `nginx:latest` |
| Service | `nginx-service` | Type `NodePort`, nodePort `30011` |

---

## Prerequisites

| Requirement | Details |
|---|---|
| Kubernetes cluster | K3s v1.34.1 or compatible |
| `kubectl` configured | Pointed at target cluster via kubeconfig |
| Cluster node status | All nodes in `Ready` state |
| Network access | Port `30011` open on the node firewall |

---

## Implementation Guide

### Step 1 - Verify Cluster Connectivity

Before provisioning any workload, confirm that `kubectl` is correctly configured and the cluster control plane is reachable. This ensures all subsequent `apply` commands are dispatched to the intended cluster.

```bash
kubectl cluster-info
```

**Expected Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

Verify all cluster nodes are in a `Ready` state:

```bash
kubectl get nodes
```

**Expected Output:**

```
NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   60m   v1.34.1+k3s1
```

> **Screenshot:**

<img width="1033" height="452" alt="image" src="https://github.com/user-attachments/assets/ad20ae68-046c-4a50-bcdf-6886ce0300de" />

>Cluster info and node readiness output confirming the control plane is reachable and the node is in Ready state.

---

### Step 2 - Create the Nginx Deployment

Author the `nginx-deployment.yaml` manifest and apply it to the cluster. The manifest specifies 3 replicas of the `nginx:latest` container, labelled consistently so the Service selector can target them correctly.

```bash
cat <<EOF > nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
EOF
```

Apply the manifest:

```bash
kubectl apply -f nginx-deployment.yaml
```

**Expected Output:**

```
deployment.apps/nginx-deployment created
```

> **Screenshot:**

 <img width="1031" height="643" alt="image" src="https://github.com/user-attachments/assets/b337f925-0455-4914-8d4e-d2c808b61c00" />  
 
 >Terminal output confirming `deployment.apps/nginx-deployment created` after applying the manifest.

---

### Step 3 - Verify Deployment Rollout

Confirm that the Deployment controller has successfully scheduled and started all 3 replicas before proceeding to Service creation.

```bash
kubectl get deployment nginx-deployment
```

**Expected Output:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           29s
```

All three fields `READY`, `UP-TO-DATE`, and `AVAILABLE` must read `3` before the deployment is considered healthy.

> **Screenshot:** `03-deployment-ready.png` - `kubectl get deployment` output showing 3/3 replicas ready and available.

---

### Step 4 - Create the NodePort Service

Author the `nginx-service.yaml` manifest and apply it. The Service uses a `selector` matching the `app: nginx` label applied to all pods in the Deployment, routing inbound traffic on `nodePort: 30011` through to container port `80`.

```bash
cat <<EOF > nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30011
EOF
```

Apply the manifest:

```bash
kubectl apply -f nginx-service.yaml
```

**Expected Output:**

```
service/nginx-service created
```

> **Screenshot:** `04-service-apply.png` - Terminal output confirming `service/nginx-service created` after applying the Service manifest.

---

### Step 5 - Verify Service Provisioning

Validate that the Service has been assigned a `ClusterIP` and is correctly bound to the expected `nodePort`.

```bash
kubectl get service nginx-service
```

**Expected Output:**

```
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.43.149.84   <none>        80:30011/TCP   30s
```

Run a full describe to confirm endpoint population. Endpoints must list the IPs of all 3 running pods. An empty `Endpoints` field indicates a label selector mismatch between the Service and the Deployment pods.

```bash
kubectl describe service nginx-service
```

**Expected Output:**

```
Name:                     nginx-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=nginx
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.149.84
IPs:                      10.43.149.84
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30011/TCP
Endpoints:                10.22.0.10:80,10.22.0.11:80,10.22.0.9:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

> **Screenshot:** `05-service-describe.png` - `kubectl describe service nginx-service` confirming 3 populated Endpoints corresponding to the 3 running pod IPs.

---

### Step 6 - Validate Full Stack Configuration

Perform targeted `jsonpath` queries against the live Deployment object to confirm that all configuration parameters match the task requirements exactly. This step eliminates ambiguity and provides audit-grade verification.

**Verify replica count:**

```bash
kubectl get deployment nginx-deployment -o jsonpath='{.spec.replicas}'
```

**Expected Output:**

```
3
```

**Verify container name:**

```bash
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[0].name}'
```

**Expected Output:**

```
nginx-container
```

**Verify container image:**

```bash
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected Output:**

```
nginx:latest
```

**List all resources under the `app=nginx` label:**

```bash
kubectl get all -l app=nginx
```

**Expected Output:**

```
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-6554fcb65f-b4l45   1/1     Running   0          6m36s
pod/nginx-deployment-6554fcb65f-wspn8   1/1     Running   0          6m36s
pod/nginx-deployment-6554fcb65f-ztlz4   1/1     Running   0          6m36s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-6554fcb65f   3         3         3       6m37s
```

All 3 pods must be in `Running` status with `0` restarts, confirming the containers have started cleanly with no crash-loop behaviour.

> **Screenshot:** `06-full-validation.png` - jsonpath verification outputs and `kubectl get all -l app=nginx` confirming all 3 pods running with 0 restarts.

---

## Manifest Reference

### `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
```

### `nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30011
```

---

## Verification Summary

| Check | Command | Expected Result | Status |
|---|---|---|---|
| Cluster reachable | `kubectl cluster-info` | Control plane at `127.0.0.1:6443` | Passed |
| Node ready | `kubectl get nodes` | `jump-host` in `Ready` state | Passed |
| Deployment created | `kubectl get deployment nginx-deployment` | `3/3` ready | Passed |
| Replica count | `jsonpath .spec.replicas` | `3` | Passed |
| Container name | `jsonpath containers[0].name` | `nginx-container` | Passed |
| Container image | `jsonpath containers[0].image` | `nginx:latest` | Passed |
| Service created | `kubectl get service nginx-service` | `80:30011/TCP` | Passed |
| Endpoints populated | `kubectl describe service nginx-service` | 3 pod IPs listed | Passed |
| All pods running | `kubectl get all -l app=nginx` | 3 pods, `Running`, 0 restarts | Passed |

---

## Best Practices Applied

**Declarative configuration over imperative commands**
Both the Deployment and Service were authored as YAML manifests and applied via `kubectl apply`. This makes the configuration version-controllable, auditable, and reapplicable without side effects, as opposed to imperative `kubectl run` or `kubectl expose` commands.

**Explicit image tag pinning with documented intent**
The `nginx:latest` tag was used as explicitly required by the task specification. In production environments outside of constrained lab tasks, image tags should be pinned to a specific digest or semantic version (e.g. `nginx:1.27.0`) to guarantee immutable, reproducible deployments and eliminate silent breaking changes from upstream image updates.

**Label-based loose coupling between Deployment and Service**
The `app: nginx` label is applied consistently to pod template metadata in the Deployment and matched by the Service `selector`. This decoupled pattern means Services can be updated, replaced, or expanded independently of the workload they front.

**Post-apply endpoint verification**
Rather than assuming the Service is wired correctly after creation, `kubectl describe service nginx-service` was executed to confirm that the `Endpoints` field was populated with all 3 pod IPs. An empty `Endpoints` field is one of the most common silent failures in Kubernetes networking and this step catches it immediately.

**`jsonpath` audit queries for configuration fidelity**
After deployment, targeted `jsonpath` queries were run directly against the live API object to verify the exact stored configuration. This is significantly more reliable than re-reading the original YAML file, as it confirms what Kubernetes actually accepted and stored, not just what was submitted.

---

## Lessons Learned

**Verify node readiness before workload deployment**
Running `kubectl get nodes` as the first step ensures the scheduler has available targets. Attempting to deploy pods when all nodes are in `NotReady` state results in pods stuck in `Pending` with no obvious error from `kubectl apply`.

**The `Endpoints` field is the ground truth for Service routing**
A Service can be created successfully and show a valid `ClusterIP` while still routing traffic to zero backends. The `Endpoints` field in `kubectl describe service` is the definitive indicator of whether the label selector is matching live pods. Always inspect it after Service creation.

**`kubectl apply` is idempotent; use it consistently**
Using `kubectl apply -f` for both initial creation and subsequent updates avoids the confusion of switching between `create` and `apply`. The `apply` approach stores the last-applied configuration as an annotation, enabling clean three-way merges on future updates.

**Single-node clusters mask multi-node scheduling concerns**
On this single-node K3s cluster, all 3 replicas were scheduled to the same node. In a production multi-node cluster, a `PodAntiAffinity` rule should be used to distribute replicas across distinct nodes to achieve true availability guarantees. This is not visible as a failure on a single-node lab cluster.

**NodePort range is cluster-enforced**
Kubernetes enforces that NodePort values fall within the range `30000-32767` by default. The specified port `30011` falls within this range. Attempting to use a port outside this range, such as `8080`, would cause the Service creation to fail with a validation error.

---

## Troubleshooting

**Pods stuck in `Pending` state**

```bash
kubectl describe pod <pod-name>
```

Inspect the `Events` section. Common causes include insufficient CPU or memory resources on the node, or taints with no matching tolerations.

**Service `Endpoints` field is empty**

```bash
kubectl get pods --show-labels
kubectl describe service nginx-service
```

Compare the `Selector` on the Service against the `Labels` on the pods. A single character mismatch will result in zero endpoints and no traffic routing.

**`ImagePullBackOff` on pods**

```bash
kubectl describe pod <pod-name>
```

The node cannot pull `nginx:latest` from the registry. Verify the node has outbound internet access or that an internal mirror is configured. On air-gapped clusters, pre-load the image with `ctr` or `crictl`.

**Port `30011` not reachable externally**

Verify the host firewall allows inbound TCP on port `30011`. On Linux systems using `iptables` or `ufw`:

```bash
# Check ufw status
sudo ufw status

# Allow the NodePort if blocked
sudo ufw allow 30011/tcp
```

---







<img width="1029" height="705" alt="image" src="https://github.com/user-attachments/assets/3463bb7d-5f48-4c03-9e98-15e443758c15" />

<img width="1035" height="672" alt="image" src="https://github.com/user-attachments/assets/4b7e961a-c80c-4a1e-8305-00bea21e4039" />
<img width="1025" height="849" alt="image" src="https://github.com/user-attachments/assets/7ec1d7d8-0e22-4eff-95b1-eec430ae8783" />
<img width="1029" height="868" alt="image" src="https://github.com/user-attachments/assets/4ef25fb4-a835-415a-8d20-bd0384dd05e1" />
<img width="1026" height="505" alt="image" src="https://github.com/user-attachments/assets/60b5a9ab-986c-4c53-8332-cb6ef5c199c0" />
<img width="1028" height="858" alt="image" src="https://github.com/user-attachments/assets/ac8f1310-92e5-4926-9ec5-b0030b08d444" />
<img width="1034" height="532" alt="image" src="https://github.com/user-attachments/assets/edca0ab1-d7bb-4db0-a627-2851e785f812" />
<img width="1031" height="765" alt="image" src="https://github.com/user-attachments/assets/e8992736-7c9d-4fdd-83de-398fc8a0edbb" />
