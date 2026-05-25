# Kubernetes DevOps Labs

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34.1-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![K3s](https://img.shields.io/badge/Distribution-K3s-FFC61C?style=flat-square&logo=k3s&logoColor=black)
![Platform](https://img.shields.io/badge/Platform-KodeKloud%20%2F%20Nautilus-0A66C2?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-28a745?style=flat-square)

---

## Overview

This directory documents production-aligned Kubernetes engineering work completed across the Nautilus DevOps lab environment. Each project targets a real operational scenario: deploying stateful workloads, repairing broken configurations, managing secrets, enforcing resource governance, exposing services, and rolling back faulty releases.

All tasks were executed on a live K3s v1.34.1 single-node cluster using `kubectl` from a `jump-host` control plane. The work reflects the decision-making, verification discipline, and remediation patterns expected in production Kubernetes environments.

---

## Directory Structure

```
kubernetes/
├── cluster-workload-orchestration/
├── deployment-fault-diagnosis-redis/
├── deployment-rollback-nginx/
├── flask-deployment-misconfiguration-remediation/
├── grafana-nodeport-deployment/
├── guestbook-redis-multitier/
├── init-containers-volume-sharing/
├── k8s-multi-tier-app-deployment/
├── k8s-rolling-update-orchestration/
├── k8s-secrets-secure-workload/
├── k8s-workload-resource-governance/
├── kubernetes-pod-specification-and-deployment/
├── multi-container-shared-emptydir-volume/
├── mysql-stateful-deployment-secrets/
├── nginx-phpfpm-configmap-repair/
├── nginx-static-deployment/
├── persistent-storage-web-deployment/
├── pod-env-var-injection/
├── redis-configmap-deployment/
└── sidecar-log-shipping-emptydir-nginx/
```

---

## Project Summaries

### [Apache HTTPD Deployment on K3s](./cluster-workload-orchestration/)

**Quick Summary:** Deployed an Apache HTTPD container as a Kubernetes `Deployment` on a single-node K3s cluster, validated all three resource layers (Pod, ReplicaSet, Deployment), and produced a full operational runbook.

| | |
|---|---|
| **Purpose** | Establish a baseline deployment workflow covering pre-flight checks, resource creation, and multi-layer post-deployment validation. |
| **Approach** | Used `kubectl create deployment` with an explicit image tag, confirmed context and cluster health upfront, and verified the resource hierarchy with `kubectl get all -l app=httpd`. |
| **Outcome** | Running deployment with `1/1` replicas, `0` restarts, SHA256 image digest confirmed, and full event log captured. |

---

### [Redis Deployment Fault Diagnosis and Repair](./deployment-fault-diagnosis-redis/)

**Quick Summary:** Diagnosed and repaired a broken `redis-deployment` with two simultaneous faults: a wrong container image and a ConfigMap name typo, both resolved in-place without deleting the deployment.

| | |
|---|---|
| **Purpose** | Simulate a real incident where a team member's change degraded a live deployment, requiring root cause isolation and surgical remediation. |
| **Approach** | Used `kubectl set image` to fix the image and a `kubectl get \| sed \| kubectl apply` pipeline to correct the ConfigMap reference typo (`redis-conig` to `redis-config`). |
| **Outcome** | Deployment restored to `1/1` Running with `0` restarts. Rollout history and audit trail preserved. |

---

### [Nginx Deployment Rollback](./deployment-rollback-nginx/)

**Quick Summary:** Executed a controlled rollback of `nginx-deployment` from a faulty `nginx:stable` release back to the previously validated `nginx:1.16` revision using `kubectl rollout undo`.

| | |
|---|---|
| **Purpose** | Respond to a customer-reported regression by reverting to the last known-good revision without downtime. |
| **Approach** | Inspected each revision with `--revision=N` before acting, issued `kubectl rollout undo`, then verified all three pods via JSONPath image query to confirm uniformity. |
| **Outcome** | All 3 pods confirmed running `nginx:1.16`. Old ReplicaSet retained for further rollback if needed. New Revision 3 created, Revision 1 consumed as expected. |

---

### [Flask Deployment Misconfiguration Remediation](./flask-deployment-misconfiguration-remediation/)

**Quick Summary:** Resolved two independent faults in a live Flask deployment: a wrong container image and a Service `targetPort` mismatch that prevented traffic from reaching the application on NodePort `32345`.

| | |
|---|---|
| **Purpose** | Restore end-to-end connectivity to a Python Flask app after a pre-existing deployment was found non-functional at both the workload and service layers. |
| **Approach** | Fixed the image with `kubectl set image`, exported the Service manifest, corrected `targetPort` from `8080` to Flask's actual port `5000`, and reapplied. |
| **Outcome** | Pod reached `Running`, endpoint resolved to `:5000`, and `curl` returned `Hello World Pyvo 1!` confirming full stack connectivity. |

---

### [Grafana NodePort Deployment](./grafana-nodeport-deployment/)

**Quick Summary:** Provisioned a Grafana monitoring instance using declarative YAML manifests and exposed it externally on NodePort `32000`, confirmed with an HTTP `302 Found` redirect to `/login`.

| | |
|---|---|
| **Purpose** | Deploy a centralized analytics tool for the Nautilus DevOps team using a manifest-first, label-consistent workflow. |
| **Approach** | Authored both Deployment and Service manifests with aligned `app: grafana` selectors. Used `rollout status` as a convergence gate before running the `curl -I` probe. |
| **Outcome** | Pod at `1/1 Running`, `3000:32000/TCP` port mapping confirmed, Grafana login page reachable. |

---

### [Multi-Tier Guestbook Application (Redis + PHP Frontend)](./guestbook-redis-multitier/)

**Quick Summary:** Deployed a full three-tier PHP-Redis Guestbook application with 6 Kubernetes resources: Redis master, Redis slave replicas, PHP frontend, and three corresponding Services, using sequential rollout validation.

| | |
|---|---|
| **Purpose** | Demonstrate distributed stateful workload deployment with DNS-based inter-service discovery and load distribution across replicas. |
| **Approach** | Applied each tier sequentially, gating on `kubectl rollout status` before creating dependent Services. Used `GET_HOSTS_FROM=dns` for CoreDNS-based hostname resolution. Frontend image pinned to a SHA256 digest for supply chain integrity. |
| **Outcome** | All 6 pods Running, 3 Services with populated endpoints, and `kubectl get all` confirming full resource health. |

---

### [Init Containers with Shared Volume Pre-Initialization](./init-containers-volume-sharing/)

**Quick Summary:** Implemented the init container pattern to stage a welcome message on a shared `emptyDir` volume before the main container starts, then validated the lifecycle with JSONPath spec queries and log inspection.

| | |
|---|---|
| **Purpose** | Demonstrate Kubernetes-native bootstrapping for workloads that require pre-runtime configuration without modifying container images. |
| **Approach** | Init container writes to `/ic/beta` via shell redirect; main container loops `cat /ic/beta` every 5 seconds. All 8 spec fields verified programmatically via JSONPath. |
| **Outcome** | Init container confirmed `Terminated / Completed / Exit Code 0`. Main container log output repeated correctly every cycle. |

---

### [Iron Gallery Multi-Tier Deployment with MariaDB](./k8s-multi-tier-app-deployment/)

**Quick Summary:** Deployed a two-tier web gallery application with a MariaDB backend in a dedicated namespace, enforcing resource limits, `emptyDir` volume isolation, and correct ClusterIP and NodePort service bindings.

| | |
|---|---|
| **Purpose** | Provision a production-aligned namespace-isolated multi-tier application with explicit resource constraints and validated end-to-end HTTP connectivity. |
| **Approach** | Created `iron-namespace-devops`, applied manifests for both deployments and both services, and confirmed via `curl` returning HTTP `200` on NodePort `32678`. |
| **Outcome** | Both pods Running at `1/1`, correct service endpoints confirmed, `curl` returned full Lychee gallery HTML confirming application health. |

---

### [Nginx Rolling Update (1.16 to 1.17)](./k8s-rolling-update-orchestration/)

**Quick Summary:** Performed a zero-downtime rolling image upgrade of `nginx-deployment` from `nginx:1.16` to `nginx:1.17`, including label discovery to correct an incorrect pod selector, image digest verification, and browser-level validation.

| | |
|---|---|
| **Purpose** | Apply an application release update across a 3-replica deployment without downtime, resolving a selector mismatch along the way. |
| **Approach** | Confirmed container name from `describe`, issued `kubectl set image`, monitored rollout, then verified all pods via `--show-labels`, ReplicaSet hash inspection, and SHA256 digest cross-check. |
| **Outcome** | All 3 pods running `nginx:1.17` (digest `sha256:6fff55...`). Old ReplicaSet scaled to 0. Browser confirmed `Welcome to nginx!` on NodePort `30008`. |

---

### [Kubernetes Secrets Management with Pod Volume Mount](./k8s-secrets-secure-workload/)

**Quick Summary:** Stored a licence key from a host file as a Kubernetes `Opaque` Secret and projected it into a running Fedora container at `/opt/cluster/news.txt` via a volume mount, confirmed with `kubectl exec`.

| | |
|---|---|
| **Purpose** | Demonstrate credential isolation by decoupling sensitive configuration from both container images and pod specs using native Kubernetes Secrets. |
| **Approach** | Used `--from-file=news.txt=/opt/news.txt` for explicit key naming, mounted via `spec.volumes[].secret.secretName`, and validated the projected value with in-container `cat`. |
| **Outcome** | Secret value `5ecur3` confirmed accessible at the expected path. Container started clean with `0` restarts. |

---

### [Pod Resource Governance with CPU and Memory Limits](./k8s-workload-resource-governance/)

**Quick Summary:** Provisioned an `httpd:latest` pod with explicit resource `requests` and `limits`, validated all fields with JSONPath queries, and confirmed `Burstable` QoS class assignment.

| | |
|---|---|
| **Purpose** | Enforce compute boundaries to prevent noisy-neighbour effects and enable scheduler-informed pod placement. |
| **Approach** | Applied a two-phase dry run (`--dry-run=client` then `--dry-run=server`) before live apply. Used JSONPath to verify `requests` and `limits` fields directly against the API object. |
| **Outcome** | Pod running with `memory: 15Mi/20Mi` and `cpu: 100m`. QoS `Burstable` confirmed. `Image ID` SHA256 recorded for audit. |

---

### [Apache HTTPD Pod with Labels](./kubernetes-pod-specification-and-deployment/)

**Quick Summary:** Created a named `httpd:latest` pod with a required `app=httpd_app` label using a declarative manifest, validated with `--show-labels` and a targeted `describe` grep.

| | |
|---|---|
| **Purpose** | Establish correct pod labelling as a prerequisite for Service selector alignment and downstream network policy targeting. |
| **Approach** | Used `kubectl run --dry-run=client -o yaml` as a pre-flight scaffold check before authoring the final manifest. Applied with `kubectl apply` for idempotency. |
| **Outcome** | Pod at `1/1 Running`, label `app=httpd_app` confirmed, container name and image verified via `describe \| grep`. |

---

### [Multi-Container Pod with Shared emptyDir Volume](./multi-container-shared-emptydir-volume/)

**Quick Summary:** Deployed a two-container Fedora pod sharing a single `emptyDir` volume at different mount paths, then validated cross-container file visibility with `kubectl exec`.

| | |
|---|---|
| **Purpose** | Demonstrate intra-pod file sharing as a prerequisite for sidecar and adapter container patterns. |
| **Approach** | Both containers reference the same volume name (`volume-share`) but mount it at different paths (`/tmp/blog` and `/tmp/apps`). File written in Container 1 read successfully from Container 2 without copying. |
| **Outcome** | `2/2 Running`, `0` restarts. Cross-container read confirmed: `Welcome to xFusionCorp Industries` output from both mount paths. |

---

### [MySQL Stateful Deployment with Secrets and Persistent Storage](./mysql-stateful-deployment-secrets/)

**Quick Summary:** Provisioned a MySQL 5.7 deployment backed by a `hostPath` PersistentVolume, three Kubernetes Secrets for credential injection, and a NodePort Service, including diagnosis and resolution of a k3s StorageClass binding failure.

| | |
|---|---|
| **Purpose** | Deploy a production-pattern stateful database workload with credential isolation, persistent storage, and external access. |
| **Approach** | Diagnosed PVC stuck in `Pending` caused by k3s `local-path` default StorageClass interference. Resolved by explicitly setting `storageClassName: ""` on both PV and PVC to enforce static binding. Credentials verified via in-cluster `mysql` client. |
| **Outcome** | PV/PVC `Bound`, pod `Running`, `SHOW DATABASES` confirmed `kodekloud_db3` created on init, `kodekloud_tim` login successful. |

---

### [Nginx and PHP-FPM Sidecar Pod Repair with ConfigMap](./nginx-phpfpm-configmap-repair/)

**Quick Summary:** Diagnosed a broken Nginx and PHP-FPM sidecar pod, rewrote the `nginx-config` ConfigMap with a correct FastCGI configuration, recreated the pod with proper `subPath` volume mounting, and validated PHP execution end-to-end.

| | |
|---|---|
| **Purpose** | Restore a non-functional PHP application stack by correcting the Nginx configuration and deploying the application file via `kubectl cp`. |
| **Approach** | Used `subPath` to mount only `nginx.conf` without replacing `/etc/nginx/` entirely. Copied `index.php` from the jump host post-deploy. Validated with `kubectl exec -- curl localhost:8099/index.php`. |
| **Outcome** | Pod at `2/2 Running`, phpinfo() page served, browser confirmed PHP 7.2.34 with FPM/FastCGI Server API on port `30008`. |

---

### [Nginx Deployment with NodePort Service](./nginx-static-deployment/)

**Quick Summary:** Deployed a 3-replica Nginx static site with a NodePort Service on port `30011`, verified with endpoint population checks and JSONPath configuration audits.

| | |
|---|---|
| **Purpose** | Provision a fault-tolerant, externally accessible web server as a baseline for routing and scaling labs. |
| **Approach** | Declarative Deployment and Service manifests with consistent `app: nginx` label selectors. Confirmed all 3 endpoints registered in `kubectl describe svc` before declaring the deployment complete. |
| **Outcome** | `3/3` pods Running, `10.22.0.9/10/11:80` endpoints confirmed, JSONPath queries validated replicas, container name, and image. |

---

### [Persistent Storage Web Deployment](./persistent-storage-web-deployment/)

**Quick Summary:** Provisioned a `hostPath` PersistentVolume, bound it via a PVC with `storageClassName: manual`, deployed an Nginx pod with the PVC mounted at the web root, and exposed it on NodePort `30008`.

| | |
|---|---|
| **Purpose** | Demonstrate static PV provisioning and PVC binding as a foundation for stateful web workloads. |
| **Approach** | Used `storageClassName: manual` on both PV and PVC for deterministic static binding. Applied `kubectl label` post-deploy to fix a missing pod label that left Service endpoints empty. |
| **Outcome** | PV/PVC `Bound` at `3Gi`, pod `Running`, `curl localhost:30008` returned `403 Forbidden` confirming nginx reachable with an empty document root. |

---

### [Pod Environment Variable Injection](./pod-env-var-injection/)

**Quick Summary:** Provisioned a one-shot Bash pod injecting three environment variables via native `env` spec fields, confirmed runtime propagation via `kubectl logs`.

| | |
|---|---|
| **Purpose** | Validate environment variable injection as a prerequisite for config-driven application deployments. |
| **Approach** | Used `restartPolicy: Never` to prevent crash loop events on a cleanly exiting container. Double-quoted the echo argument to handle the space in `"Welcome to"`. |
| **Outcome** | Pod reached `Completed` with `0` restarts. Log output: `Welcome to xFusionCorp Datacenter`. |

---

### [Redis ConfigMap Volume Deployment](./redis-configmap-deployment/)

**Quick Summary:** Created a `my-redis-config` ConfigMap with a `maxmemory 2mb` directive, mounted it as a file into a `redis:alpine` container alongside an `emptyDir` data volume, and validated all 13 spec requirements.

| | |
|---|---|
| **Purpose** | Externalize Redis runtime configuration into a Kubernetes ConfigMap to support dynamic parameter changes without image rebuilds. |
| **Approach** | Mounted ConfigMap as a volume (not an env var) so Redis reads it as a file. Used JSONPath exec to confirm file content at `/redis-master/redis-config` post-deploy. |
| **Outcome** | All 13 validation checkpoints passed. Pod at `1/1 Running`, CPU request `1` confirmed, file content `maxmemory 2mb` verified in-container. |

---

### [Nginx Sidecar Log Shipping with emptyDir](./sidecar-log-shipping-emptydir-nginx/)

**Quick Summary:** Implemented the Sidecar container pattern with an Nginx primary and an Ubuntu sidecar sharing a common `emptyDir` volume for continuous log shipping, validated with full event log and digest inspection.

| | |
|---|---|
| **Purpose** | Decouple log-forwarding concerns from the NGINX process using a purpose-built sidecar, enabling future integration with a log-aggregation endpoint. |
| **Approach** | Both containers mount `shared-logs` at `/var/log/nginx`. The sidecar runs `while true; do cat access.log error.log; sleep 30; done`. Dry run validation applied before live apply. |
| **Outcome** | Pod at `2/2 Running`, `0` restarts, both containers confirmed `Ready: True` with all 5 pod conditions `True`. Image digests recorded for audit. |

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Orchestration | Kubernetes v1.34.1, K3s |
| CLI | kubectl, curl, sed, bash heredoc |
| Workload Types | Deployment, Pod, StatefulSet-pattern, CronJob-equivalent |
| Storage | PersistentVolume, PersistentVolumeClaim, emptyDir, hostPath |
| Configuration | ConfigMap, Secret (Opaque, secretKeyRef, volumeMount) |
| Networking | ClusterIP, NodePort, FastCGI, CoreDNS |
| Images | nginx, httpd, redis:alpine, php:fpm-alpine, grafana/grafana, mysql:5.7, fedora, ubuntu, bash |
| Validation | JSONPath, kubectl describe, kubectl exec, rollout status, dry-run |

---

## Key Skills Demonstrated

- **Declarative manifest authoring** with heredoc patterns, dry-run gating, and idempotent `kubectl apply` workflows
- **Multi-layer post-deployment validation** using `describe`, `get all -l`, JSONPath queries, and in-container exec probes
- **Live incident diagnosis and repair** without deleting or recreating resources, preserving rollout history
- **Controlled rollbacks** with revision inspection and pod-level image uniformity verification
- **Secrets management** decoupling credentials from manifests using `secretKeyRef` and volume projection
- **Persistent storage provisioning** including static binding, StorageClass conflict resolution, and PVC state diagnosis
- **Multi-container pod patterns**: Sidecar log shipping, init container pre-initialization, shared emptyDir file exchange
- **Resource governance** with `requests`/`limits`, QoS class analysis, and scheduler-aware placement
- **Multi-tier application deployment** with DNS-based service discovery, endpoint verification, and sequential rollout gating
- **ConfigMap-driven runtime configuration** for Nginx, Redis, and PHP-FPM without image rebuilds

---

## How to Navigate

Each subdirectory contains a standalone `README.md` with:

- A problem statement and architecture diagram
- Step-by-step implementation with exact commands and expected outputs
- Screenshots from live terminal sessions
- An errors and resolutions section documenting real issues encountered
- Best practices and lessons learned sections

**Recommended reading paths:**

- **Foundations:** `kubernetes-pod-specification-and-deployment` > `pod-env-var-injection` > `multi-container-shared-emptydir-volume`
- **Deployments and Services:** `cluster-workload-orchestration` > `nginx-static-deployment` > `grafana-nodeport-deployment`
- **Fault Diagnosis:** `deployment-fault-diagnosis-redis` > `flask-deployment-misconfiguration-remediation` > `nginx-phpfpm-configmap-repair`
- **Stateful Workloads:** `persistent-storage-web-deployment` > `mysql-stateful-deployment-secrets` > `redis-configmap-deployment`
- **Advanced Patterns:** `init-containers-volume-sharing` > `sidecar-log-shipping-emptydir-nginx` > `guestbook-redis-multitier`

---

*Documented by Arinze Edeh | Cloud and DevOps Engineer | [GitHub: @arinze-edeh](https://github.com/arinze-edeh)*
