# Jenkins CI/CD Engineering Portfolio

**Domain:** CI/CD Infrastructure | **Platform:** Jenkins | **Environment:** Stratos Datacenter (xFusionCorp / Nautilus Labs)

---

## Overview

This directory documents a production-oriented Jenkins CI/CD engineering portfolio built across the Stratos Datacenter environment. Each project reflects a real operational scenario faced by DevOps teams: standing up automation infrastructure from scratch, enforcing least-privilege access controls, connecting distributed agent nodes, and building pipelines that deploy, validate, and maintain live web applications.

The work spans the full Jenkins lifecycle, from initial server installation through multi-node agent configuration, plugin management, parameterized job design, declarative pipeline authoring, and RBAC enforcement. All implementations follow production patterns including key-based SSH authentication, credential isolation, idempotent deployment scripts, and out-of-band verification.

---

## Directory Structure

```
jenkins/
├── agent-node-configuration/           # Multi-node SSH agent setup across 3 app servers
├── automated-db-backup-pipeline/       # Scheduled mysqldump pipeline with chained SSH transfer
├── gitea-apache-branch-deploy-pipeline/ # Parameterized branch-aware deployment to Apache via Gitea
├── jenkins-apache-log-collection-pipeline/ # Scheduled log collection from app server to storage server
├── jenkins-chained-deployment-apache-restart/ # Upstream/downstream chained build with service restart
├── jenkins-datacenter-app-deployment/  # Webhook-triggered auto-deployment with Gitea integration
├── jenkins-plugin-configuration/       # Git and GitLab plugin installation and verification
├── jenkins-rbac-configuration/         # Matrix Authorization Strategy with per-user, per-job permissions
├── jenkins-server-setup/               # Full Jenkins installation and initial configuration on Ubuntu
├── jenkins-static-site-deploy/         # Declarative pipeline deploying static site with live validation
├── job-permissions-configuration/      # Job-level RBAC scoping for multiple developer accounts
├── parameterized-job/                  # String and Choice parameter job with runtime echo validation
├── parameterized-package-install-job/  # SSH-based remote package installation via parameterized build
└── static-site-deploy-pipeline/        # Agent-pinned rsync deployment pipeline from Gitea to Apache
```

---

## Project Summaries

### [Agent Node Configuration](./agent-node-configuration/)

**Quick Summary:** Registers three AlmaLinux 9 application servers as permanent SSH build agent nodes on a Jenkins controller, establishing full key-based authentication and validating all nodes online.

**Purpose:** Enable distributed job execution across `stapp01`, `stapp02`, and `stapp03` so the Jenkins controller can offload pipeline workloads to the datacenter's application tier.

**Approach:** Generated a 4096-bit RSA key pair on the Jenkins controller, distributed the public key to all three servers via `ssh-copy-id`, installed Java 17 on each agent, and registered each node in the Jenkins UI using the SSH Build Agents plugin. Credentials are stored per-user (`tony-key`, `steve-key`, `banner-key`) to enable independent rotation and audit trails.

**Outcome:** All three agent nodes came online concurrently with sub-100ms response times. The Non-verifying host key strategy was selected deliberately for the controlled datacenter environment, with documented guidance for stricter strategies in zero-trust contexts.

---

### [Automated DB Backup Pipeline](./automated-db-backup-pipeline/)

**Quick Summary:** Freestyle job that runs every 10 minutes to dump a MySQL database on `stapp01` and transfer the timestamped file to a centralized storage server via chained SSH.

**Purpose:** Replace a manual backup process with a reliable, scheduled pipeline that produces auditable, date-stamped artifacts at `/home/natasha/db_backups/` on `ststor01`.

**Approach:** Provisioned a key pair under the Jenkins OS user, established a two-hop SSH trust chain (Jenkins controller to `stapp01`, then `stapp01` to `ststor01`), and wrote a hardened shell script with explicit exit-code guards after each critical command. Temporary dump files are cleaned from `stapp01` after transfer to prevent disk accumulation.

**Outcome:** Pipeline executes reliably on the cron schedule, produces ISO 8601 named files, and was verified end-to-end by confirming file presence and ownership on the storage server.

---

### [Gitea Apache Branch Deploy Pipeline](./gitea-apache-branch-deploy-pipeline/)

**Quick Summary:** Parameterized Jenkins pipeline that deploys either the `master` or `feature` branch from a Gitea repository directly to an Apache document root on a registered agent node.

**Purpose:** Give the development team a branch-aware deployment mechanism without requiring SSH access or manual file operations on the app server.

