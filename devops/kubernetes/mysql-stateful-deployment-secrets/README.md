# Kubernetes MySQL Deployment with Persistent Storage, Secrets, and NodePort Service

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify Cluster Connectivity](#step-1-verify-cluster-connectivity)
  - [Step 2: Create Kubernetes Secrets](#step-2-create-kubernetes-secrets)
  - [Step 3: Initial PersistentVolume Creation](#step-3-initial-persistentvolume-creation)
  - [Step 4: Initial PersistentVolumeClaim Creation and Failure](#step-4-initial-persistentvolumeclaim-creation-and-failure)
  - [Step 5: Diagnose PVC Pending State](#step-5-diagnose-pvc-pending-state)
  - [Step 6: Delete Failing PVC and PV](#step-6-delete-failing-pvc-and-pv)
  - [Step 7: Recreate PV with Explicit Empty StorageClassName](#step-7-recreate-pv-with-explicit-empty-storageclassname)
  - [Step 8: Recreate PVC with Explicit Empty StorageClassName](#step-8-recreate-pvc-with-explicit-empty-storageclassname)
  - [Step 9: Confirm PV and PVC Are Bound](#step-9-confirm-pv-and-pvc-are-bound)
  - [Step 10: Deploy MySQL](#step-10-deploy-mysql)
  - [Step 11: Watch Pod Reach Running State](#step-11-watch-pod-reach-running-state)
  - [Step 12: Create NodePort Service](#step-12-create-nodeport-service)
  - [Step 13: Verify Service](#step-13-verify-service)
  - [Step 14: Full Stack Verification](#step-14-full-stack-verification)
  - [Step 15: Validate Environment Variable Injection](#step-15-validate-environment-variable-injection)
  - [Step 16: Validate Database Access](#step-16-validate-database-access)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This project documents the end-to-end deployment of a MySQL 5.7 database server on a Kubernetes cluster for the Nautilus DevOps team. The implementation covers the full lifecycle of a production-grade stateful workload: imperative secret management for sensitive credentials, manual PersistentVolume provisioning with static binding, a Deployment with secret-backed environment variables, and a NodePort Service for external access.

The implementation encountered and resolved a real PVC binding failure caused by k3s default StorageClass interference. A second issue involving an incomplete Deployment heredoc was also observed and is fully documented. All steps below reflect the exact execution sequence performed on the cluster, including errors and their resolutions.

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
│  │ mysql-root-  │─────▶│                            │  │
│  │ pass         │      │  ┌────────────────────────┐ │  │
│  │              │      │  │  Container: mysql:5.7  │ │  │
│  │ mysql-user-  │─────▶│  │  Port: 3306           │ │  │
│  │ pass         │      │  │  Env: secretKeyRef     │ │  │
│  │              │      │  └────────────┬───────────┘ │  │
│  │ mysql-db-url │─────▶│               │            │  │
│  └──────────────┘      └───────────────┼─────────────┘  │
│                                        │                │
│  ┌──────────────────────────┐          │                │
│  │  PersistentVolume        │          │                │
│  │  mysql-pv (250Mi)        │◀─────────┤                │
│  │  storageClassName: ""    │          │                │
│  │  HostPath: /tmp/mysql-pv │          │                │
│  └──────────────────────────┘          │                │
│  ┌──────────────────────────┐          │                │
│  │  PersistentVolumeClaim   │◀─────────┘               │
│  │  mysql-pv-claim (250Mi)  │                           │
│  │  storageClassName: ""    │                           │
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

## Implementation Guide

### Step 1: Verify Cluster Connectivity

Before provisioning any resources, the cluster control plane was confirmed reachable and the node verified in `Ready` state.

```bash
kubectl cluster-info
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```bash
kubectl get nodes
```

**Output:**

```
NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   97m   v1.34.1+k3s1
```

> Screenshot: cluster-info-and-nodes

<img width="1027" height="588" alt="image" src="https://github.com/user-attachments/assets/aa1fe6b9-9aff-442e-8ad1-9a9609693e03" />

---

### Step 2: Create Kubernetes Secrets

Three `Opaque` secrets were created imperatively using `kubectl create secret generic`. This keeps all credential values out of YAML manifests and version control.

**2a. MySQL root password:**

```bash
kubectl create secret generic mysql-root-pass \
  --from-literal=password=YU1idhb667
```

**Output:**

```
secret/mysql-root-pass created
```

**2b. MySQL application user credentials:**

```bash
kubectl create secret generic mysql-user-pass \
  --from-literal=username=kodekloud_tim \
  --from-literal=password=GyQkFRVNr3
```

**Output:**

```
secret/mysql-user-pass created
```

**2c. MySQL database name:**

```bash
kubectl create secret generic mysql-db-url \
  --from-literal=database=kodekloud_db3
```

**Output:**

```
secret/mysql-db-url created
```

**Verify all three secrets:**

```bash
kubectl get secrets
```

**Output:**

```
NAME              TYPE     DATA   AGE
mysql-db-url      Opaque   1      24s
mysql-root-pass   Opaque   1      99s
mysql-user-pass   Opaque   2      62s
```

> Screenshot: kubectl-get-secrets

<img width="1033" height="693" alt="image" src="https://github.com/user-attachments/assets/5c471f5b-a644-457e-9a72-3c2617010349" />

---

### Step 3: Initial PersistentVolume Creation

A `hostPath` PersistentVolume named `mysql-pv` was created with 250Mi capacity. At this stage, no `storageClassName` was explicitly set in the manifest.

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
  hostPath:
    path: /tmp/mysql-pv
  persistentVolumeReclaimPolicy: Retain
EOF
```

**Output:**

```
persistentvolume/mysql-pv created
```

**Verify PV:**

```bash
kubectl get pv mysql-pv
```

**Output:**

```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
mysql-pv   250Mi      RWO            Retain           Available                          <unset>                          25s
```

> Screenshot: pv-initial-created

<img width="1028" height="826" alt="image" src="https://github.com/user-attachments/assets/664c89c3-96d8-4cf8-a2c2-d37da680a1ca" />

---

### Step 4: Initial PersistentVolumeClaim Creation and Failure

A PVC named `mysql-pv-claim` was created requesting 250Mi, also without an explicit `storageClassName`.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
EOF
```

**Output:**

```
persistentvolumeclaim/mysql-pv-claim created
```

**Verify PVC status:**

```bash
kubectl get pvc mysql-pv-claim
```

**Output:**

```
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mysql-pv-claim   Pending                                      local-path     <unset>                 44s
```

***The PVC was stuck in `Pending` and had been automatically assigned the `local-path` StorageClass by k3s. It did not bind to `mysql-pv`.***

> Screenshot: pvc-pending-local-path

<img width="1034" height="835" alt="image" src="https://github.com/user-attachments/assets/b2910067-8107-4eb6-9ef8-ee95a7444744" />

---

### Step 5: Diagnose PVC Pending State

`kubectl describe` was run on the PVC to inspect its events and determine the root cause of the binding failure.

```bash
kubectl describe pvc mysql-pv-claim
```

**Output:**

```
Name:          mysql-pv-claim
Namespace:     default
StorageClass:  local-path
Status:        Pending
Volume:
Labels:        <none>
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:
Access Modes:
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type    Reason                Age                  From                         Message
  ----    ------                ----                 ----                         -------
  Normal  WaitForFirstConsumer  5s (x10 over 2m18s)  persistentvolume-controller  waiting for first consumer to be created before binding
```

***The `WaitForFirstConsumer` event confirmed the k3s `local-path` provisioner had taken ownership of the PVC. It was waiting for a pod to be scheduled before proceeding. The manually created `mysql-pv` was being ignored entirely because of the StorageClass mismatch.***

> Screenshot: pvc-describe-pending-events

<img width="1150" height="739" alt="image" src="https://github.com/user-attachments/assets/b64aed21-aeb7-4112-8ba6-fb2bdf6ab1a6" />

---

### Step 6: Delete Failing PVC and PV

Both resources were deleted to allow clean recreation with the correct `storageClassName` configuration.

```bash
kubectl delete pvc mysql-pv-claim
```

**Output:**

```
persistentvolumeclaim "mysql-pv-claim" deleted from default namespace
```

```bash
kubectl delete pv mysql-pv
```

**Output:**

```
persistentvolume "mysql-pv" deleted
```

> Screenshot: pv-pvc-deleted

<img width="1145" height="821" alt="image" src="https://github.com/user-attachments/assets/2762b481-6d00-43c0-8add-58817d2dca69" />

---

### Step 7: Recreate PV with Explicit Empty StorageClassName

The PV was recreated with `storageClassName: ""` explicitly set. This opts the PV out of any default StorageClass and makes it eligible only for static binding.

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

**Output:**

```
persistentvolume/mysql-pv created
```

> Screenshot: pv-recreated-empty-storageclass

<img width="1144" height="737" alt="image" src="https://github.com/user-attachments/assets/4c1d6a3c-1edc-46f6-b145-013bac148dd7" />

---

### Step 8: Recreate PVC with Explicit Empty StorageClassName

The PVC was recreated with `storageClassName: ""` to match the corrected PV and enforce static binding.

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

**Output:**

```
persistentvolumeclaim/mysql-pv-claim created
```

> Screenshot: pvc-recreated-empty-storageclass

---

### Step 9: Confirm PV and PVC Are Bound

Both resources were verified in a single command block.

```bash
kubectl get pv mysql-pv
kubectl get pvc mysql-pv-claim
```

**Output:**

```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
mysql-pv   250Mi      RWO            Retain           Bound    default/mysql-pv-claim                  <unset>                          67s

NAME             STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mysql-pv-claim   Bound    mysql-pv   250Mi      RWO                           <unset>                 36s
```

***Both resources now show `Bound`, confirming successful static binding between the PV and PVC.***

> Screenshot: pv-pvc-bound-confirmed

---

### Step 10: Deploy MySQL

A `Deployment` named `mysql-deployment` was applied with one replica running `mysql:5.7`. All four MySQL environment variables were injected via `secretKeyRef`. The manifest was submitted as follows, exactly as it was entered in the terminal session:

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
EOF         claimName: mysql-pv-claim
```

**Output:**

```
deployment.apps/mysql-deployment created
```

***The `spec.volumes` section was not included in this heredoc. The text `claimName: mysql-pv-claim` appeared after `EOF` on the same line and was treated by the shell as a separate command, not as YAML content. The `volumeMounts` block references a volume named `mysql-storage` that was never defined in `spec.volumes`. Despite this, Kubernetes accepted the Deployment without error and the pod reached `Running`. The PVC was not actually mounted into the container as a result. See [Error 2](#error-2-deployment-applied-with-incomplete-volumes-block) for full analysis.***

> Screenshot: mysql-deployment-created

---

### Step 11: Watch Pod Reach Running State

```bash
kubectl get pods -l app=mysql --watch
```

**Output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
mysql-deployment-74446ff65d-nxw8q   1/1     Running   0          69s
```

> Screenshot: mysql-pod-running

---

### Step 12: Create NodePort Service

A `NodePort` Service named `mysql` was created to expose port `3306` externally on node port `30007`.

```bash
cat <<EOF | kubectl apply -f - -
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

**Output:**

```
service/mysql created
```

> Screenshot: mysql-service-created

---

### Step 13: Verify Service

```bash
kubectl get svc mysql
```

**Output:**

```
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
mysql   NodePort   10.43.65.229   <none>        3306:30007/TCP   43s
```

> Screenshot: mysql-service-verified

---

### Step 14: Full Stack Verification

All provisioned resources were reviewed in a single wide query.

```bash
kubectl get pv,pvc,deployment,pod,svc
```

**Output:**

```
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/mysql-pv   250Mi      RWO            Retain           Bound    default/mysql-pv-claim                  <unset>                          6m13s

NAME                                   STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/mysql-pv-claim   Bound    mysql-pv   250Mi      RWO                           <unset>                 5m42s

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

### Step 15: Validate Environment Variable Injection

The pod was exec'd into using a dynamic pod name lookup via `jsonpath` to confirm all secret-backed environment variables were correctly injected at runtime.

```bash
kubectl exec -it $(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- env | grep MYSQL
```

**Output:**

```
MYSQL_MAJOR=5.7
MYSQL_VERSION=5.7.44-1.el7
MYSQL_SHELL_VERSION=8.0.35-1.el7
MYSQL_ROOT_PASSWORD=YU1idhb667
MYSQL_DATABASE=kodekloud_db3
MYSQL_USER=kodekloud_tim
MYSQL_PASSWORD=GyQkFRVNr3
```

***All four application-level environment variables match the values defined in their respective secrets.***

> Screenshot: env-injection-verified

---

### Step 16: Validate Database Access

MySQL connectivity was validated by logging in as the application user and listing available databases from inside the running container.

```bash
kubectl exec -it $(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u kodekloud_tim -pGyQkFRVNr3 -e "SHOW DATABASES;"
```

**Output:**

```
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| kodekloud_db3      |
+--------------------+
```

***The `kodekloud_db3` database was automatically created by MySQL on initialization because `MYSQL_DATABASE` was set. Login as `kodekloud_tim` succeeded, confirming both the user credentials and database secret were correctly consumed.***

> Screenshot: mysql-database-access-verified

---

## Errors Encountered and Resolutions

### Error 1: PVC Stuck in `Pending` Due to k3s Default StorageClass Assignment

**Step where it occurred:** Step 4

**Symptom:**

After creating both the PV and PVC without specifying `storageClassName`, the PVC remained in `Pending` indefinitely and was automatically assigned `local-path` by the cluster.

```
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mysql-pv-claim   Pending                                      local-path     <unset>                 44s
```

`kubectl describe pvc mysql-pv-claim` revealed the following event, repeated ten times over two minutes:

```
Normal  WaitForFirstConsumer  5s (x10 over 2m18s)  persistentvolume-controller  waiting for first consumer to be created before binding
```

**Root Cause:**

k3s ships with `local-path` configured as the cluster-default `StorageClass`. When `storageClassName` is omitted from a PVC definition, Kubernetes automatically annotates it with the default StorageClass. The `local-path` provisioner operates in `WaitForFirstConsumer` binding mode, meaning it defers volume creation until a pod is scheduled to a node. As a result, the PVC was tied to the `local-path` provisioner's lifecycle and the manually created `mysql-pv`, which had no `storageClassName` set, was invisible to the binding process.

**Resolution:**

Both the PVC and PV were deleted. They were then recreated with `storageClassName: ""` explicitly on both manifests. An empty string value is the Kubernetes-standard mechanism to opt out of dynamic provisioning and force static binding between a specific PV and PVC pair. After recreation, the PVC immediately transitioned to `Bound`.

---

### Error 2: Deployment Applied with Incomplete `volumes` Block

**Step where it occurred:** Step 10

**Symptom:**

The Deployment heredoc was submitted before the `spec.volumes` section was included. The text `claimName: mysql-pv-claim` was placed after the `EOF` terminator on the same terminal line, which caused the shell to treat it as a separate command rather than part of the YAML body. Kubernetes accepted and created the Deployment without returning any error.

**What was missing from the applied manifest:**

```yaml
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
```

**Impact:**

The `volumeMounts` block inside the container spec referenced a volume named `mysql-storage` that was never defined under `spec.volumes`. In this k3s environment, Kubernetes did not reject the manifest. The pod reached `1/1 Running` successfully. However, the PVC `mysql-pv-claim` was not mounted into the container. MySQL data was being written to the container's ephemeral filesystem rather than the persistent volume, meaning any data would be lost on pod deletion or rescheduling.

**Production Remediation:**

A corrected Deployment manifest including the complete `spec.volumes` block should be applied with `kubectl apply` to patch the existing Deployment:

```yaml
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
```

After the corrected Deployment is applied, the existing pod will be replaced by a new pod that mounts the PVC at `/var/lib/mysql` as intended.

---

## Best Practices Applied

* **Secret isolation per credential scope:** Three separate `Opaque` secrets were created rather than consolidating all values into one. This follows the principle of least privilege and simplifies future secret rotation by scoping each secret to a single credential concern.

* **`secretKeyRef` for all sensitive environment variables:** No plaintext credentials appear in the Deployment manifest. All sensitive values are injected at runtime via `secretKeyRef`, preventing accidental credential exposure in terminal history, YAML files, or version control.

* **`persistentVolumeReclaimPolicy: Retain`:** The PV is configured with `Retain` to ensure the underlying host data directory is not automatically purged when the PVC is deleted, providing a safety net against accidental data loss.

* **Label-based selector alignment:** The Deployment uses `app: mysql` on pod templates and the Service uses the same label as its selector, maintaining a clean, decoupled binding between the workload and the service layer.

* **Dynamic pod name resolution in `kubectl exec`:** Rather than hardcoding a pod name, the `jsonpath` pattern `kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}'` was used inline. This pattern is portable across rollouts and directly reusable in CI/CD health-check scripts.

* **Full-stack resource verification before functional testing:** `kubectl get pv,pvc,deployment,pod,svc` was run as a single command to confirm all layers of the stack were healthy before proceeding to in-cluster validation.

---

## Lessons Learned

* **Omitting `storageClassName` in a k3s cluster silently assigns `local-path`.** This does not produce an error at PVC creation time. The failure manifests only as a `Pending` PVC. Always explicitly set `storageClassName: ""` on both PV and PVC when performing static provisioning on any cluster with a default StorageClass configured.

* **`WaitForFirstConsumer` is not always a pod-scheduling issue.** This event message is misleading when seen on a manually created PV. It indicates the `local-path` provisioner has taken ownership of the PVC and is deferring volume creation. The actual problem is a StorageClass mismatch. `kubectl describe pvc` is the correct first diagnostic step whenever a PVC does not bind immediately.

* **`storageClassName: ""` must be set on both the PV and the PVC.** Setting it only on the PV is not sufficient. If the PVC omits it, the cluster-default StorageClass will be applied to the PVC automatically, and the binding will fail silently.

* **Kubernetes does not always reject a Deployment with an undefined volume reference.** A `volumeMounts` entry referencing a volume name not defined under `spec.volumes` was accepted without error in this k3s environment and the pod ran successfully. This means admission validation alone cannot be relied on to confirm that persistent storage is correctly wired. After deploying a stateful workload, always verify actual mount state inside the container using `kubectl exec -- df -h` or `kubectl exec -- mount | grep mysql`.

* **Passing credentials inline on the MySQL CLI triggers a security warning.** The `mysql: [Warning] Using a password on the command line interface can be insecure` message is expected behavior when using `-p` inline. In production, credentials should be passed via a MySQL options file or environment variable to avoid exposure in shell history and process listings.

* **Heredoc submissions should be reviewed before closing the `EOF` marker.** In interactive terminal sessions, it is easy to close a heredoc before all required YAML sections are included. A reliable practice is to write the full manifest to a file first, review it, and then apply it with `kubectl apply -f filename.yaml` rather than piping inline.

---

## References

* [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
* [MySQL 5.7 Docker Image](https://hub.docker.com/_/mysql)
* [k3s Default StorageClass](https://docs.k3s.io/storage)
* [Kubernetes Volume Binding Modes](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)














<img width="1034" height="853" alt="image" src="https://github.com/user-attachments/assets/cacef03e-9f8c-489f-b2c0-83adea609885" />


<img width="1150" height="783" alt="image" src="https://github.com/user-attachments/assets/1bb5a1d1-32b9-4dc8-aa2f-7119492f9d6b" />


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
