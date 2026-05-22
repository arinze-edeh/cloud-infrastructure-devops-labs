# Docker

> Hands-on container engineering across the full Docker lifecycle: image authoring, container orchestration, network provisioning, volume management, and multi-service deployments. All tasks simulate real-world Nautilus DevOps operations within the Stratos Datacenter, following production-aligned access controls, jump-host architecture, and infrastructure hardening standards.

---

## Directory Structure

```
docker/
├── apache-service-containerization/
├── container-image-snapshotting/
├── container-runtime-provisioning/
├── containerized-httpd-service-provisioning/
├── containerized-python-app-deployment/
├── docker-compose-lamp-stack-provisioning/
├── docker-container-file-transfer/
├── docker-image-retagging-workflow/
├── docker-macvlan-network-provisioning/
├── dockerfile-fault-resolution/
├── nginx-alpine-deployment/
├── nginx-container-deployment/
└── README.md
```

---

## Project Summaries

### [Apache Service Containerization](./apache-service-containerization)

**Quick Summary:** Authored a production-ready Dockerfile on a remote server to build a custom Ubuntu 24.04 + Apache2 image reconfigured to serve on a non-standard port (6100), without modifying document root or other Apache directives.

| | |
|---|---|
| **Purpose** | Deliver a custom base image for the Nautilus development team, satisfying strict port and configuration constraints. |
| **Approach** | Used targeted `sed` in-place substitution across both `ports.conf` and `000-default.conf` within a single `RUN` layer to atomically remap the port. Combined `apt-get update` and `install` in one layer to prevent stale cache builds. |
| **Outcome** | Reproducible image with Apache2 listening on port 6100. Verified via `curl` and `docker exec ss`. Documented layer caching strategy and config patching pattern as reusable runbook. |

---

### [Container Image Snapshotting](./container-image-snapshotting)

**Quick Summary:** Captured the live filesystem state of a running container into a reusable Docker image using `docker commit`, following a jump-host access pattern with full pre- and post-commit verification.

| | |
|---|---|
| **Purpose** | Preserve developer changes made inside a running container as a named, tagged image for downstream reuse. |
| **Approach** | Confirmed container running state before commit. Used `docker commit ubuntu_latest beta:xfusion` and validated the resulting image via SHA256 digest and `docker images` output. |
| **Outcome** | Image `beta:xfusion` confirmed in local registry with correct metadata. Documented the distinction between `docker commit` for ad-hoc snapshots versus Dockerfile-based builds for reproducible pipelines. |

---

### [Container Runtime Provisioning](./container-runtime-provisioning)

**Quick Summary:** Provisioned the Docker Engine and Compose plugin on a CentOS Stream 9 host from scratch, registered the service with systemd, and validated the full runtime stack.

| | |
|---|---|
| **Purpose** | Bootstrap a containerization-ready host for the Nautilus fleet as the first step in an application modernization initiative. |
| **Approach** | Registered the official Docker CE yum repository, installed `docker-ce`, `docker-ce-cli`, `containerd.io`, and `docker-compose-plugin` in a single transaction. Enabled and started the daemon via systemd. |
| **Outcome** | Docker 29.3.0 and Compose v5.1.0 confirmed operational. Documented FQDN resolution failure in the Kubernetes-managed `/etc/hosts` environment and the short-hostname workaround as a reusable pattern. |

---

### [Containerized HTTPD Service Provisioning](./containerized-httpd-service-provisioning)

**Quick Summary:** Deployed a Docker Compose-managed Apache HTTPD container with a bind-mount volume on a remote application server, validated end-to-end via HTTP connectivity test.

| | |
|---|---|
| **Purpose** | Serve static web content via HTTPD on port 8089 with host-mounted content from `/opt/itadmin`, without modifying any existing data. |
| **Approach** | Authored `docker-compose.yml` at the canonical path `/opt/docker/`, ran `docker compose config` as a pre-flight gate, pre-pulled the image separately, then launched with `docker compose up -d`. |
| **Outcome** | Container confirmed running with correct port binding and bind-mount validated via `docker inspect` and `curl`. Established `docker compose config` as a mandatory pre-deployment step in the runbook. |

---

### [Containerized Python App Deployment](./containerized-python-app-deployment)

**Quick Summary:** Containerized a Flask application by authoring a Dockerfile against pre-staged source files in a root-owned directory, then built and deployed the image with custom port mapping.