**Approach:** Registered `stapp01` as a permanent SSH agent with an explicit Java 17 path to resolve a version compatibility issue. The declarative pipeline uses a `BRANCH` string parameter with conditional `git checkout` and `git pull` logic. The `git safe.directory` exception and credential store entry for `sarah` were applied before the first run to avoid ownership and authentication failures.

**Outcome:** Build #2 succeeded after diagnosing a dirty working tree that blocked branch switching on Build #1. Both `master` and `feature` paths are functional and the pipeline is rerunnable without side effects.

---

### [Jenkins Apache Log Collection Pipeline](./jenkins-apache-log-collection-pipeline/)

**Quick Summary:** Freestyle job scheduled every 3 minutes that pulls Apache `access_log` and `error_log` from `stapp01` into the Jenkins workspace, then transfers both files to a centralized storage server using the Publish Over SSH plugin.

**Purpose:** Automate recurring log aggregation from the application tier to a centralized storage location for ongoing analysis and troubleshooting.

**Approach:** Implemented a two-phase transfer design: `sshpass`-driven SCP pulls logs into the Jenkins workspace (Build Step), and the Publish Over SSH post-build action deposits them at `/usr/src/itadmin` on `ststor01`. The destination directory was pre-created with correct `natasha:natasha` ownership before the Test Configuration check, a required prerequisite the plugin validates at connection time.

**Outcome:** Build #2 was triggered automatically by the cron timer 3 minutes after the manually triggered Build #1, confirming scheduled execution. Both log files were verified on the storage server with timestamps matching the build window.

---

### [Jenkins Chained Deployment and Apache Restart](./jenkins-chained-deployment-apache-restart/)

**Quick Summary:** Two linked Freestyle jobs where `datacenter-app-deployment` deploys web content from Gitea via SSH exec, then triggers `manage-services` to restart Apache only on a stable upstream build.

**Purpose:** Separate the deployment and service restart concerns into independent, auditable jobs while maintaining a strict dependency: the restart only fires when deployment succeeds.

**Approach:** Configured the Publish Over SSH global settings with the RSA private key generated on the Jenkins OS user. Both jobs reference the same `app-server-1` SSH server entry. The upstream post-build trigger and downstream "Build after other projects are built" trigger were configured on both jobs to make the relationship visible from either status page.

**Outcome:** Two consecutive builds confirmed idempotency. The `manage-services` job status page shows `datacenter-app-deployment` as the upstream project, and the application returned `Welcome to KodeKloud!` at the load balancer URL after each run.

---

### [Jenkins Datacenter App Deployment](./jenkins-datacenter-app-deployment/)

**Quick Summary:** Webhook-triggered Freestyle job that deploys the `sarah/web` repository to `stapp01`'s Apache document root on every `git push`, using a Gitea webhook and Poll SCM as a redundant trigger.

**Purpose:** Eliminate manual deployment steps by wiring Gitea push events directly to Jenkins, giving developers immediate delivery of every commit.

**Approach:** The job uses the Publish Over SSH build step to transfer workspace files to a staging directory on `stapp01`, then runs a hardened exec script (`rm`, `cp -r`, `chown`, `systemctl restart`) to atomically promote content to `/var/www/html`. The Gitea webhook embeds admin credentials in the target URL alongside the `?token=deploy` authentication token. Poll SCM at `* * * * *` provides fallback delivery if the webhook fails.

**Outcome:** Three successful builds confirmed idempotency. A first-build silent failure (files not reaching the web root despite a SUCCESS status) was diagnosed and resolved by replacing `rsync` with `cp -r` and adding a defensive `.git` directory cleanup step.

---

### [Jenkins Plugin Configuration](./jenkins-plugin-configuration/)

**Quick Summary:** Installs and verifies the Git (5.10.1) and GitLab (1.9.13) plugins on a freshly provisioned Jenkins server, completing all prerequisite steps for SCM-driven pipeline work.

**Purpose:** Establish the foundational plugin layer required before any SCM-integrated pipeline jobs can be created.

**Approach:** SSH access to the Jenkins host was used to verify the running process and home directory state before touching the UI. Plugin installation was batched to minimize restart cycles, and the Installed Plugins tab was used post-restart to confirm both plugins were active with their expected version numbers.

**Outcome:** Git and GitLab plugins confirmed installed and enabled. Health score differences between plugins (Git at 100, GitLab at 97) were documented as community rating metrics rather than functional indicators.

---

### [Jenkins RBAC Configuration](./jenkins-rbac-configuration/)

**Quick Summary:** Configures the Matrix Authorization Strategy plugin to enforce least-privilege access: `admin` retains full control, `javed` gets Overall Read globally and Job Read on a single job, and Anonymous access is fully removed.

