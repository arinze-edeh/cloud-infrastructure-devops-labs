# Kubernetes Deployment Troubleshooting: Flask Application on Nautilus Datacenter Cluster

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Root Cause Analysis](#root-cause-analysis)
* [Architecture and Components](#architecture-and-components)
* [Prerequisites](#prerequisites)
* [Implementation](#implementation)
  * [Step 1: Inspect the Deployment State](#step-1-inspect-the-deployment-state)
  * [Step 2: Identify the Misconfigured Container Image](#step-2-identify-the-misconfigured-container-image)
  * [Step 3: Patch the Container Image](#step-3-patch-the-container-image)
  * [Step 4: Confirm Rollout Completion](#step-4-confirm-rollout-completion)
  * [Step 5: Verify Pod Health](#step-5-verify-pod-health)
  * [Step 6: Inspect the Existing Service](#step-6-inspect-the-existing-service)
  * [Step 7: Export the Service Manifest](#step-7-export-the-service-manifest)
  * [Step 8: Edit and Correct the Service Port Configuration](#step-8-edit-and-correct-the-service-port-configuration)
  * [Step 9: Apply the Updated Service Manifest](#step-9-apply-the-updated-service-manifest)
  * [Step 10: Validate Service and Endpoint Binding](#step-10-validate-service-and-endpoint-binding)
  * [Step 11: End-to-End Application Validation](#step-11-end-to-end-application-validation)
* [Errors and Resolutions](#errors-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)

---

## Overview

This document captures the end-to-end troubleshooting and remediation of a misconfigured Python Flask application deployment running on a Kubernetes cluster within the Nautilus Datacenter environment. The deployment and service were pre-provisioned but non-functional due to two distinct configuration defects. This walkthrough demonstrates structured Kubernetes fault isolation, in-place resource patching, and manifest-driven service correction using production-grade workflows.

---

## Problem Statement

A DevOps engineer on the Nautilus team deployed a Python Flask application to the Kubernetes cluster using a pre-existing deployment named `python-deployment-datacenter` and a NodePort service named `python-service-datacenter`. Upon deployment, the application pod failed to reach a `Running` state and the application was not accessible on the designated NodePort `32345`.

Two specific requirements were mandated for remediation:

* The deployment must use the correct image: `poroko/flask-demo-app`
* The service NodePort must remain `32345`, and the targetPort must be corrected to Flask's default port `5000`

---

## Root Cause Analysis

Two independent misconfigurations were identified:

| # | Resource | Configuration Defect | Impact |
|---|----------|----------------------|--------|
| 1 | Deployment `python-deployment-datacenter` | Container image set to `poroko/flask-app-demo` (incorrect) instead of `poroko/flask-demo-app` | Pod unable to start; `0/1` replicas available |
| 2 | Service `python-service-datacenter` | Port and targetPort both set to `8080`, mismatching Flask's actual listener port `5000` | No traffic could reach the running container even after fixing the image |

Neither defect alone was sufficient to restore service. Both had to be resolved in sequence for end-to-end connectivity.

---

## Architecture and Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  Deployment: python-deployment-datacenter            │  │
│   │  Image: poroko/flask-demo-app                        │  │
│   │  Container Port: 5000                                │  │
│   │  Label: app=python_app                               │  │
│   └──────────────────────────────────────────────────────┘  │
│                           |                                 │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  Service: python-service-datacenter (NodePort)       │  │
│   │  Port: 5000  -->  TargetPort: 5000                   │  │
│   │  NodePort: 32345                                     │  │
│   │  Selector: app=python_app                            │  │
│   └──────────────────────────────────────────────────────┘  │
│                           |                                 │
│                    NodePort :32345                          │
└─────────────────────────────────────────────────────────────┘
                            |
                      External curl
                  curl http://<node-ip>:32345
```

---

## Prerequisites

* Access to the `jump-host` with `kubectl` configured against the Nautilus Kubernetes cluster
* Cluster-level permissions to update deployments and services in the `default` namespace
* Basic familiarity with Kubernetes Deployments, Services (NodePort), and rollout mechanics

---

## Implementation

### Step 1: Inspect the Deployment State

Begin by retrieving the deployment status to confirm the replica availability issue.

```bash
kubectl get deployment python-deployment-datacenter
```

**Output:**

```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
python-deployment-datacenter   0/1     1            0           4m39s
```

The `0/1` under `READY` immediately signals that no replicas are running successfully.

Screenshot: `kubectl get deployment showing 0/1 ready replicas`

<img width="1029" height="610" alt="image" src="https://github.com/user-attachments/assets/90becf41-978d-47d6-8e36-ffdbbf9936ee" />

---

### Step 2: Identify the Misconfigured Container Image

Run a full describe on the deployment to surface the pod template configuration, including the container image in use.

```bash
kubectl describe deployment python-deployment-datacenter
```

**Key output excerpt:**

```
Containers:
  python-container-datacenter:
    Image: poroko/flask-app-demo
```

**Conditions:**

```
Available      False   MinimumReplicasUnavailable
Progressing    True    ReplicaSetUpdated
```

The image `poroko/flask-app-demo` is incorrect. The required image is `poroko/flask-demo-app`. This is the primary reason the pod cannot start successfully.

Screenshot: `kubectl describe deployment output showing incorrect image and MinimumReplicasUnavailable condition`

<img width="1117" height="859" alt="image" src="https://github.com/user-attachments/assets/5f2139e9-ad1c-4540-9630-c36a3590bad9" />

---

### Step 3: Patch the Container Image

Use `kubectl set image` to update the container image in-place without modifying or reapplying the full deployment manifest.

```bash
kubectl set image deployment/python-deployment-datacenter \
  python-container-datacenter=poroko/flask-demo-app
```

**Output:**

```
deployment.apps/python-deployment-datacenter image updated
```

This triggers a rolling update, creating a new ReplicaSet with the corrected image while terminating the old pod.

Screenshot: `kubectl set image command confirming image update`

---

### Step 4: Confirm Rollout Completion

Monitor the rollout to ensure the new pod reaches a healthy state before proceeding.

```bash
kubectl rollout status deployment/python-deployment-datacenter
```

**Output:**

```
deployment "python-deployment-datacenter" successfully rolled out
```

This confirms the updated ReplicaSet has reached the desired state with `1/1` replicas available.

Screenshot: `kubectl rollout status showing successful rollout`

---

### Step 5: Verify Pod Health

Confirm the pod is in a `Running` state and no crash loops or restarts are present.

```bash
kubectl get pods -l app=python_app
```

**Output:**

```
NAME                                            READY   STATUS    RESTARTS   AGE
python-deployment-datacenter-57d654488b-j8tgf   1/1     Running   0          79s
```

The pod is fully operational with `0` restarts, confirming the image correction resolved the first defect.

Screenshot: `kubectl get pods showing 1/1 Running with 0 restarts`

---

### Step 6: Inspect the Existing Service

List all services to confirm the NodePort service exists and then describe it to inspect the current port bindings.

```bash
kubectl get svc
```

**Output:**

```
NAME                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes                  ClusterIP   10.43.0.1     <none>        443/TCP          119m
python-service-datacenter   NodePort    10.43.71.28   <none>        8080:32345/TCP   8m59s
```

Note that the service port mapping shows `8080:32345/TCP`. This means the service is listening on port `8080` internally and exposing it via NodePort `32345`. However, the Flask application listens on port `5000` internally, creating a port mismatch.

```bash
kubectl describe svc python-service-datacenter
```

**Output:**

```
Port:          <unset>  8080/TCP
TargetPort:    8080/TCP
NodePort:      <unset>  32345/TCP
Endpoints:     10.22.0.10:8080
```

The endpoint `10.22.0.10:8080` confirms traffic is being directed to port `8080` on the pod, which no application process is listening on.

Screenshot: `kubectl describe svc showing targetPort 8080 mismatched against Flask port 5000`

---

### Step 7: Export the Service Manifest

Export the live service configuration to a local YAML file for editing. This is the safest approach for patching a running service's port bindings.

```bash
kubectl get svc python-service-datacenter -o yaml > /tmp/flask-svc.yaml
```

Screenshot: `kubectl get svc -o yaml command exporting manifest to /tmp/flask-svc.yaml`

---

### Step 8: Edit and Correct the Service Port Configuration

Open the exported YAML and update both `port` and `targetPort` from `8080` to `5000`. The `nodePort` value of `32345` must remain unchanged.

```bash
vi /tmp/flask-svc.yaml
```

**Change applied inside the manifest:**

```yaml
# Before
  - port: 8080
    targetPort: 8080
    nodePort: 32345

# After
  - port: 5000
    targetPort: 5000
    nodePort: 32345
```

Screenshot: `vi editor showing the corrected port and targetPort values set to 5000`

---

### Step 9: Apply the Updated Service Manifest

Apply the corrected manifest using `kubectl apply`, which performs a declarative update against the live service object.

```bash
kubectl apply -f /tmp/flask-svc.yaml
```

**Output:**

```
service/python-service-datacenter configured
```

Screenshot: `kubectl apply confirming service/python-service-datacenter configured`

---

### Step 10: Validate Service and Endpoint Binding

Describe the service again to confirm the port correction took effect, and verify the endpoint is now resolving to port `5000`.

```bash
kubectl describe svc python-service-datacenter
```

**Output:**

```
Port:          <unset>  5000/TCP
TargetPort:    5000/TCP
NodePort:      <unset>  32345/TCP
Endpoints:     10.22.0.10:5000
```

```bash
kubectl get endpoints python-service-datacenter
```

**Output:**

```
NAME                        ENDPOINTS         AGE
python-service-datacenter   10.22.0.10:5000   16m
```

The endpoint now reflects `10.22.0.10:5000`, confirming the service selector is correctly matching the pod and routing traffic to the Flask application's actual listener port.

Screenshot: `kubectl describe svc and get endpoints confirming port 5000 binding`

---

### Step 11: End-to-End Application Validation

Perform a live HTTP request against the NodePort to confirm the Flask application is serving responses successfully.

```bash
curl http://$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}'):32345
```

**Output:**

```
Hello World Pyvo 1!
```

The application is fully accessible on NodePort `32345`. The remediation is complete.

Screenshot: `curl command returning "Hello World Pyvo 1!" confirming end-to-end connectivity`

---

## Errors and Resolutions

### Error 1: Service Describe Against Wrong Resource Name

**Command attempted:**

```bash
kubectl describe svc python-deployment-datacenter
```

**Error output:**

```
Error from server (NotFound): services "python-deployment-datacenter" not found
```

**Root cause:** The service name is `python-service-datacenter`, not `python-deployment-datacenter`. The deployment name and service name differ and cannot be used interchangeably.

**Resolution:** Ran `kubectl get svc` to list all services and identified the correct service name before re-running describe.

---

### Error 2: Pod Not Ready Due to Wrong Image

**Symptom:** `kubectl get deployment` showed `0/1` under `READY` with `MinimumReplicasUnavailable` condition.

**Root cause:** The container image `poroko/flask-app-demo` does not exist or is not the intended application image. The correct image is `poroko/flask-demo-app`.

**Resolution:** Patched the deployment image using `kubectl set image`, triggering a rolling update that successfully pulled and started the correct container.

---

### Error 3: Application Not Reachable After Pod Fix

**Symptom:** Even after the pod reached `Running` state, the application was not accessible on NodePort `32345`.

**Root cause:** The service `targetPort` was set to `8080`, while the Flask application inside the container listens on port `5000`. Traffic was being forwarded to a port with no listener.

**Resolution:** Exported the service YAML, corrected `port` and `targetPort` to `5000`, and applied the updated manifest.

---

## Best Practices Applied

* **Structured fault isolation:** Investigated deployment health before inspecting the service, isolating each defect independently before taking corrective action.

* **Non-destructive image patching:** Used `kubectl set image` to perform an in-place rolling update, avoiding full manifest deletion and re-application which can introduce downtime risk.

* **Rollout confirmation before proceeding:** Ran `kubectl rollout status` to gate progression to the next troubleshooting step, ensuring the pod was stable before validating service connectivity.

* **Label-scoped pod queries:** Used `-l app=python_app` with `kubectl get pods` to scope output to only the relevant workload, avoiding ambiguity in multi-workload namespaces.

* **Manifest export before edit:** Exported the live service to `/tmp/flask-svc.yaml` using `-o yaml` before editing, preserving the original resource metadata and avoiding manual manifest reconstruction.

* **Declarative apply for service update:** Used `kubectl apply -f` instead of `kubectl edit` or imperative patching, maintaining a file-based audit trail of the configuration change.

* **Endpoint validation:** Verified `kubectl get endpoints` after the service correction to confirm the kube-proxy resolved the pod IP and port correctly before executing the final curl test.

* **Dynamic node IP resolution:** Used `kubectl get nodes -o jsonpath` inline within the curl command to avoid hardcoding a node IP, making the validation step portable and environment-agnostic.

---

## Lessons Learned

* **Image name typos cause silent failures.** A one-character difference between `flask-app-demo` and `flask-demo-app` is enough to leave a deployment in a permanently degraded state. Always validate image names against the registry before deploying, particularly in task-driven or exam environments where images are pre-specified.

* **Pod running does not guarantee service reachability.** A pod can reach `Running` status while remaining completely unreachable if the service targetPort does not match the container's actual listener. Deployment health and service routing must be validated independently.

* **Resource naming discipline matters in Kubernetes.** Deployments and their corresponding services share similar base names but must be referenced precisely. Using `kubectl get all` or `kubectl get svc` before running describe operations prevents wasted iterations on `NotFound` errors.

* **Export-edit-apply is safer than kubectl edit for port changes.** Direct `kubectl edit` can auto-close before changes are fully validated. Exporting to a file, editing deliberately, and applying via manifest gives a reviewable intermediate state and is reproducible in automation pipelines.

* **Endpoint objects are the ground truth for service-to-pod routing.** Checking `kubectl get endpoints` after a service update is a more reliable signal than describe output alone, as it confirms that kube-proxy has registered the pod's actual IP and port as a valid backend.






<img width="1121" height="857" alt="image" src="https://github.com/user-attachments/assets/805af6e1-b99b-4076-bd1d-fd66cfedbdef" />
<img width="1117" height="861" alt="image" src="https://github.com/user-attachments/assets/63a328f8-f6ca-4385-90ef-5367c6c84400" />
<img width="1117" height="863" alt="image" src="https://github.com/user-attachments/assets/2b8cb3c4-579f-4d35-983c-ff1486dd910c" />
<img width="1124" height="340" alt="image" src="https://github.com/user-attachments/assets/41820a92-a3d0-474c-a5ed-5df46e0ab195" />
<img width="1121" height="513" alt="image" src="https://github.com/user-attachments/assets/eb6b9b0a-ae80-4545-aaa6-b120b7afb51f" />
<img width="1111" height="856" alt="image" src="https://github.com/user-attachments/assets/042cd7f6-147d-4f53-845f-7d45d2938360" />
<img width="1123" height="759" alt="image" src="https://github.com/user-attachments/assets/5fd529cd-91e9-49ed-8ea4-fdc458fce0a3" />
<img width="1118" height="864" alt="image" src="https://github.com/user-attachments/assets/03c0fffc-8302-411b-b174-a97af53819fd" />
<img width="1121" height="863" alt="image" src="https://github.com/user-attachments/assets/924bd8e0-dbb9-4a27-923e-404270e763cc" />
<img width="1121" height="781" alt="image" src="https://github.com/user-attachments/assets/9a2d064c-e564-469b-b121-97f4fb50bad8" />
<img width="1119" height="820" alt="image" src="https://github.com/user-attachments/assets/9ba3120b-f9c8-41d9-9f78-6f811da51854" />
<img width="1121" height="843" alt="image" src="https://github.com/user-attachments/assets/bab85b8f-dd75-4db4-9921-995d9905c37c" />
<img width="1123" height="558" alt="image" src="https://github.com/user-attachments/assets/6fdb603b-4165-4e9e-81bf-b115f65577eb" />
<img width="1095" height="597" alt="image" src="https://github.com/user-attachments/assets/3a07935b-b2aa-431b-8dea-144882f34d8f" />
