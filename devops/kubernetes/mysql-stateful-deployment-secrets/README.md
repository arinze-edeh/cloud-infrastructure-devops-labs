# Kubernetes MySQL Deployment with Persistent Storage, Secrets, and NodePort Service

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify Cluster Connectivity](#step-1-verify-cluster-connectivity)
  - [Step 2: Create Kubernetes Secrets](#step-2-create-kubernetes-secrets)
  - [Step 3: Provision PersistentVolume](#step-3-provision-persistentvolume)
  - [Step 4: Provision PersistentVolumeClaim](#step-4-provision-persistentvolumeclaim)
  - [Step 5: Deploy MySQL](#step-5-deploy-mysql)
  - [Step 6: Expose MySQL via NodePort Service](#step-6-expose-mysql-via-nodeport-service)
  - [Step 7: Verify Full Stack](#step-7-verify-full-stack)
  - [Step 8: Validate Environment Variables and Database Access](#step-8-validate-environment-variables-and-database-access)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This project documents the end-to-end deployment of a MySQL 5.7 database server on a Kubernetes cluster for the Nautilus DevOps team. The implementation covers the full lifecycle of a production-grade stateful workload: secret management for sensitive credentials, manual PersistentVolume provisioning with static binding, Deployment configuration with secret-backed environment variables, and NodePort Service exposure.

The deployment satisfies all six requirements defined in the task brief and has been validated through in-cluster `mysql` CLI verification.

| Attribute | Value |
|---|---|
| Cluster | k3s single-node (`jump-host`) |
| Kubernetes Version | v1.34.1+k3s1 |
| MySQL Image | `mysql:5.7` |
| Storage | 250Mi HostPath PersistentVolume |
| Service Type | NodePort |
| NodePort | 30007 |
| Namespace | `default` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                    │
│                                                         │
│  ┌──────────────┐      ┌─────────────────────────────┐  │
│  │   Secrets    │      │     mysql-deployment        │  │
│  │              │      │   (Deployment / 1 replica)  │  │
│  │ mysql-root-  │─────▶│                             │  │
│  │ pass         │      │  ┌────────────────────────┐ │  │
│  │              │      │  │  Container: mysql:5.7  │ │  │
│  │ mysql-user-  │─────▶│  │  Port: 3306            │ │  │
│  │ pass         │      │  │  Env: secretKeyRef     │ │  │
│  │              │      │  └────────────┬───────────┘ │  │
│  │ mysql-db-url │─────▶│               │             │  │
│  └──────────────┘      └───────────────┼─────────────┘  │
│                                        │                │
│  ┌──────────────────────────┐          │                │
│  │  PersistentVolume        │          │                │
│  │  mysql-pv (250Mi)        │◀─────────┤                │
│  │  HostPath: /tmp/mysql-pv │          │                │
│  └──────────────────────────┘          │                │
│  ┌──────────────────────────┐          │                │
│  │  PersistentVolumeClaim   │◀─────────┘                │
│  │  mysql-pv-claim (250Mi)  │                           │
│  └──────────────────────────┘                           │
│                                                         │
│  ┌──────────────────────────┐                           │
│  │  Service: mysql          │                           │
│  │  Type: NodePort          │                           │
│  │  Port: 3306 → 30007      │                           │
│  └──────────────────────────┘                           │
└─────────────────────────────────────────────────────────┘
```

---

## Prerequisites

* A running Kubernetes cluster with `kubectl` configured on the jump host
* Cluster-admin or namespace-scoped permissions for Secrets, PV, PVC, Deployment, and Service resources
* Internet access from cluster nodes to pull the `mysql:5.7` image from Docker Hub

---

## Project Structure

```
kubernetes-mysql-deployment/
README.md                    # This document
screenshots/                 # Execution screenshots (see placeholders below)
```

---

## Implementation Guide

### Step 1: Verify Cluster Connectivity

Before provisioning any resources, confirm that the cluster control plane is reachable and that the node is in `Ready` state.

```bash
kubectl cluster-info
kubectl get nodes
```

**Expected output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   97m   v1.34.1+k3s1
```

> Screenshot: cluster-info-and-nodes

---

### Step 2: Create Kubernetes Secrets

Three `Opaque` secrets are created imperatively using `kubectl create secret generic`. This approach keeps sensitive credential values out of YAML manifests and version control.

**2a. MySQL root password**

```bash
kubectl create secret generic mysql-root-pass \
  --from-literal=password=YU1idhb667
```

**2b. MySQL application user credentials**

```bash
kubectl create secret generic mysql-user-pass \
  --from-literal=username=kodekloud_tim \
  --from-literal=password=GyQkFRVNr3
```

**2c. MySQL database name**

```bash
kubectl create secret generic mysql-db-url \
  --from-literal=database=kodekloud_db3
```

**Verify all three secrets exist:**

```bash
kubectl get secrets
```

**Expected output:**

```
NAME              TYPE     DATA   AGE
mysql-db-url      Opaque   1      24s
mysql-root-pass   Opaque   1      99s
mysql-user-pass   Opaque   2      62s
```

> Screenshot: kubectl-get-secrets

---

### Step 3: Provision PersistentVolume

A `hostPath` PersistentVolume named `mysql-pv` is created with a capacity of 250Mi and a `storageClassName` explicitly set to `""` (empty string). Setting `storageClassName: ""` is critical for static binding; it prevents the default `StorageClass` from interfering with PVC assignment.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 250Mi
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  hostPath:
    path: /tmp/mysql-pv
  persistentVolumeReclaimPolicy: Retain
EOF
```

**Verify PV status:**

```bash
kubectl get pv mysql-pv
```

**Expected output:**

```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
mysql-pv   250Mi      RWO            Retain           Available                          25s
```

> Screenshot: pv-created-available

---

### Step 4: Provision PersistentVolumeClaim

A PersistentVolumeClaim named `mysql-pv-claim` is created requesting 250Mi of storage with `storageClassName: ""` to match the statically provisioned PV above. This explicit pairing bypasses dynamic provisioning entirely.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  resources:
    requests:
      storage: 250Mi
EOF
```

**Verify PV and PVC are mutually Bound:**

```bash
kubectl get pv mysql-pv
kubectl get pvc mysql-pv-claim
```

**Expected output:**

```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   AGE
mysql-pv   250Mi      RWO            Retain           Bound    default/mysql-pv-claim                  67s

NAME             STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    mysql-pv   250Mi      RWO                           36s
```

> Screenshot: pv-pvc-bound

---

### Step 5: Deploy MySQL

A `Deployment` named `mysql-deployment` is created with one replica running `mysql:5.7`. All four MySQL environment variables (`MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`) are injected via `secretKeyRef`, referencing the secrets created in Step 2. The PVC is mounted at `/var/lib/mysql`, which is the standard MySQL data directory.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-root-pass
                  key: password
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-db-url
                  key: database
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-user-pass
                  key: username
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-user-pass
                  key: password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
EOF
```

**Watch pod reach Running state:**

```bash
kubectl get pods -l app=mysql --watch
```

**Expected output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
mysql-deployment-74446ff65d-nxw8q   1/1     Running   0          69s
```

> Screenshot: mysql-pod-running

---

### Step 6: Expose MySQL via NodePort Service

A `NodePort` Service named `mysql` is created to expose the MySQL Deployment on node port `30007`, mapping to container port `3306`.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: NodePort
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
      nodePort: 30007
EOF
```

**Verify Service:**

```bash
kubectl get svc mysql
```

**Expected output:**

```
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
mysql   NodePort   10.43.65.229   <none>        3306:30007/TCP   43s
```

> Screenshot: mysql-service-nodeport

---

### Step 7: Verify Full Stack

Confirm that all resources are healthy in a single view.

```bash
kubectl get pv,pvc,deployment,pod,svc
```

**Expected output:**

```
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   AGE
persistentvolume/mysql-pv   250Mi      RWO            Retain           Bound    default/mysql-pv-claim                  6m13s

NAME                                   STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql-pv-claim   Bound    mysql-pv   250Mi      RWO                           5m42s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql-deployment   1/1     1            1           4m5s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/mysql-deployment-74446ff65d-nxw8q   1/1     Running   0          4m5s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP          112m
service/mysql        NodePort    10.43.65.229   <none>        3306:30007/TCP   106s
```

> Screenshot: full-stack-verification

---

### Step 8: Validate Environment Variables and Database Access

**8a. Confirm environment variables are injected correctly from secrets:**

```bash
kubectl exec -it $(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- env | grep MYSQL
```

**Expected output:**

```
MYSQL_MAJOR=5.7
MYSQL_VERSION=5.7.44-1.el7
MYSQL_SHELL_VERSION=8.0.35-1.el7
MYSQL_ROOT_PASSWORD=YU1idhb667
MYSQL_DATABASE=kodekloud_db3
MYSQL_USER=kodekloud_tim
MYSQL_PASSWORD=GyQkFRVNr3
```

> Screenshot: env-injection-verified

**8b. Validate database connectivity and schema creation as the application user:**

```bash
kubectl exec -it $(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u kodekloud_tim -pGyQkFRVNr3 -e "SHOW DATABASES;"
```

**Expected output:**

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| kodekloud_db3      |
+--------------------+
```

> Screenshot: mysql-database-access-verified

---

## Errors Encountered and Resolutions

### Error 1: PVC Stuck in `Pending` State Due to StorageClass Mismatch

**Symptom:**

After creating the initial PV and PVC without an explicit `storageClassName`, the PVC remained in `Pending` status indefinitely.

```
kubectl describe pvc mysql-pv-claim
...
Events:
  Normal  WaitForFirstConsumer  5s (x10 over 2m18s)  persistentvolume-controller  waiting for first consumer to be created before binding
```

**Root Cause:**

On k3s clusters, a default `StorageClass` named `local-path` is automatically configured. When `storageClassName` is omitted from both the PV and PVC definitions, the PVC is assigned `local-path` by default. The manually created PV had no `storageClassName`, so the binding controller could not match them. The `WaitForFirstConsumer` event indicated the scheduler was expecting dynamic provisioning from `local-path`, not static binding to the existing PV.

**Resolution:**

Delete both the PVC and PV, then recreate them with `storageClassName: ""` explicitly set on both resources. An empty string value opts out of the default `StorageClass` and enables direct static binding between the PV and PVC.

```bash
kubectl delete pvc mysql-pv-claim
kubectl delete pv mysql-pv
```

Recreate both with `storageClassName: ""` on both manifests. The PVC immediately transitioned to `Bound`.

---

## Best Practices Applied

* **Secret isolation per credential type:** Three separate `Opaque` secrets (`mysql-root-pass`, `mysql-user-pass`, `mysql-db-url`) are created rather than aggregating all values into a single secret. This follows the principle of least privilege and simplifies secret rotation.

* **`secretKeyRef` for all environment variables:** No plaintext credentials appear in the Deployment manifest. All sensitive values are injected at runtime via `secretKeyRef`, preventing accidental credential exposure in Git history or CI/CD logs.

* **Explicit `storageClassName: ""`:** Both PV and PVC carry `storageClassName: ""` to force static binding. This avoids non-deterministic behavior from the default `StorageClass` in k3s environments, making the binding relationship explicit and reproducible.

* **`persistentVolumeReclaimPolicy: Retain`:** The PV is configured with `Retain` rather than `Delete`. In a production MySQL deployment, `Retain` ensures that the underlying data on the node is not automatically purged when the PVC is deleted, providing a safety net against accidental data loss.

* **Label-based pod selection:** The Deployment uses `app: mysql` labels on both the pod template and the Service selector, ensuring a clean, decoupled binding between the workload and the service layer.

* **Pod readiness verification before proceeding:** `kubectl get pods --watch` was used to confirm the pod reached `1/1 Running` before creating the Service, avoiding premature traffic routing to an unready backend.

* **In-cluster functional validation:** Both environment variable injection and actual MySQL connectivity were verified from inside the running container using `kubectl exec`, providing end-to-end confidence without requiring any external network tooling.

---

## Lessons Learned

* **Default `StorageClass` in k3s silently prevents static PV binding.** In any k3s environment, always explicitly set `storageClassName: ""` on both PV and PVC when performing static provisioning. Omitting this field does not result in an immediate error, only a silent `Pending` state that requires describe output to diagnose.

* **`WaitForFirstConsumer` does not always mean "wait for a pod."** This event message is misleading in a static binding context. It appears because the `local-path` provisioner is volume-topology-aware and defers provisioning until a pod is scheduled. When you see this on a manually created PV, the actual issue is a `StorageClass` mismatch, not a missing consumer.

* **Separating secrets by functional scope is operationally superior.** Having three distinct secrets means rotating the root password, for example, does not require touching the application user credentials. In a real production rotation workflow, this separation significantly reduces blast radius.

* **`kubectl exec` with `jsonpath` pod selection is a reliable validation pattern.** Using `kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}'` inline within `kubectl exec` avoids hardcoding pod names, which change with every rollout. This pattern is directly portable to CI/CD health-check scripts.

* **`hostPath` volumes are appropriate only for single-node clusters.** This PV type works correctly in this k3s single-node environment but is not suitable for multi-node clusters where pod scheduling is not guaranteed to land on the node hosting the data directory. In a production multi-node setup, a network-attached StorageClass (EBS, Ceph, NFS) should be used instead.

---

## References

* [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
* [MySQL 5.7 Docker Image](https://hub.docker.com/_/mysql)
* [k3s Storage Classes](https://docs.k3s.io/storage)



<img width="1027" height="588" alt="image" src="https://github.com/user-attachments/assets/aa1fe6b9-9aff-442e-8ad1-9a9609693e03" />
<img width="1035" height="573" alt="image" src="https://github.com/user-attachments/assets/6cece40f-4d14-4b07-8201-51d21b15d270" />
<img width="1032" height="708" alt="image" src="https://github.com/user-attachments/assets/e7845f4b-7afb-427b-8202-e027e6a6a547" />
<img width="1039" height="709" alt="image" src="https://github.com/user-attachments/assets/a807c131-c627-4ff5-a1d2-b132d4b5ccc1" />
<img width="1033" height="693" alt="image" src="https://github.com/user-attachments/assets/5c471f5b-a644-457e-9a72-3c2617010349" />
<img width="1027" height="822" alt="image" src="https://github.com/user-attachments/assets/ad1c9874-5f18-48d4-acde-8bb586a59eba" />
<img width="1028" height="826" alt="image" src="https://github.com/user-attachments/assets/664c89c3-96d8-4cf8-a2c2-d37da680a1ca" />
<img width="1034" height="853" alt="image" src="https://github.com/user-attachments/assets/cacef03e-9f8c-489f-b2c0-83adea609885" />
<img width="1034" height="835" alt="image" src="https://github.com/user-attachments/assets/b2910067-8107-4eb6-9ef8-ee95a7444744" />
<img width="1150" height="739" alt="image" src="https://github.com/user-attachments/assets/b64aed21-aeb7-4112-8ba6-fb2bdf6ab1a6" />
<img width="1150" height="783" alt="image" src="https://github.com/user-attachments/assets/1bb5a1d1-32b9-4dc8-aa2f-7119492f9d6b" />
<img width="1145" height="821" alt="image" src="https://github.com/user-attachments/assets/2762b481-6d00-43c0-8add-58817d2dca69" />
<img width="1144" height="737" alt="image" src="https://github.com/user-attachments/assets/4c1d6a3c-1edc-46f6-b145-013bac148dd7" />
<img width="1144" height="686" alt="image" src="https://github.com/user-attachments/assets/6dcd458a-82f1-4f5c-a256-3655630eb1d4" />
<img width="1150" height="724" alt="image" src="https://github.com/user-attachments/assets/d6781290-76d2-4c63-822a-2070b75f785a" />
<img width="1148" height="857" alt="image" src="https://github.com/user-attachments/assets/ff48fff5-6f5e-4ad8-90e2-e96aee23eb81" />
<img width="1150" height="862" alt="image" src="https://github.com/user-attachments/assets/9227713b-bc95-4d6c-814f-3437b67e4757" />
<img width="1155" height="573" alt="image" src="https://github.com/user-attachments/assets/11424933-bde8-48df-a28e-22657b001824" />
<img width="1155" height="383" alt="image" src="https://github.com/user-attachments/assets/6cfee30a-fe1f-4df1-b824-ee069b98feb0" />
<img width="1154" height="439" alt="image" src="https://github.com/user-attachments/assets/7b08894e-a835-4e69-aa24-90d1ddb49147" />
<img width="1144" height="442" alt="image" src="https://github.com/user-attachments/assets/4ebbb89f-4bf9-407d-a06f-2d61b76d89e6" />
<img width="1158" height="615" alt="image" src="https://github.com/user-attachments/assets/f7e2a05d-b857-43f3-aa6b-c35566614fdf" />
<img width="1150" height="787" alt="image" src="https://github.com/user-attachments/assets/99bc1df5-00cd-43f4-a207-cccecb575a05" />