**Purpose:** Harden a new Jenkins instance by replacing the default permissive authorization model with a scoped, auditable access control structure.

**Approach:** Updated the Bouncy Castle API plugin before installing Matrix Authorization Strategy to avoid cryptographic dependency conflicts. Added `admin` to the global matrix with Administer permission before switching the authorization strategy, preventing administrative lockout. Job-level security was enabled on the target job separately after saving the global strategy.

**Outcome:** Per-user permissions verified at both the global and job level. The Anonymous row was fully cleared, eliminating unauthenticated access to any Jenkins resource.

---

### [Jenkins Server Setup](./jenkins-server-setup/)

**Quick Summary:** End-to-end Jenkins installation on Ubuntu 24.04 from package repository registration through the Getting Started wizard, producing a fully operational instance with a named admin user.

**Purpose:** Document the complete installation procedure including a non-trivial GPG signature verification failure that is common in minimal Ubuntu environments.

**Approach:** Installed OpenJDK 17, registered the Jenkins APT repository, and resolved a persistent `NO_PUBKEY` error over three attempts. The resolution involved converting the ASCII-armored key to binary GPG format via `gpg --dearmor` and placing it in `/etc/apt/trusted.gpg.d/`. The `service` command was used explicitly to start Jenkins after the `policy-rc.d` auto-start block.

**Outcome:** Jenkins 2.541.3 running, unlocked, and configured with the `theadmin` admin account and suggested plugin set. Initial admin password rotation confirmed through the setup wizard.

---

### [Jenkins Static Site Deploy](./jenkins-static-site-deploy/)

**Quick Summary:** Declarative pipeline with a `Deploy` stage that force-syncs a Gitea repository to Apache's document root and a `Test` stage that validates live content via the load balancer URL.

**Purpose:** Provide an automated, self-validating deployment pipeline that fails the build if the deployed content does not match the repository source.

**Approach:** The `Deploy` stage uses `git fetch` plus `git reset --hard origin/master` for deterministic, conflict-free deployments. The `Test` stage curls the load balancer URL and compares the response against the local `index.html`, exiting with code 1 on mismatch. `git safe.directory` and `sudo chown` are applied within the pipeline to handle permission drift between runs.

**Outcome:** Build #1 succeeded with validation output confirming `Welcome to xFusionCorp Industries` matched between the live URL and the repository file.

---

### [Job Permissions Configuration](./job-permissions-configuration/)

**Quick Summary:** Applies per-job, per-user permissions to the `Packages` job using the Project-based Matrix Authorization Strategy, granting distinct permission sets to `sam` and `rohan` without touching any other job.

**Purpose:** Enforce granular developer access scoped to a single job, following the principle of least privilege for two newly onboarded engineers.

**Approach:** Installed the Matrix Authorization Strategy plugin, confirmed the three required user accounts existed, then switched the global authorization strategy and added `admin` with Administer rights before saving to prevent lockout. Job-level security was enabled on the `Packages` job with the `Inherit permissions from parent ACL` strategy.

**Outcome:** `sam` received Build, Configure, and Read. `rohan` received Build, Cancel, Configure, Read, Update (Run), and Tag (SCM). No other jobs were modified.

---

### [Parameterized Job](./parameterized-job/)

**Quick Summary:** Freestyle job with a String parameter (`Stage`, default: `Build`) and a Choice parameter (`env`: Development / Staging / Production) that echoes both values to the console on each run.

**Purpose:** Validate parameterized build support on a fresh Jenkins instance and demonstrate the foundational pattern for environment-targeted pipeline execution.

**Approach:** Parameterized build support was confirmed as a Jenkins core feature requiring no separate plugin. String and Choice parameters were configured with meaningful defaults (first Choice entry defaults to `Development` to prevent accidental Production runs). Parameter names were matched exactly in the shell step to avoid silent empty-variable substitution.

**Outcome:** Build #1 completed with `Stage: Build` and `env: Development` confirmed in console output.

---

### [Parameterized Package Install Job](./parameterized-package-install-job/)

**Quick Summary:** Freestyle job that accepts a `PACKAGE` string parameter at runtime and remotely executes `sudo yum install -y $PACKAGE` on the storage server via the Publish Over SSH plugin.

**Purpose:** Give operations teams a reusable, auditable mechanism to install packages on `ststor01` without requiring direct SSH access.

**Approach:** Pre-flight connectivity and credential validation was performed manually via SSH before any Jenkins configuration was attempted, ruling out network and sudo issues as variables. The SSH server entry for `natasha@ststor01` was configured in Jenkins System settings for credential centralization. The exec-only transfer step (no source files) was used, with the remote directory set to `/tmp` as a valid but non-functional placeholder.

