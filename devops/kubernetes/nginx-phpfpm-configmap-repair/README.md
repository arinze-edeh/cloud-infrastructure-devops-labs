# Kubernetes Nginx and PHP-FPM Sidecar Pod: ConfigMap-Driven Repair and File Injection

## Table of Contents

* [Project Overview](#project-overview)
* [Problem Statement](#problem-statement)
* [Architecture and Design](#architecture-and-design)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Inspect and Reconfigure the Nginx ConfigMap](#step-1-inspect-and-reconfigure-the-nginx-configmap)
  * [Step 2: Delete the Broken Pod](#step-2-delete-the-broken-pod)
  * [Step 3: Recreate the Pod with Correct Volume and Container Spec](#step-3-recreate-the-pod-with-correct-volume-and-container-spec)
  * [Step 4: Verify Pod Health](#step-4-verify-pod-health)
  * [Step 5: Inject the Application File into the nginx-container](#step-5-inject-the-application-file-into-the-nginx-container)
  * [Step 6: Validate End-to-End PHP Execution via curl](#step-6-validate-end-to-end-php-execution-via-curl)
* [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [References](#references)

---

## Project Overview

This project documents the diagnosis and remediation of a broken Nginx and PHP-FPM sidecar Pod running on a Kubernetes cluster. The environment had halted functionality due to a misconfigured Nginx ConfigMap. The resolution involved reconstructing the ConfigMap with a correct nginx.conf, deleting and recreating the Pod with properly defined shared volumes and container specifications, injecting a PHP application file via `kubectl cp`, and confirming end-to-end PHP processing through an in-cluster `curl` request.

| Attribute | Detail |
|---|---|
| Platform | Kubernetes (K3s cluster) |
| Pod Name | `nginx-phpfpm` |
| ConfigMap Name | `nginx-config` |
| Containers | `nginx-container`, `php-fpm-container` |
| Pattern | Sidecar with shared `emptyDir` volume |
| Application Port | `8099` |
| Namespace | `default` |

---

## Problem Statement

The `nginx-phpfpm` Pod on the Kubernetes cluster was non-functional. Nginx was unable to serve PHP content because the existing `nginx-config` ConfigMap contained an invalid or incomplete `nginx.conf`. Specifically, the nginx server block lacked a valid `fastcgi_pass` directive pointing to the PHP-FPM process, causing all PHP requests to fail. Additionally, the application file `/home/thor/index.php` residing on the jump host had not been deployed into the nginx document root inside the container, leaving the web server with no content to serve.

---

## Architecture and Design

The solution uses the **sidecar container pattern** within a single Kubernetes Pod. Both containers share access to the same filesystem path through an `emptyDir` volume, enabling Nginx to serve PHP files that PHP-FPM processes over a local FastCGI socket.

```
+------------------------------------------------------+
|                 Pod: nginx-phpfpm                    |
|                                                      |
|  +-------------------+   +------------------------+ |
|  | php-fpm-container |   |    nginx-container     | |
|  | php:7.2-fpm-alpine|   |    nginx:latest        | |
|  |                   |   |                        | |
|  | /var/www/html  <--|---|-->  /var/www/html       | |
|  |  (shared-files)   |   |  (shared-files)        | |
|  +-------------------+   |                        | |
|                           |  /etc/nginx/nginx.conf | |
|                           |  (nginx-config-volume) | |
|                           | Listens on :8099       | |
|                           +------------------------+ |
|                                                      |
|  Volumes:                                            |
|    shared-files      --> emptyDir {}                 |
|    nginx-config-volume --> ConfigMap: nginx-config   |
+------------------------------------------------------+
```

**FastCGI flow:** Nginx receives an HTTP request on port 8099, routes `.php` requests via `fastcgi_pass` to `127.0.0.1:9000` (PHP-FPM), and returns the rendered HTML response. Because both containers share the same network namespace within the Pod, loopback communication is valid and performant.

---

## Prerequisites

* `kubectl` configured on the jump host to communicate with the target Kubernetes cluster
* Sufficient RBAC permissions to manage Pods, ConfigMaps, and execute `kubectl cp` and `kubectl exec`
* The application file `/home/thor/index.php` present on the jump host
* Access to the `default` namespace

---

## Implementation Guide

### Step 1: Inspect and Reconfigure the Nginx ConfigMap

The existing `nginx-config` ConfigMap contained an invalid Nginx configuration. The corrected configuration was applied using a heredoc-based `kubectl apply` to ensure the server block was complete, valid, and FastCGI-aware.

The nginx.conf defines:

* An `events {}` block (required by Nginx)
* An HTTP server listening on port `8099`
* A document root of `/var/www/html`
* A standard `try_files` location block for static content
* A PHP location block routing `.php` requests to PHP-FPM at `127.0.0.1:9000` via FastCGI

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 8099;
        index index.php index.html;
        root /var/www/html;
        location / {
          try_files \$uri \$uri/ =404;
        }
        location ~ \.php$ {
          include fastcgi_params;
          fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
          fastcgi_pass 127.0.0.1:9000;
        }
      }
    }
EOF
```

**Expected output:**

```
configmap/nginx-config configured
```

> **Screenshot:**

<img width="1035" height="655" alt="image" src="https://github.com/user-attachments/assets/c2ff07f6-9269-40c6-8f30-a412dedbf6d9" />

> Terminal output showing `configmap/nginx-config configured` after applying the corrected ConfigMap.

---

### Step 2: Delete the Broken Pod

The existing `nginx-phpfpm` Pod was deleted to ensure a clean state. The `--ignore-not-found` flag was used to prevent a non-zero exit code if the Pod had already been removed, enabling idempotent execution.

```bash
kubectl delete pod nginx-phpfpm --ignore-not-found
```

**Expected output:**

```
pod "nginx-phpfpm" deleted
```

> **Screenshot:**

<img width="1032" height="671" alt="image" src="https://github.com/user-attachments/assets/5e906e13-78e2-48c3-987d-b6091718d5dc" />

> Terminal confirming deletion of the `nginx-phpfpm` Pod.

---

### Step 3: Recreate the Pod with Correct Volume and Container Spec

The Pod was recreated with two containers sharing a common `emptyDir` volume at `/var/www/html`, and the corrected ConfigMap mounted as the Nginx configuration file via `subPath`.

Key design decisions embedded in the Pod spec:

* `shared-files` (`emptyDir`) is mounted at `/var/www/html` in both containers, enabling Nginx to serve files written by PHP-FPM and vice versa
* `nginx-config-volume` mounts the ConfigMap key `nginx.conf` directly to `/etc/nginx/nginx.conf` using `subPath`, overriding only the target file without replacing the entire `/etc/nginx/` directory
* The `php-fpm-container` runs `php:7.2-fpm-alpine` and binds PHP-FPM to its default port `9000`
* The `nginx-container` runs `nginx:latest` and serves traffic on port `8099`

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-phpfpm
  labels:
    app: php-app
spec:
  volumes:
  - name: shared-files
    emptyDir: {}
  - name: nginx-config-volume
    configMap:
      name: nginx-config
  containers:
  - name: php-fpm-container
    image: php:7.2-fpm-alpine
    volumeMounts:
    - name: shared-files
      mountPath: /var/www/html
  - name: nginx-container
    image: nginx:latest
    volumeMounts:
    - name: shared-files
      mountPath: /var/www/html
    - name: nginx-config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
EOF
```

**Expected output:**

```
pod/nginx-phpfpm created
```

> **Screenshot:**

<img width="1035" height="867" alt="image" src="https://github.com/user-attachments/assets/4138c35f-1ead-493e-9736-290ead0932ad" />

>  Terminal output confirming `pod/nginx-phpfpm created` after applying the Pod manifest.

---

### Step 4: Verify Pod Health

The Pod status was verified to confirm both containers reached `Running` state with zero restarts before proceeding with file injection.

```bash
kubectl get pod nginx-phpfpm
```

**Expected output:**

```
NAME           READY   STATUS    RESTARTS   AGE
nginx-phpfpm   2/2     Running   0          32s
```

`2/2` confirms both `php-fpm-container` and `nginx-container` are healthy and operational.

> **Screenshot Placeholder:** `kubectl get pod nginx-phpfpm` output showing `2/2 Running` status.

---

### Step 5: Inject the Application File into the nginx-container

The PHP application file was copied from the jump host filesystem into the `nginx-container` at the Nginx document root. The `-c nginx-container` flag was required because the target Pod runs multiple containers, and `kubectl cp` requires an explicit container name in multi-container Pods.

```bash
kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container
```

Because `shared-files` is an `emptyDir` volume mounted at `/var/www/html` in both containers, the file becomes immediately accessible to `php-fpm-container` as well, without any additional copy operation.

> **Screenshot Placeholder:** Terminal executing the `kubectl cp` command with no error output, confirming successful file transfer.

---

### Step 6: Validate End-to-End PHP Execution via curl

An in-cluster `curl` request was executed from within the `nginx-container` against `localhost:8099/index.php` to confirm that Nginx was correctly receiving the request, delegating PHP execution to PHP-FPM via FastCGI, and returning a rendered HTML response.

```bash
kubectl exec nginx-phpfpm -c nginx-container -- curl -s http://localhost:8099/index.php | head -n 5
```

**Expected output:**

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml"><head>
<style type="text/css">
body {background-color: #fff; color: #222; font-family: sans-serif;}
pre {margin: 0; font-family: monospace;}
```

The HTML output confirms the full PHP processing pipeline is functional: Nginx accepted the request, PHP-FPM executed the PHP script, and the rendered HTML was returned through FastCGI.

> **Screenshot Placeholder:** Terminal output of the `kubectl exec curl` command showing the first five lines of the rendered HTML response.

---

## Errors Encountered and Resolutions

### Error 1: Pod Not Serving PHP Content

**Symptom:** The `nginx-phpfpm` Pod was running but the website was inaccessible. HTTP requests returned errors or no response.

**Root Cause:** The `nginx-config` ConfigMap contained an invalid or incomplete `nginx.conf`. The server block was missing the `fastcgi_pass` directive or the `SCRIPT_FILENAME` FastCGI parameter, causing Nginx to be unable to delegate `.php` requests to PHP-FPM.

**Resolution:** The ConfigMap was deleted and recreated (via `kubectl apply`) with a complete and valid `nginx.conf` that included the `location ~ \.php$` block with correct `fastcgi_params`, `SCRIPT_FILENAME`, and `fastcgi_pass 127.0.0.1:9000` directives. The Pod was then deleted and recreated to pick up the updated ConfigMap.

---

### Error 2: Application File Absent from Document Root

**Symptom:** Even after fixing the ConfigMap and recreating the Pod, the web server returned a 404 because no application file existed at `/var/www/html/index.php`.

**Root Cause:** The `emptyDir` volume initializes as an empty directory on Pod creation. No persistent or pre-seeded content is available. The `index.php` file was located on the jump host and had not been transferred into the container.

**Resolution:** `kubectl cp` was used to copy `/home/thor/index.php` from the jump host into the `nginx-container` at `/var/www/html/index.php`. Because the `shared-files` volume is shared between both containers, the file became available to PHP-FPM automatically.

---

## Best Practices Applied

* **ConfigMap-driven Nginx configuration:** Externalizing `nginx.conf` into a Kubernetes ConfigMap decouples application configuration from the container image, enabling configuration changes without rebuilding or re-pulling images.

* **`subPath` volume mount for targeted file override:** Mounting the ConfigMap using `subPath: nginx.conf` ensures only the target file is replaced at `/etc/nginx/nginx.conf`, preserving all other files in `/etc/nginx/` (such as `mime.types` and `fastcgi_params`) that Nginx depends on at runtime.

* **`emptyDir` shared volume for sidecar communication:** Using an `emptyDir` volume mounted at a shared path in both containers is the idiomatic pattern for sidecar workloads in Kubernetes where containers need to exchange files within the same Pod lifecycle.

* **`--ignore-not-found` on delete:** Using `kubectl delete pod --ignore-not-found` makes the teardown step idempotent, safe for inclusion in scripts and CI pipelines without requiring pre-existence checks.

* **Explicit `-c` container flag on `kubectl cp` and `kubectl exec`:** Specifying the target container by name in multi-container Pods eliminates ambiguity and prevents accidental file injection or command execution in the wrong container.

* **In-cluster validation with `kubectl exec curl`:** Validating PHP execution from inside the `nginx-container` using `curl localhost:8099` confirms the FastCGI integration is functional independently of any external networking configuration, isolating the test to the application layer.

---

## Lessons Learned

* **ConfigMap changes do not automatically propagate to running Pods.** Kubernetes eventually syncs ConfigMap-backed volume mounts, but there is propagation delay. For configuration changes that must take effect immediately, deleting and recreating the Pod is the reliable and predictable approach in non-production environments.

* **The `subPath` mount is critical when overriding a single file within a directory.** Without `subPath`, mounting a ConfigMap to `/etc/nginx/nginx.conf` would replace the entire `/etc/nginx/` directory with only the files defined in the ConfigMap. This would cause Nginx to fail at startup due to missing `mime.types` and other required includes. `subPath` restricts the mount to only the specified file.

* **`emptyDir` volumes are ephemeral and Pod-scoped.** Files injected via `kubectl cp` into an `emptyDir` volume are lost when the Pod is deleted or restarted. In production workloads, application files should be baked into the container image or sourced from a persistent volume or init container, not injected post-creation via `kubectl cp`.

* **FastCGI communication over loopback is valid within a Pod.** Because all containers in a Pod share the same network namespace, `127.0.0.1:9000` in the Nginx `fastcgi_pass` directive correctly routes to the PHP-FPM process in the sibling container. No Service or cross-Pod networking is required for this pattern.

* **`head -n 5` on curl output is a fast triage tool.** When validating a PHP page, the full output may be verbose. Piping to `head -n 5` provides an immediate confirmation that Nginx returned an HTML response with a valid doctype, confirming the full request-to-response pipeline is functional without requiring full page inspection.

---

## References

* [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
* [Kubernetes Volumes: emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
* [kubectl cp Reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_cp/)
* [Nginx FastCGI Configuration](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)
* [PHP-FPM Official Documentation](https://www.php.net/manual/en/install.fpm.php)
* [Kubernetes Sidecar Pattern](https://kubernetes.io/docs/concepts/workloads/pods/#using-pods)







<img width="1030" height="705" alt="image" src="https://github.com/user-attachments/assets/f80ee53f-35f7-4df7-9b0a-73e72dee2fec" />
<img width="1029" height="725" alt="image" src="https://github.com/user-attachments/assets/58776dcb-ea15-4f08-9751-9c3ec0d930c5" />
<img width="1032" height="863" alt="image" src="https://github.com/user-attachments/assets/4192317e-abaa-4c88-a15b-9e71ce4d2746" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/eff67b08-bed3-4e42-8a97-db430a0cd6f0" />