| | |
|---|---|
| **Purpose** | Package a Python Flask app into a portable container image (`nautilus/python-app`) and expose it on host port 8095. |
| **Approach** | Used `sudo tee` to write the Dockerfile into a root-owned path, applied layer caching best practices (copy `requirements.txt` before source), and used exec-form `CMD` for proper signal handling. |
| **Outcome** | Image built successfully in 33 seconds. Container `pythonapp_nautilus` confirmed running and serving expected response via `curl`. Documented `sudo tee` as the correct privileged write pattern versus shell redirection. |

---

### [Docker Compose LAMP Stack Provisioning](./docker-compose-lamp-stack-provisioning)

**Quick Summary:** Deployed a multi-service PHP/Apache and MariaDB stack using Docker Compose with bind-mount volumes, environment-variable-based secrets, and end-to-end HTTP validation.

| | |
|---|---|
| **Purpose** | Provision a containerized two-tier application stack (frontend + database) on App Server 3 from a single Compose manifest. |
| **Approach** | Pre-created host bind-mount directories before stack launch to enforce correct ownership. Defined both services in a single `docker-compose.yml` at `/opt/sysops/`. MariaDB configured with a non-root application user and scoped environment variables. |
| **Outcome** | Both containers running with correct port bindings (8085, 3306). Web service returned expected HTML via `curl`. Documented credential exposure risk in Compose files and pointed to Docker Secrets as the production-grade alternative. |

---

### [Docker Container File Transfer](./docker-container-file-transfer)

**Quick Summary:** Securely copied a GPG-encrypted file from the host filesystem into a running container using `docker cp`, with MD5 checksum verification at both source and destination to guarantee byte-for-byte integrity.

| | |
|---|---|
| **Purpose** | Transfer confidential encrypted data into a running container without decryption, modification, or network exposure. |
| **Approach** | Captured `md5sum` of the source file before transfer. Used `docker cp` with an absolute destination path and trailing slash to preserve filename. Re-ran `md5sum` via `docker exec` inside the container post-transfer. |
| **Outcome** | All six verification checks passed: path, size, permissions, owner, timestamp, and MD5 hash. Confirmed `docker cp` operates at the block level, making it safe for encrypted payloads. |

---

### [Docker Image Retagging Workflow](./docker-image-retagging-workflow)

**Quick Summary:** Pulled `busybox:musl` from Docker Hub on a remote server and created a new tag `busybox:blog` pointing to the same image, with IMAGE ID equality used as the acceptance criterion.

| | |
|---|---|
| **Purpose** | Make a specific upstream image available under a project-specific tag for internal feature testing workflows. |
| **Approach** | Verified Docker daemon health before pull. Used `docker tag` (silent-success semantics) and followed immediately with `docker images busybox` to confirm both tags share an identical IMAGE ID. |
| **Outcome** | Both `busybox:musl` and `busybox:blog` confirmed in local registry with IMAGE ID `0188a8de47ca`. Documented digest pinning as the production-grade alternative to floating tags. |

---

### [Docker Macvlan Network Provisioning](./docker-macvlan-network-provisioning)

**Quick Summary:** Created a `macvlan`-driver Docker network named `ecommerce` on App Server 2 with a defined subnet and IP range, validated via `docker network inspect`.

| | |
|---|---|
| **Purpose** | Pre-provision a dedicated network for containerized services requiring direct Layer 2 access and custom IP addressing. |
| **Approach** | Audited existing networks before creation to confirm no name conflicts. Issued `docker network create` with explicit `--driver`, `--subnet`, and `--ip-range` flags. Validated all IPAM fields via `docker network inspect`. |
| **Outcome** | Network `ecommerce` confirmed with `macvlan` driver, correct subnet/IP range, and `local` scope. Documented the `--opt parent=<NIC>` requirement for physical environments and the port inventory discipline needed to avoid conflicts. |

---

### [Dockerfile Fault Resolution](./dockerfile-fault-resolution)

**Quick Summary:** Identified and corrected five broken Dockerfile directives (`IMAGE` instead of `FROM`, `ADD` instead of `RUN`) on a remote server, built the repaired image, and investigated a port conflict using raw `/proc` filesystem analysis when standard tools were absent.

| | |
|---|---|
| **Purpose** | Unblock a failed Docker build for an Apache HTTPD 2.4.43 image with SSL, custom port, and certificate bundling. |
| **Approach** | Used `sed -i` with anchored patterns to surgically fix only broken lines without touching valid `COPY` instructions. When `ss`, `netstat`, and `fuser` were all unavailable, resolved the port conflict owner via `/proc/net/tcp` hex lookup and a PID inode loop. |
| **Outcome** | Image `nautilus-image` built successfully across all 8 stages. Port 8080 conflict traced to `ttyd` (PID 66), the platform terminal daemon, identified as out-of-scope. Documented the full `/proc` fallback investigation pattern as a reusable diagnostic. |