**Outcome:** `vim-enhanced` successfully installed and verified via `rpm -q`. Multiple repeat builds confirmed idempotency, as `yum install -y` exits cleanly when the package is already present.

---

### [Static Site Deploy Pipeline](./static-site-deploy-pipeline/)

**Quick Summary:** Agent-pinned declarative pipeline that clones the `web_app` repository from an internal Gitea instance and synchronizes content to `/var/www/html` using `rsync`, running exclusively on the `stapp01` agent node.

**Purpose:** Deploy a static website from source control to Apache without any file transfer steps on the Jenkins controller, keeping execution close to the deployment target.

**Approach:** A 4096-bit RSA key pair was generated under the `sarah` user on `stapp01` and registered as an SSH credential in Jenkins. A separate `gitea-sarah` username/password credential was stored for Git authentication. The pipeline used the internal Gitea hostname (`gitea:3000`) rather than the external lab URL, which is unreachable from within the cluster network. `rsync` was installed on the agent after a command-not-found failure on Build #3.

**Outcome:** Build #4 succeeded, transferring `index.html` to `/var/www/html/` at 406 bytes/sec. Three prior builds produced diagnostic data that informed and resolved each root cause.

---

## Technologies and Tools

| Category | Technologies |
|---|---|
| CI/CD Platform | Jenkins 2.541.x (LTS) |
| Source Control | Gitea, Git 2.52, GitHub |
| Languages | Groovy (Declarative Pipeline), Bash |
| Authentication | OpenSSH RSA 4096, ssh-copy-id, ssh-keygen, JSch |
| Plugins | SSH Build Agents, Publish Over SSH, Git, GitLab, Pipeline, Credentials, Matrix Authorization Strategy, Bouncy Castle API |
| Web Server | Apache HTTP Server (httpd), port 8080 |
| Package Management | APT (Ubuntu), YUM/DNF (AlmaLinux / CentOS Stream 9) |
| Runtimes | OpenJDK 17 (agent), OpenJDK 17 JRE (controller) |
| OS Platforms | Ubuntu 24.04 LTS (controller), AlmaLinux 9 / CentOS Stream 9 (agents) |
| Data Tools | mysqldump, rsync, sshpass, scp |
| Infrastructure | Multi-node Stratos Datacenter, load balancer, storage server, jump host |

---

## Key Outcomes and Skills Demonstrated

**Infrastructure as Automation**
Stood up Jenkins from bare OS installation, resolved non-trivial GPG signing issues, and produced a fully operational CI/CD platform without relying on pre-configured environments.

**Distributed Agent Architecture**
Registered and validated multiple permanent SSH agent nodes, managed Java version compatibility, and pinned pipeline execution to specific nodes via label selectors.

**Pipeline Design Patterns**
Authored declarative pipelines with conditional branching, parameterized inputs, live validation stages, and idempotent deployment scripts that behave correctly across repeated executions.

**Security and Access Control**
Implemented project-based Matrix Authorization Strategy across both global and job-level scopes, eliminated anonymous access, and enforced least-privilege credential isolation with per-user SSH keys and Jenkins credential store entries.

**Operational Reliability**
Applied consistent patterns across all projects: exit-code guards in shell scripts, pre-flight connectivity validation, staged restarts, out-of-band verification on target servers, and documented resolutions for each failure encountered.

**Plugin Ecosystem Management**
Managed plugin dependency chains, applied updates before installing new plugins to avoid version conflicts, and used safe restarts throughout to protect in-flight builds.

---

## How to Navigate

Each subdirectory contains a `README.md` with the full implementation guide for that project, including architecture diagrams, command references, configuration screenshots, error logs, and lessons learned.

**For a specific topic, start here:**

| Goal | Recommended Project |
|---|---|
| Jenkins installation from scratch | `jenkins-server-setup` |
| Setting up distributed agent nodes | `agent-node-configuration` |
| Webhook-triggered auto-deployment | `jenkins-datacenter-app-deployment` |
| Scheduled automation with SSH | `automated-db-backup-pipeline` |
| Parameterized operational jobs | `parameterized-package-install-job` |
| RBAC and access control | `jenkins-rbac-configuration` or `job-permissions-configuration` |
| Declarative pipelines with validation | `jenkins-static-site-deploy` or `static-site-deploy-pipeline` |
| Chained build dependencies | `jenkins-chained-deployment-apache-restart` |

---

## Author

**Arinze Edeh**
Cloud Infrastructure and DevOps Engineering
[GitHub: arinze-edeh](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)
