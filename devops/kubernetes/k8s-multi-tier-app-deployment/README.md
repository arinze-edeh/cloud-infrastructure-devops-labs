# Kubernetes Multi-Tier Application Deployment: Iron Gallery with MariaDB Backend

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Implementation](#implementation)
  * [Step 1: Verify Cluster Health](#step-1-verify-cluster-health)
  * [Step 2: Create Namespace](#step-2-create-namespace)
  * [Step 3: Deploy Iron Gallery Frontend](#step-3-deploy-iron-gallery-frontend)
  * [Step 4: Deploy Iron DB Backend](#step-4-deploy-iron-db-backend)
  * [Step 5: Create Iron DB ClusterIP Service](#step-5-create-iron-db-clusterip-service)
  * [Step 6: Create Iron Gallery NodePort Service](#step-6-create-iron-gallery-nodeport-service)
  * [Step 7: Final Validation](#step-7-final-validation)
* [Resource Summary](#resource-summary)
* [Best Practices](#best-practices)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)

---

## Overview

This implementation documents the deployment of the **Iron Gallery** application, a multi-tier web application developed by the Nautilus DevOps team, onto a production Kubernetes cluster. The deployment comprises a frontend gallery service backed by a MariaDB database, organized under a dedicated namespace with appropriate resource constraints, persistent volume strategies, and service exposure configurations.

The scope covers the full lifecycle from namespace provisioning through service validation, applying enterprise Kubernetes patterns including label-based service discovery, emptyDir volume isolation, resource limit enforcement, and NodePort-based external access.

---

## Problem Statement

The Nautilus DevOps team completed customizations to the Iron Gallery application and required a structured Kubernetes deployment that satisfies the following constraints:

* Logical isolation via a dedicated namespace
* Two independent deployments: one for the frontend gallery and one for the MariaDB backend
* Resource limits enforced at the container level to prevent runaway consumption
* Volume mounts using `emptyDir` for transient data storage aligned with the application's data and upload directories
* Internal database access via a stable `ClusterIP` service
* External frontend access via a `NodePort` service on a specific port
* Verification that the application installation page is reachable before declaring the deployment complete

---

## Architecture

```
                        External Traffic
                              |
                         NodePort: 32678
                              |
                 +------------------------+
                 | iron-gallery-service   |
                 |  (NodePort, port 80)   |
                 +------------------------+
                              |
                 +------------------------+
                 | iron-gallery-deployment|
                 |  kodekloud/irongallery |
                 |  CPU: 50m | Mem: 100Mi |
                 |  Vol: config, images   |
                 +------------------------+

                 +------------------------+
                 |  iron-db-service       |
                 |  (ClusterIP, port 3306)|
                 +------------------------+
                              |
                 +------------------------+
                 | iron-db-deployment     |
                 |  kodekloud/irondb:2.0  |
                 |  Env: MYSQL_* vars     |
                 |  Vol: db (emptyDir)    |
                 +------------------------+

         Namespace: iron-namespace-devops
         Cluster: k3s v1.34.1 (single-node control plane)
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Kubernetes cluster | k3s v1.34.1, single-node control plane |
| kubectl access | Configured on `jump-host`, pointed at `https://127.0.0.1:6443` |
| Container images | `kodekloud/irongallery:2.0`, `kodekloud/irondb:2.0` |
| Network access | NodePort `32678` reachable from jump-host loopback |

---

## Implementation

### Step 1: Verify Cluster Health

Before provisioning any resources, confirm the cluster control plane and CoreDNS are fully operational.

```bash
kubectl cluster-info
kubectl get nodes
```

**Output:**

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at ...

NAME        STATUS   ROLES           AGE   VERSION
jump-host   Ready    control-plane   34m   v1.34.1+k3s1
```

The node is `Ready` and the control plane components are healthy. Proceed with resource creation.

*Screenshot: kubectl cluster-info and node status confirming healthy control plane*

<img width="1020" height="645" alt="image" src="https://github.com/user-attachments/assets/0be9cf6f-a3ca-4e5c-b04c-d8d564bc9388" />

---

### Step 2: Create Namespace

Isolate all Iron Gallery resources under a dedicated namespace to enforce logical boundary separation and prevent resource collision with other workloads.

```bash
kubectl create namespace iron-namespace-devops
kubectl get namespace iron-namespace-devops
```

**Output:**

```
namespace/iron-namespace-devops created

NAME                    STATUS   AGE
iron-namespace-devops   Active   13s
```

The namespace is `Active` and ready to accept workloads.

*Screenshot: Namespace creation and verification output*

<img width="1034" height="588" alt="image" src="https://github.com/user-attachments/assets/2c262524-dbd0-4a81-b2e1-bd7f44ac9e12" />

---

### Step 3: Deploy Iron Gallery Frontend

Define and apply the gallery deployment manifest. The deployment enforces CPU and memory limits, mounts two `emptyDir` volumes for runtime data separation, and uses label `run: iron-gallery` for downstream service discovery.

**Create the manifest:**

```bash
cat <<'EOF' > iron-gallery-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iron-gallery-deployment-devops
  namespace: iron-namespace-devops
  labels:
    run: iron-gallery
spec:
  replicas: 1
  selector:
    matchLabels:
      run: iron-gallery
  template:
    metadata:
      labels:
        run: iron-gallery
    spec:
      containers:
        - name: iron-gallery-container-devops
          image: kodekloud/irongallery:2.0
          resources:
            limits:
              memory: "100Mi"
              cpu: "50m"
          volumeMounts:
            - name: config
              mountPath: /usr/share/nginx/html/data
            - name: images
              mountPath: /usr/share/nginx/html/uploads
      volumes:
        - name: config
          emptyDir: {}
        - name: images
          emptyDir: {}
EOF
```

**Apply and verify:**

```bash
kubectl apply -f iron-gallery-deployment.yaml
kubectl get deployment iron-gallery-deployment-devops -n iron-namespace-devops
kubectl describe deployment iron-gallery-deployment-devops -n iron-namespace-devops
```

**Output:**

```
deployment.apps/iron-gallery-deployment-devops created

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
iron-gallery-deployment-devops   1/1     1            1           33s
```

The `describe` output confirms:
* Selector: `run=iron-gallery`
* Container: `iron-gallery-container-devops` running `kodekloud/irongallery:2.0`
* Limits: `cpu: 50m`, `memory: 100Mi`
* Volume `config` mounted at `/usr/share/nginx/html/data` (emptyDir)
* Volume `images` mounted at `/usr/share/nginx/html/uploads` (emptyDir)
* ReplicaSet progressed successfully: `1/1` available

*Screenshots: kubectl get and describe output for iron-gallery-deployment-devops*

<img width="1025" height="860" alt="image" src="https://github.com/user-attachments/assets/abadcbc0-fd8b-4785-8907-35a10bf13a25" />
<img width="1034" height="868" alt="image" src="https://github.com/user-attachments/assets/4cfa2be1-44f7-4516-b75b-c333f1792180" />
<img width="1032" height="864" alt="image" src="https://github.com/user-attachments/assets/fc204cae-f4c9-471b-b2e8-9f1b9b0961a7" />

---

### Step 4: Deploy Iron DB Backend

Define and apply the database deployment manifest. The container is configured with four environment variables for database initialization. A single `emptyDir` volume backs the MariaDB data directory.

**Create the manifest:**

```bash
cat <<'EOF' > iron-db-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iron-db-deployment-devops
  namespace: iron-namespace-devops
  labels:
    db: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      db: mariadb
  template:
    metadata:
      labels:
        db: mariadb
    spec:
      containers:
        - name: iron-db-container-devops
          image: kodekloud/irondb:2.0
          env:
            - name: MYSQL_DATABASE
              value: "database_host"
            - name: MYSQL_ROOT_PASSWORD
              value: "R00t@S3cur3P@ss#2024"
            - name: MYSQL_PASSWORD
              value: "Db@P@ssw0rd#Secure99"
            - name: MYSQL_USER
              value: "iron_db_user"
          volumeMounts:
            - name: db
              mountPath: /var/lib/mysql
      volumes:
        - name: db
          emptyDir: {}
EOF
```

**Apply and verify:**

```bash
kubectl apply -f iron-db-deployment.yaml
kubectl get deployment iron-db-deployment-devops -n iron-namespace-devops
kubectl describe deployment iron-db-deployment-devops -n iron-namespace-devops
```

**Output:**

```
deployment.apps/iron-db-deployment-devops created

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
iron-db-deployment-devops   1/1     1            1           28s
```

The `describe` output confirms:
* Selector: `db=mariadb`
* Container: `iron-db-container-devops` running `kodekloud/irondb:2.0`
* Environment variables: `MYSQL_DATABASE`, `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`, `MYSQL_USER` all set
* Volume `db` mounted at `/var/lib/mysql` (emptyDir)
* ReplicaSet `iron-db-deployment-devops-5544d79954` progressed: `1/1` available

*Screenshots: kubectl describe output showing environment variables and volume configuration for iron-db-deployment-devops*

<img width="1028" height="858" alt="image" src="https://github.com/user-attachments/assets/ceacb600-6abd-4237-a910-9c519237e85e" />
<img width="1030" height="842" alt="image" src="https://github.com/user-attachments/assets/1b38333b-f101-4dae-b9d3-d730193db15e" />
<img width="1034" height="861" alt="image" src="https://github.com/user-attachments/assets/8f091a6b-aa81-45ab-ac34-49d41c0d347f" />
<img width="1032" height="867" alt="image" src="https://github.com/user-attachments/assets/e7107b43-d167-4f89-8017-f7cb897d2684" />

>*Full YAML output of iron-db-deployment-devops including status conditions*

---

### Step 5: Create Iron DB ClusterIP Service

Expose the MariaDB deployment internally within the cluster on port 3306 using a `ClusterIP` service. This ensures the gallery frontend can resolve the database endpoint via DNS without exposing it externally.

**Create the manifest:**

```bash
cat <<'EOF' > iron-db-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: iron-db-service-devops
  namespace: iron-namespace-devops
spec:
  type: ClusterIP
  selector:
    db: mariadb
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
EOF
```

**Apply and verify:**

```bash
kubectl apply -f iron-db-service.yaml
kubectl get service iron-db-service-devops -n iron-namespace-devops
kubectl describe service iron-db-service-devops -n iron-namespace-devops
```

**Output:**

```
service/iron-db-service-devops created

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
iron-db-service-devops   ClusterIP   10.43.252.109   <none>        3306/TCP   116s
```

The `describe` output confirms:
* Type: `ClusterIP`
* Selector: `db=mariadb` matching the database pod label
* Endpoint resolved: `10.22.0.10:3306` (database pod IP)
* No external access surface

*Screenshot: kubectl get and describe output for iron-db-service-devops showing ClusterIP and active endpoint*

<img width="1061" height="867" alt="image" src="https://github.com/user-attachments/assets/275c4b13-ed88-43d0-9394-2c357b7f6603" />

>*Full YAML output of iron-db-service-devops confirming spec configuration*

---

### Step 6: Create Iron Gallery NodePort Service

Expose the frontend gallery deployment externally on NodePort `32678`. Traffic entering on the node's port is forwarded to the gallery pod on port 80.

**Create the manifest:**

```bash
cat <<'EOF' > iron-gallery-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: iron-gallery-service-devops
  namespace: iron-namespace-devops
spec:
  type: NodePort
  selector:
    run: iron-gallery
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 32678
EOF
```

**Apply and verify:**

```bash
kubectl apply -f iron-gallery-service.yaml
kubectl get service iron-gallery-service-devops -n iron-namespace-devops
kubectl describe service iron-gallery-service-devops -n iron-namespace-devops
```

**Output:**

```
service/iron-gallery-service-devops created

NAME                          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
iron-gallery-service-devops   NodePort   10.43.165.231   <none>        80:32678/TCP   48s
```

The `describe` output confirms:
* Type: `NodePort`
* Selector: `run=iron-gallery` matching the frontend pod label
* NodePort: `32678` bound correctly
* Endpoint resolved: `10.22.0.9:80` (gallery pod IP)

*Screenshot: kubectl get and describe output for iron-gallery-service-devops showing NodePort binding*

*Screenshot: Full YAML output of iron-gallery-service-devops confirming nodePort 32678 configuration*

---

### Step 7: Final Validation

Perform a complete end-state verification across all deployed resources, then confirm the application is reachable over HTTP.

**Verify all deployments:**

```bash
kubectl get deployments -n iron-namespace-devops
```

```
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
iron-db-deployment-devops        1/1     1            1           8m39s
iron-gallery-deployment-devops   1/1     1            1           11m
```

**Verify all pods:**

```bash
kubectl get pods -n iron-namespace-devops
```

```
NAME                                              READY   STATUS    RESTARTS   AGE
iron-db-deployment-devops-5544d79954-t982k        1/1     Running   0          9m22s
iron-gallery-deployment-devops-7b59878ffd-2wvqf   1/1     Running   0          12m
```

**Verify all services:**

```bash
kubectl get services -n iron-namespace-devops
```

```
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
iron-db-service-devops        ClusterIP   10.43.252.109   <none>        3306/TCP       6m58s
iron-gallery-service-devops   NodePort    10.43.165.231   <none>        80:32678/TCP   2m52s
```

**HTTP connectivity test:**

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:32678
```

```
200
```

**Application content verification:**

```bash
curl -s http://127.0.0.1:32678 | head -10
```

```
<!DOCTYPE HTML>
<html>
      <head>
            <meta http-equiv="Content-Type" content="text/html;charset=utf-8">
            <title>Lychee</title>
            <meta name="author" content="Tobias Reich">
            <link type="text/css" rel="stylesheet" href="dist/main.css">
```

The application returns HTTP `200` and serves the Lychee gallery installation page, confirming the deployment is fully functional.

*Screenshot: kubectl get deployments, pods, and services showing all resources in Running/Available state*

*Screenshot: curl HTTP 200 response and HTML head output confirming application reachability on NodePort 32678*

---

## Resource Summary

| Resource Kind | Name | Namespace | Key Configuration |
|---|---|---|---|
| Namespace | iron-namespace-devops | cluster-wide | Dedicated isolation boundary |
| Deployment | iron-gallery-deployment-devops | iron-namespace-devops | 1 replica, cpu: 50m, mem: 100Mi, 2x emptyDir |
| Deployment | iron-db-deployment-devops | iron-namespace-devops | 1 replica, 4 env vars, 1x emptyDir |
| Service | iron-db-service-devops | iron-namespace-devops | ClusterIP, TCP 3306 |
| Service | iron-gallery-service-devops | iron-namespace-devops | NodePort, TCP 80:32678 |

---

## Best Practices

**Namespace isolation:** Grouping all application resources under `iron-namespace-devops` enables clean RBAC boundaries, resource quota application, and namespace-scoped deletion without affecting other workloads in the cluster.

**Label consistency:** The label schema (`run: iron-gallery`, `db: mariadb`) is applied uniformly across Deployment metadata, pod template metadata, and Service selectors. This eliminates selector drift and ensures services always route to the correct pods.

**Resource limits on frontend container:** Setting explicit `cpu: 50m` and `memory: 100Mi` limits on the gallery container prevents a poorly behaved pod from monopolizing node resources, which is critical in shared cluster environments. The absence of resource limits on the database container is acceptable given the controlled lab scope, but should be remedied in production.

**Service type selection:** `ClusterIP` for the database correctly restricts network access to intra-cluster traffic only. `NodePort` for the frontend provides a predictable external entry point on a fixed port without requiring an Ingress controller or LoadBalancer provisioner.

**emptyDir for transient workloads:** Using `emptyDir` volumes for both application data directories and the database volume is appropriate for ephemeral or stateless lab contexts. The volumes are lifecycle-bound to the pod, ensuring clean state on restart.

**Declarative manifests:** All resources were applied using YAML manifest files (`kubectl apply -f`) rather than imperative commands. This ensures the configuration is version-controllable, auditable, and reproducible across environments.

**Pre-deployment cluster health check:** Validating cluster-info and node status before resource creation prevents wasted effort against a degraded control plane.

---

## Errors and Resolutions

No blocking errors were encountered during this deployment. All manifests applied cleanly on first attempt. The following potential failure modes were proactively avoided:

| Potential Issue | Avoidance Strategy Applied |
|---|---|
| Service selector mismatch | Label values on Deployment template and Service selector were verified to be identical before apply |
| NodePort conflict | Port `32678` was confirmed unused by other services in the cluster before manifest creation |
| Image pull failure | Exact image tags (`kodekloud/irongallery:2.0`, `kodekloud/irondb:2.0`) were specified per task requirements; `imagePullPolicy` defaulted to `IfNotPresent` for cached environments |
| Pod not reaching Running state | `kubectl describe` was used after each deployment apply to inspect ReplicaSet events and confirm pod scheduling |

---

## Lessons Learned

**ClusterIP endpoints are immediately resolvable via kube-dns.** Once `iron-db-service-devops` was created, the database became reachable at `iron-db-service-devops.iron-namespace-devops.svc.cluster.local:3306` from any pod within the cluster. This reinforces that service-based DNS is the correct mechanism for inter-pod communication rather than hardcoded pod IPs.

**emptyDir volumes are non-persistent by design.** If the pod is deleted or rescheduled, all data in emptyDir volumes is lost. For production MariaDB deployments, a `PersistentVolumeClaim` backed by a `StorageClass` is required. This distinction is critical when planning upgrade or restart strategies.

**Resource limits without requests can cause scheduling ambiguity.** This deployment specifies `limits` but not `requests` on the gallery container. Kubernetes will treat the request as equal to the limit in this case, which can make bin-packing less efficient on multi-node clusters. Always define both `requests` and `limits` in production manifests.

**HTTP 200 from the NodePort confirms end-to-end routing health.** The curl validation using `%{http_code}` is a lightweight but effective smoke test. It confirms the NodePort binding, kube-proxy forwarding, container responsiveness, and nginx health within a single command. This pattern should be integrated into any post-deployment verification script.

**Credential management via plain-text env vars is a starting point only.** The database passwords were passed as literal string values in the Deployment spec. In production, these values must be stored in Kubernetes `Secret` objects and referenced via `secretKeyRef` to prevent credential exposure in version-controlled manifests and `kubectl describe` output.






<img width="1030" height="864" alt="image" src="https://github.com/user-attachments/assets/a36e0224-f422-48fa-ad68-bb4d31f7a61c" />
<img width="1030" height="864" alt="image" src="https://github.com/user-attachments/assets/dd983238-3114-4aba-99d1-d089d6f7843f" />

<img width="1036" height="854" alt="image" src="https://github.com/user-attachments/assets/d6d0ad39-5e65-4f46-91a7-59f5c0175d4b" />

<img width="1038" height="552" alt="image" src="https://github.com/user-attachments/assets/cfd64210-b4b1-4395-8b79-65ab5ba66424" />
<img width="1032" height="589" alt="image" src="https://github.com/user-attachments/assets/e983f1e0-4923-4350-9416-cf833af82415" />
<img width="1065" height="864" alt="image" src="https://github.com/user-attachments/assets/aead86d9-0be4-4c1e-a7fc-c27d9cb1dae8" />
<img width="1065" height="622" alt="image" src="https://github.com/user-attachments/assets/8681de3a-435c-46f2-8bec-9aec2ad2a6db" />

<img width="1063" height="762" alt="image" src="https://github.com/user-attachments/assets/891b7266-a11c-4b1e-adaf-5b2725ad7b56" />
<img width="1057" height="802" alt="image" src="https://github.com/user-attachments/assets/c7621d76-15c5-40eb-b2a7-b2b9dba2f7e1" />
<img width="1066" height="856" alt="image" src="https://github.com/user-attachments/assets/536b1954-80e9-405a-957c-d0c59e07d8c0" />
<img width="1057" height="862" alt="image" src="https://github.com/user-attachments/assets/e0d25f8f-0065-4343-bb77-7aaa102e2e67" />
<img width="1055" height="571" alt="image" src="https://github.com/user-attachments/assets/bd0a16fc-db29-42cf-8467-4998821f8c44" />
<img width="1055" height="727" alt="image" src="https://github.com/user-attachments/assets/f828b57c-0f7d-4bb5-b784-6e24183fee94" />
<img width="1068" height="856" alt="image" src="https://github.com/user-attachments/assets/fd0759f4-741a-4baf-a3a6-005383a5c2ce" />
<img width="1061" height="860" alt="image" src="https://github.com/user-attachments/assets/1c1fd1cb-c129-4feb-a4ba-7da09e0a903f" />
<img width="1059" height="857" alt="image" src="https://github.com/user-attachments/assets/b793cf29-f31f-43c4-bc9c-ea7ca8b603e8" />
<img width="1066" height="865" alt="image" src="https://github.com/user-attachments/assets/c3f9a15c-942a-4851-86cc-b631680d4ba7" />
<img width="1069" height="867" alt="image" src="https://github.com/user-attachments/assets/c7938702-b182-433b-856b-2a352e5c7f7c" />
<img width="1067" height="861" alt="image" src="https://github.com/user-attachments/assets/99e12f0a-ede5-4240-9b43-f5b64da8a57f" />
<img width="1063" height="856" alt="image" src="https://github.com/user-attachments/assets/8d5c87a2-bc69-4e20-8cb6-b03874399871" />
<img width="1064" height="725" alt="image" src="https://github.com/user-attachments/assets/2cb285f2-8646-408e-ab6e-06830d35dd4b" />
<img width="1059" height="274" alt="image" src="https://github.com/user-attachments/assets/a7e56b15-5585-402a-9f99-cd93af9cc2a5" />
