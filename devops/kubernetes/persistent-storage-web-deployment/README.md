# Kubernetes Persistent Volume Provisioning and NodePort Service Exposure

>A hands-on infrastructure exercise provisioning a `hostPath`-backed PersistentVolume, binding it via a PersistentVolumeClaim, deploying an nginx pod with persistent storage mounted at the web root, and exposing the workload externally through a NodePort service on a single-node k3s cluster.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Cluster Verification](#phase-1-cluster-verification)
  - [Phase 2: PersistentVolume Provisioning](#phase-2-persistentvolume-provisioning)
  - [Phase 3: PersistentVolumeClaim Binding](#phase-3-persistentvolumeclaim-binding)
  - [Phase 4: Pod Deployment with Volume Mount](#phase-4-pod-deployment-with-volume-mount)
  - [Phase 5: NodePort Service Creation](#phase-5-nodeport-service-creation)
  - [Phase 6: Pod Label Correction and Endpoint Verification](#phase-6-pod-label-correction-and-endpoint-verification)
  - [Phase 7: Service Connectivity Validation](#phase-7-service-connectivity-validation)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Lessons Learned](#lessons-learned)

---

## Overview

The Nautilus DevOps team required a Kubernetes template to deploy a web application backed by persistent storage. The objectives were:

* Provision a `PersistentVolume` named `pv-datacenter` using `hostPath` storage at `/mnt/data` with a `manual` storage class and `3Gi` capacity
* Bind it via a `PersistentVolumeClaim` named `pvc-datacenter` requesting `1Gi` with `ReadWriteOnce` access
* Mount the claim inside an nginx pod named `pod-datacenter` at the web server document root `/usr/share/nginx/html`
* Expose the pod externally via a `NodePort` service named `web-datacenter` on port `30008`

All resources were provisioned in the `default` namespace on a k3s single-node cluster accessible via `jump-host`.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     jump-host (k3s)                     │
│                                                         │
│   ┌──────────────┐       ┌──────────────────────────┐   │
│   │  NodePort    │       │     pod-datacenter       │   │
│   │  Service     │──────>│  container-datacenter    │   │
│   │  :30008      │       │  image: nginx:latest     │   │
│   └──────────────┘       │  mountPath:              │   │
│                          │  /usr/share/nginx/html   │   │
│                          └────────────┬─────────────┘   │
│                                       │                 │
│                          ┌────────────▼─────────────┐   │
│                          │    pvc-datacenter        │   │
│                          │    1Gi / RWO / manual    │   │
│                          └────────────┬─────────────┘   │
│                                       │                 │
│                          ┌────────────▼─────────────┐   │
│                          │    pv-datacenter         │   │
│                          │    3Gi / RWO / hostPath  │   │
│                          │    /mnt/data             │   │
│                          └──────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Prerequisites

* kubectl configured against the target cluster at `https://127.0.0.1:6443`
* k3s single-node cluster (`jump-host`) in `Ready` state
* Host directory `/mnt/data` pre-existing on the node (provided by the lab environment)

---

## Implementation Guide

### Phase 1: Cluster Verification

Confirmed the cluster control plane was reachable and the node was in `Ready` state before provisioning any resources.

```bash
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
```

**Terminal output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   54m   v1.34.1+k3s1

NAME              STATUS   AGE
default           Active   54m
kube-node-lease   Active   54m
kube-public       Active   54m
kube-system       Active   54m
```

> Screenshot: Cluster info and node status showing jump-host in Ready state

<img width="1026" height="528" alt="image" src="https://github.com/user-attachments/assets/0059ebbd-3c8e-4a9c-9bd3-aa6db99c0348" />

---

### Phase 2: PersistentVolume Provisioning

Created the PersistentVolume manifest and applied it to the cluster. The PV used `hostPath` storage referencing `/mnt/data` on the node, with `storageClassName: manual` to enable static binding and `Retain` reclaim policy by default.

```bash
cat <<'EOF' > /tmp/pv-datacenter.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-datacenter
spec:
  storageClassName: manual
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
EOF

kubectl apply -f /tmp/pv-datacenter.yaml
```

**Terminal output:**

```
persistentvolume/pv-datacenter created
```

Verified the PV status and confirmed it entered `Available` state, indicating it was unbound and ready to accept a claim:

```bash
kubectl get pv pv-datacenter
kubectl describe pv pv-datacenter
```

**Terminal output:**

```
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
pv-datacenter   3Gi        RWO            Retain           Available           manual         29s

Name:            pv-datacenter
StorageClass:    manual
Status:          Available
Reclaim Policy:  Retain
Access Modes:    RWO
Capacity:        3Gi
Source:
    Type:    HostPath
    Path:    /mnt/data
```

> Screenshot: PV in Available state with hostPath source confirmed

<img width="1062" height="753" alt="image" src="https://github.com/user-attachments/assets/549d73c1-0de0-46c5-bea8-50f5b265526d" />

---

### Phase 3: PersistentVolumeClaim Binding

>Created the PersistentVolumeClaim requesting `1Gi` of storage with `ReadWriteOnce` access mode and `storageClassName: manual`. Kubernetes static binding matched this PVC to `pv-datacenter` because the storage class and access mode aligned.

```bash
cat <<'EOF' > /tmp/pvc-datacenter.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-datacenter
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f /tmp/pvc-datacenter.yaml
```

**Terminal output:**

```
persistentvolumeclaim/pvc-datacenter created
```

Verified the PVC bound successfully to `pv-datacenter`:

```bash
kubectl get pvc pvc-datacenter
kubectl describe pvc pvc-datacenter
```

**Terminal output:**

```
NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-datacenter   Bound    pv-datacenter   3Gi        RWO            manual         37s

Name:          pvc-datacenter
Namespace:     default
StorageClass:  manual
Status:        Bound
Volume:        pv-datacenter
Capacity:      3Gi
Access Modes:  RWO
Used By:       <none>
```

> Screenshot: PVC status showing Bound to pv-datacenter with 3Gi capacity

<img width="1063" height="671" alt="image" src="https://github.com/user-attachments/assets/79e02f15-71e2-4f4d-940d-ae0ba5c38808" />

>Note: The PVC requested `1Gi` but was bound to the entire `3Gi` PV. This is expected Kubernetes behavior. A PVC binds to the smallest available PV that satisfies its constraints; the full PV capacity is allocated, not just the requested amount.

---

### Phase 4: Pod Deployment with Volume Mount

Created the Pod manifest referencing `pvc-datacenter` as the volume source and mounting it at `/usr/share/nginx/html`, which is the nginx document root. The container image was pinned to `nginx:latest` as required by the task specification.

```bash
cat <<'EOF' > /tmp/pod-datacenter.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-datacenter
spec:
  volumes:
    - name: datacenter-storage
      persistentVolumeClaim:
        claimName: pvc-datacenter
  containers:
    - name: container-datacenter
      image: nginx:latest
      volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: datacenter-storage
EOF

kubectl apply -f /tmp/pod-datacenter.yaml
```

**Terminal output:**

```
pod/pod-datacenter created
```

Confirmed the pod reached `Running` state:

```bash
kubectl get pod pod-datacenter
```

**Terminal output:**

```
NAME             READY   STATUS    RESTARTS   AGE
pod-datacenter   1/1     Running   0          12s
```

> Screenshot: Pod pod-datacenter in 1/1 Running state

<img width="1055" height="839" alt="image" src="https://github.com/user-attachments/assets/da02ef18-ce5b-4fde-a21f-da54c4416a11" />

---

### Phase 5: NodePort Service Creation

Created the NodePort service manifest to expose the nginx pod on cluster port `80` mapped to host port `30008`. The service selector used `run: pod-datacenter` as the label key-value pair.

```bash
cat <<'EOF' > /tmp/svc-datacenter.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-datacenter
spec:
  type: NodePort
  selector:
    run: pod-datacenter
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30008
EOF

kubectl apply -f /tmp/svc-datacenter.yaml
```

**Terminal output:**

```
service/web-datacenter created
```

Verified the service was created with the correct NodePort and ClusterIP:

```bash
kubectl get svc web-datacenter
kubectl describe svc web-datacenter
```

**Terminal output:**

```
NAME             TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
web-datacenter   NodePort   10.43.42.16   <none>        80:30008/TCP   30s

Name:                     web-datacenter
Selector:                 run=pod-datacenter
Type:                     NodePort
IP:                       10.43.42.16
Port:                     80/TCP
TargetPort:               80/TCP
NodePort:                 30008/TCP
Endpoints:
```

> Screenshot: Service web-datacenter created with NodePort 30008, Endpoints field empty

<img width="1046" height="864" alt="image" src="https://github.com/user-attachments/assets/17eaa1cd-9cde-4e17-8b07-ea0377a0c8f9" />

>The `Endpoints` field was empty at this stage, indicating the service selector was not matching any pod. This required investigation.

---

### Phase 6: Pod Label Correction and Endpoint Verification

Inspected the pod labels to diagnose the missing endpoint registration:

```bash
kubectl get pod pod-datacenter --show-labels
```

**Terminal output:**

```
NAME             READY   STATUS    RESTARTS   AGE     LABELS
pod-datacenter   1/1     Running   0          4m24s   <none>
```

The pod had no labels. The service selector `run=pod-datacenter` could not match a pod with no labels. Applied the required label imperatively:

```bash
kubectl label pod pod-datacenter run=pod-datacenter
```

**Terminal output:**

```
pod/pod-datacenter labeled
```

Confirmed the endpoint was now registered:

```bash
kubectl get endpoints web-datacenter
```

**Terminal output:**

```
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME             ENDPOINTS      AGE
web-datacenter   10.22.0.9:80   3m19s
```

> Screenshot: Endpoint 10.22.0.9:80 registered under web-datacenter service after label applied

<img width="1056" height="672" alt="image" src="https://github.com/user-attachments/assets/d3ee3b35-6f71-4d91-ba60-59f21ddef753" />

---

### Phase 7: Service Connectivity Validation

Validated end-to-end connectivity from the jump-host using `curl` against the NodePort:

```bash
curl http://localhost:30008
```

**Terminal output:**

```html
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.29.7</center>
</body>
</html>
```

> Screenshot: curl to localhost:30008 returning HTTP 403 from nginx/1.29.7

<img width="1063" height="817" alt="image" src="https://github.com/user-attachments/assets/31633b34-c290-4425-b176-fa343be373b6" />

>The `403 Forbidden` response confirms the nginx container is running and reachable through the NodePort service. The 403 is an expected response: the PersistentVolume mount at `/usr/share/nginx/html` points to an empty host directory (`/mnt/data`), so nginx has no `index.html` to serve and returns a directory listing denial. This is correct behavior given the lab environment pre-condition that the directory exists but is empty.

---

## Errors and Resolutions

### Service Endpoints Empty After Creation

**Symptom:** After applying `svc-datacenter.yaml`, `kubectl describe svc web-datacenter` showed `Endpoints: <empty>`. The service selector `run=pod-datacenter` was not matching the pod.

**Root Cause:** The pod manifest did not include a `labels` field under `metadata`. The pod was created without any labels. Kubernetes services use label selectors to discover pods; without a matching label on the pod, the service registers no endpoints.

**Resolution:** Applied the label imperatively after pod creation:

```bash
kubectl label pod pod-datacenter run=pod-datacenter
```

**Prevention:** Always define pod labels in the manifest `metadata.labels` field that align with the intended service selector. The pod manifest should have included:

```yaml
metadata:
  name: pod-datacenter
  labels:
    run: pod-datacenter
```

---

### HTTP 403 from nginx on Service Validation

**Symptom:** `curl http://localhost:30008` returned `403 Forbidden` rather than an nginx welcome page.

**Root Cause:** The PVC mount at `/usr/share/nginx/html` pointed to `/mnt/data` on the host, which was an empty directory. nginx served a 403 because there were no files to serve and directory listing was disabled by default.

**Resolution:** Not a bug. The 403 confirmed the pod was reachable through the service, the PV mount was functioning correctly, and nginx was operating as expected with an empty document root. Task validation passed successfully.

---

## Key Decisions

* **Static provisioning with `storageClassName: manual`:** Dynamic provisioning was not available or required in this environment. Using `manual` as the storage class name on both the PV and PVC ensured deterministic static binding without invoking a StorageClass provisioner.

* **`hostPath` volume type:** Appropriate for single-node clusters and lab environments. In production multi-node clusters, `hostPath` is not recommended because pod scheduling is not pinned to the node hosting the data. Production workloads should use networked storage backends such as NFS, Ceph, or cloud-provider block storage.

* **Mounting PVC at nginx document root:** Mounting at `/usr/share/nginx/html` rather than a sidecar path or secondary directory correctly positions the PV as the serving layer for the web application, which was the architectural intent of the task.

* **Imperative label application:** The label was applied imperatively via `kubectl label` after identifying the missing selector match. In a production workflow the label would be embedded in the manifest at authoring time to avoid this remediation step.

* **Pinning image to `nginx:latest`:** Done per task specification. In production environments, image tags should be pinned to specific digest-based references to ensure reproducibility and avoid unintended upgrades during node restarts.

---

## Lessons Learned

* **Service selectors are silently non-matching.** Kubernetes does not warn at service creation time if the selector matches zero pods. The `Endpoints` field is the diagnostic signal. Always verify endpoints immediately after service creation with `kubectl get endpoints <service-name>`.

* **PVC capacity reflects PV capacity, not the request.** A PVC requesting `1Gi` that binds to a `3Gi` PV will report `3Gi` capacity. This is not an error. The full PV is allocated to the claim; over-requesting is not possible, but under-requesting is handled by allocating the full volume.

* **`hostPath` PVs are node-local.** Data written to a `hostPath` volume is tied to the specific node. If the pod is rescheduled to a different node, it will not have access to the same data. This reinforces the recommendation to use distributed storage for any stateful production workload.

* **HTTP 403 from nginx is a valid success signal in storage validation contexts.** When the document root is empty, a 403 confirms the service routing, pod networking, and volume mount are all working. A connection refused or timeout would indicate a service or pod issue.

* **Deprecation warnings are informational.** The `v1 Endpoints is deprecated in v1.33+` warning does not affect functionality. The resource continues to work. Migration to `EndpointSlice` is the forward-looking recommendation for new tooling.

* **Label-selector alignment is a foundational Kubernetes contract.** Services, Deployments, and NetworkPolicies all rely on label selectors. Establishing a disciplined labeling strategy at manifest authoring time prevents runtime misconfigurations and eliminates the need for post-deployment corrective actions.