---

### [Nginx Alpine Deployment](./nginx-alpine-deployment)

**Quick Summary:** Pulled `nginx:alpine` and deployed a named container in detached mode on App Server 1, confirming running state with port and image validation.

| | |
|---|---|
| **Purpose** | Deploy a lightweight nginx container (`nginx_1`) for application deployment testing across selected infrastructure nodes. |
| **Approach** | Pulled the alpine-tagged image to minimize footprint, ran with `--name` and `-d` flags, and validated via `docker ps` against all four required fields: container ID, image, status, and name. |
| **Outcome** | Container `nginx_1` confirmed running on `stapp01` with `80/tcp` exposed. Documented the jump-host gateway pattern and detached-mode requirement for SSH-based deployments. |

---

### [Nginx Container Deployment](./nginx-container-deployment)

**Quick Summary:** Deployed `nginx:stable` as a named, port-mapped container on App Server 1, validated through `docker inspect`, `docker ps`, and a live `curl` HTTP test.

| | |
|---|---|
| **Purpose** | Provision a containerized nginx web server (`ecommerce`) on host port 8084 for an e-commerce application hosting initiative. |
| **Approach** | Pulled `nginx:stable` (LTS-equivalent, preferred over `latest`), ran with `-p 8084:80` binding, and applied a three-layer verification: `docker ps` for runtime state, `docker inspect` for port binding metadata, and `curl` for application-layer response. |
| **Outcome** | Container confirmed serving nginx default HTML on `0.0.0.0:8084`. Documented restart policy recommendations, interface-scoped port binding for production hardening, and the distinction between infrastructure-layer and application-layer validation. |

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Container Runtime | Docker Engine 26.x / 29.x, containerd |
| Orchestration | Docker Compose v2 (plugin) |
| Base Images | Ubuntu 24.04, CentOS Stream 9, Alpine Linux, python:latest, php:apache, mariadb:latest, httpd:latest, nginx:stable, nginx:alpine, busybox:musl |
| Networking | Bridge, Host, Macvlan drivers |
| Web Servers | Apache2, Apache HTTPD 2.4.43, Nginx |
| Languages / Frameworks | Python, Flask, PHP |
| System Tools | systemd, sed, curl, tee, md5sum, /proc filesystem |
| Access Model | SSH jump-host architecture, sudo privilege escalation |
| OS Platforms | CentOS Stream 9, Ubuntu 24.04 |
| Version Control | Git / GitHub |

---

## Key Skills Demonstrated

**Image Engineering**
- Multi-stage Dockerfile authoring with layer caching optimization
- In-place configuration patching via `sed` without overwriting upstream files
- Privilege-aware file writes using `sudo tee` in root-owned directories

**Container Operations**
- Full container lifecycle management: pull, run, exec, commit, cp, rm
- Bind-mount and named-volume provisioning with correct pre-creation discipline
- File integrity verification using cryptographic checksums across host/container boundaries

**Networking**
- Macvlan network provisioning with custom IPAM configuration
- Port binding validation at both the runtime and metadata layers

**Orchestration**
- Multi-service Docker Compose authoring with environment-based secrets
- Pre-flight validation using `docker compose config` before deployment

**Diagnostics and Fault Resolution**
- Dockerfile linting and surgical directive correction
- Port conflict investigation via `/proc/net/tcp` inode resolution when standard tools are absent
- Systematic fallback methodology: `ss` > `netstat` > `fuser` > `/proc`

**Operational Discipline**
- Jump-host access patterns with host identity verification at every hop
- Consistent pre- and post-change validation across all tasks
- Production-aligned documentation with error catalogues, validation matrices, and lessons learned

---

## How to Navigate

Each subdirectory is a self-contained task with its own `README.md` covering:

- Problem statement and acceptance criteria
- Step-by-step resolution with commands and expected output
- Error catalogue and troubleshooting reference
- Best practices and lessons learned specific to that task

**Recommended reading order for new contributors:**

1. Start with `container-runtime-provisioning` for environment context
2. Review `apache-service-containerization` and `dockerfile-fault-resolution` for image authoring depth
3. Explore `containerized-httpd-service-provisioning` and `docker-compose-lamp-stack-provisioning` for Compose patterns
4. Reference `docker-macvlan-network-provisioning` and `docker-container-file-transfer` for networking and data handling

---

