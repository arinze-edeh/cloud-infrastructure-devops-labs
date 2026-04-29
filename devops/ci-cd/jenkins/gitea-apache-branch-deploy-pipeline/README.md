# Jenkins CI/CD Pipeline: Conditional Branch Deployment to Apache via Gitea

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture and Context](#architecture-and-context)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1: App Server Preparation](#phase-1-app-server-preparation)
  - [Phase 2: Verifying the Web Root and Apache Status](#phase-2-verifying-the-web-root-and-apache-status)
  - [Phase 3: Resolving Git Safe Directory and Inspecting Repository](#phase-3-resolving-git-safe-directory-and-inspecting-repository)
  - [Phase 4: Confirming Apache Port and Host Network Identity](#phase-4-confirming-apache-port-and-host-network-identity)
  - [Phase 5: Jenkins Login and Plugin Update](#phase-5-jenkins-login-and-plugin-update)
  - [Phase 6: Installing SSH Build Agents and Credentials Plugins](#phase-6-installing-ssh-build-agents-and-credentials-plugins)
  - [Phase 7: Storing Sarah Credentials in Jenkins](#phase-7-storing-sarah-credentials-in-jenkins)
  - [Phase 8: Registering App Server 1 as a Jenkins Slave Node](#phase-8-registering-app-server-1-as-a-jenkins-slave-node)
  - [Phase 9: Upgrading Java to Version 17 on App Server 1](#phase-9-upgrading-java-to-version-17-on-app-server-1)
  - [Phase 10: Configuring the JavaPath on the Jenkins Agent Node](#phase-10-configuring-the-javapath-on-the-jenkins-agent-node)
  - [Phase 11: Confirming Agent Connectivity](#phase-11-confirming-agent-connectivity)
  - [Phase 12: Installing the Pipeline Plugin](#phase-12-installing-the-pipeline-plugin)
  - [Phase 13: Resolving Dirty Working Tree Before Pipeline Runs](#phase-13-resolving-dirty-working-tree-before-pipeline-runs)
  - [Phase 14: Creating the devops-webapp-job Pipeline](#phase-14-creating-the-devops-webapp-job-pipeline)
  - [Phase 15: Configuring the BRANCH Parameter and Pipeline Script](#phase-15-configuring-the-branch-parameter-and-pipeline-script)
  - [Phase 16: Build 1 Failure and Diagnosis](#phase-16-build-1-failure-and-diagnosis)
  - [Phase 17: Build 2 Success with master Branch](#phase-17-build-2-success-with-master-branch)
- [Pipeline Script](#pipeline-script)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)

---

## Project Overview

**Problem Statement**

xFusionCorp Industries is developing a new static website and requires a repeatable, automated deployment mechanism to push code from a Gitea repository onto a production Apache HTTP server running on App Server 1. The team needs the deployment to be branch-aware: passing the value `master` deploys the stable release, while passing `feature` deploys the active development branch. No manual SSH or file copy step should be involved after the pipeline is triggered.

**Solution Summary**

A Jenkins Pipeline job named `devops-webapp-job` was created with a single `string` parameter called `BRANCH`. The pipeline runs on a dedicated Jenkins slave node (`App Server 1`, labeled `stapp01`) connected via SSH. A single stage named `Deploy` contains a conditional shell block that checks the value of `BRANCH` and performs the corresponding `git checkout` and `git pull` directly against the web root at `/var/www/html`, which is both the Apache document root and the cloned Gitea repository. The pipeline is not a Multibranch pipeline.

**Outcome**

Build #2 completed with `Finished: SUCCESS`, deploying the `master` branch cleanly to the Apache server. The deployment path serves content at the load balancer URL without any subdirectory.

---

## Architecture and Context

| Component | Detail |
|---|---|
| Jenkins Controller | Port 8080, version 2.541.2 |
| Slave Node | App Server 1 (stapp01), IP 10.244.73.162 |
| Agent Root Directory | `/home/sarah/jenkins_agent` |
| Web Root / Repository | `/var/www/html` |
| Git Remote | `http://gitea:3000/sarah/web_app.git` |
| Repository Owner | sarah (Gitea user) |
| Apache Listen Port | 8080 |
| Branches | `master`, `feature` |
| Jump Host | `thor@jumphost` |
| Java on Agent | OpenJDK 17 (`/usr/lib/jvm/java-17-openjdk-17.0.18.0.8-2.el9.x86_64/bin/java`) |

---

## Prerequisites

- Jenkins 2.541.2 accessible on port 8080
- Gitea accessible on port 3000 with repository `sarah/web_app` already cloned to `/var/www/html` on App Server 1
- Apache (`httpd`) installed, enabled, and listening on port 8080
- SSH access from the jump host (`thor@jumphost`) to App Server 1 using the `tony` account with sudo privileges
- The `sarah` OS user owns `/var/www/html` and `/home/sarah/jenkins_agent`

---

## Implementation

### Phase 1: App Server Preparation

SSH into App Server 1 from the jump host as `tony`, then create and permission the Jenkins agent workspace directory for the `sarah` user.

```bash
thor@jumphost ~$ ssh tony@stapp01
```

> The host key fingerprint was verified and permanently added to known hosts on first connection.

Screenshot: SSH connection from jumphost to stapp01 with host key acceptance

<img width="1036" height="605" alt="image" src="https://github.com/user-attachments/assets/99531f59-0b08-40ad-bcaa-f455d0aa5438" />

```bash
[tony@stapp01 ~]$ sudo mkdir -p /home/sarah/jenkins_agent
sudo chown -R sarah:sarah /home/sarah/jenkins_agent
sudo chmod 755 /home/sarah/jenkins_agent
```

The directory `/home/sarah/jenkins_agent` serves as the remote root for the Jenkins agent. Setting ownership to `sarah:sarah` ensures the Jenkins process, which authenticates as `sarah`, can write workspace files without privilege escalation.

Screenshot: Creation of /home/sarah/jenkins_agent with correct ownership and permissions

<img width="1034" height="608" alt="image" src="https://github.com/user-attachments/assets/c7f45f5a-64bc-41c7-af2b-117212a54f0d" />

The pre-installed Java version was noted:

```bash
[tony@stapp01 ~]$ java -version
openjdk version "11.0.20.1" 2023-08-24 LTS
```

Java 11 was present but would later require an upgrade to Java 17 to satisfy Jenkins agent requirements.

---

### Phase 2: Verifying the Web Root and Apache Status

The existing web root was inspected to confirm the repository clone and file ownership:

```bash
[tony@stapp01 ~]$ ls /var/www/html
feature.html  index.html

[tony@stapp01 ~]$ ls -la /var/www/html
total 24
drwxr-xr-x 1 sarah sarah 4096 Apr 29 02:12 .
drwxr-xr-x 1 root  root  4096 Mar  5 14:50 ..
drwxr-xr-x 7 sarah sarah 4096 Apr 29 02:12 .git
-rw-r--r-- 1 sarah sarah   18 Apr 29 02:12 feature.html
-rw-r--r-- 1 sarah sarah    8 Apr 29 02:12 index.html
```

Ownership and permissions were set to allow the `sarah` user full access:

```bash
[tony@stapp01 ~]$ sudo chown -R sarah:sarah /var/www/html
sudo chmod -R 755 /var/www/html
ls -la /var/www/html
```

Screenshot: ls -la output showing sarah ownership and 755 permissions on /var/www/html

<img width="1026" height="758" alt="image" src="https://github.com/user-attachments/assets/14601799-0edc-4698-95e1-3e4ad1cbac2a" />

Apache status was confirmed as active and running:

```bash
[tony@stapp01 ~]$ sudo systemctl status httpd
```

```
httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
   Active: active (running) since Tue 2026-04-28 19:43:24 UTC; 6h ago
   Main PID: 1277 (httpd)
```

Screenshot: systemctl status httpd showing active (running) state with PID 1277

<img width="1036" height="864" alt="image" src="https://github.com/user-attachments/assets/15f13839-fa60-4362-9820-372e2fa09926" />

---

### Phase 3: Resolving Git Safe Directory and Inspecting Repository

Changing into `/var/www/html` and running `git remote -v` and `git branch -a` as `tony` produced an immediate error:

```bash
[tony@stapp01 ~]$ cd /var/www/html
git remote -v
git branch -a
fatal: detected dubious ownership in repository at '/var/www/html'
To add an exception for this directory, call:
        git config --global --add safe.directory /var/www/html
```

**Root Cause:** Git 2.35.2 and later enforces ownership checks. Since `tony` does not own `/var/www/html` (owned by `sarah`), Git refuses to operate on the repository.

**Resolution:** The safe directory exception was registered for both `tony` and `sarah`:

```bash
[tony@stapp01 html]$ git config --global --add safe.directory /var/www/html
```

The remote and branch configuration was then inspected successfully:

```bash
[tony@stapp01 html]$ git remote -v
git branch -a
cat .git/config
origin  http://sarah:Sarah_pass123@gitea:3000/sarah/web_app.git (fetch)
origin  http://sarah:Sarah_pass123@gitea:3000/sarah/web_app.git (push)
* feature
  master
  remotes/origin/feature
  remotes/origin/master
```

Screenshot: git remote -v and git branch -a output confirming origin URL and both local branches

<img width="1038" height="658" alt="image" src="https://github.com/user-attachments/assets/fbcce9a9-8cc7-4fc5-ab2a-e8f5e6e04d89" />

The `.git/config` confirmed the remote URL embeds credentials, and the local working copy was on the `feature` branch.

The same safe directory and credential configuration was applied for the `sarah` user:

```bash
[tony@stapp01 html]$ sudo -u sarah git config --global --add safe.directory /var/www/html
sudo -u sarah git config --global credential.helper store
```

A fetch and branch listing as `sarah` was performed to confirm:

```bash
[tony@stapp01 html]$ sudo -u sarah git -C /var/www/html fetch origin
sudo -u sarah git -C /var/www/html branch -a
* feature
  master
  remotes/origin/HEAD -> origin/master
  remotes/origin/feature
  remotes/origin/master
```

Screenshot: sudo -u sarah git fetch and branch -a confirming both branches visible with correct tracking refs

<img width="1029" height="823" alt="image" src="https://github.com/user-attachments/assets/a205afbe-a0e3-4fb8-9785-24c63f648375" />

---

### Phase 4: Confirming Apache Port and Host Network Identity

The Apache configuration was inspected to confirm the listening port:

```bash
[tony@stapp01 html]$ sudo grep -E "^Listen" /etc/httpd/conf/httpd.conf
Listen 8080
```

The server's IP address was retrieved for use in Jenkins node configuration:

```bash
[tony@stapp01 html]$ hostname -I
10.244.73.162 172.12.0.1
```

Screenshot: grep confirming Listen 8080 and hostname -I showing 10.244.73.162

<img width="1035" height="713" alt="image" src="https://github.com/user-attachments/assets/8f17b1e2-1871-4fd8-a5f3-89baeb61ffae" />

The first SSH session was then exited:

```bash
[tony@stapp01 html]$ exit
logout
Connection to stapp01 closed.
```

---

### Phase 5: Jenkins Login and Plugin Update

The Jenkins UI was accessed at the environment URL on port 8080. Login was performed using the `admin` account.

Screenshot: Jenkins Sign In page with admin credentials entered

After login, the Plugin Manager was opened at **Manage Jenkins > Plugins > Updates**. The `bouncycastle API` plugin update was pending. It was selected and the **Update** button was clicked.

Screenshot: Plugin Updates page showing bouncycastle API selected for update

The download progress page confirmed:

```
bouncycastle API - Downloaded Successfully. Will be activated during the next boot
```

Screenshot: Download progress page showing bouncycastle API downloaded successfully

Jenkins was restarted by checking **Restart Jenkins when installation is complete and no jobs are running**.

Screenshot: Jenkins restarting page showing "Jenkins is restarting" spinner

---

### Phase 6: Installing SSH Build Agents and Credentials Plugins

After Jenkins came back online, the Plugin Manager was opened at **Available plugins**. The search term `crede` was used to locate the required plugins. The following were selected for installation:

- **SSH Build Agents** (enables launching agents over SSH using a Java SSH implementation)
- **Credentials** (enables storing credentials in Jenkins)

Screenshot: Available plugins search for "crede" with SSH Build Agents and Credentials checked

The **Install** button was clicked. The download progress page showed all dependencies resolved and installed successfully:

```
EDDSA API         - Success
Gson API          - Success
Trilead API       - Success
SSH Credentials   - Success
SSH Build Agents  - Success
Credentials       - Success
Loading plugin extensions - Success
```

Screenshot: Plugin download progress showing all credential and SSH agent plugins installed successfully

Jenkins was restarted again by enabling the restart checkbox.

Screenshot: Jenkins restarting page after credential and SSH agent plugin installation

---

### Phase 7: Storing Sarah Credentials in Jenkins

With the Credentials plugin active, the global credential store was opened at **Manage Jenkins > Credentials > System > Global**. The **Add Credentials** button was clicked.

Screenshot: Add Credentials modal showing three credential type options (Username with password, SSH Username with private key, Certificate)

The credential type **Username with password** was selected and the **Next** button was clicked.

The following values were entered:

| Field | Value |
|---|---|
| Scope | Global (Jenkins, nodes, items, all child items, etc.) |
| Username | sarah |
| Password | Sarah\_pass123 |
| ID | sarah-stapp01 |
| Description | Sarah credentials for App Server 1 |

Screenshot: Add Username with password form filled with sarah credentials and ID sarah-stapp01

The **Create** button was clicked. The credential appeared in the global store:

```
sarah-stapp01  sarah/******  Sarah credentials for App Server 1
```

Screenshot: Global credentials page showing sarah-stapp01 credential listed successfully

---

### Phase 8: Registering App Server 1 as a Jenkins Slave Node

The node registration was initiated at **Manage Jenkins > Nodes > New Node**. The node name was entered as `App Server 1` and the type was set to **Permanent Agent**.

Screenshot: New node page with node name "App Server 1" and Permanent Agent selected

The **Create** button was clicked, opening the full node configuration form. The following values were entered:

| Field | Value |
|---|---|
| Name | App Server 1 |
| Number of Executors | 1 |
| Remote Root Directory | /home/sarah/jenkins\_agent |
| Labels | stapp01 |
| Usage | Use this node as much as possible |
| Launch Method | Launch agents via SSH |
| Host | 10.244.73.162 |
| Credentials | sarah/\*\*\*\*\*\* (Sarah credentials for App Server 1) |
| Host Key Verification Strategy | Non verifying Verification Strategy |

Screenshot: Node configuration form showing remote root directory /home/sarah/jenkins_agent, label stapp01, SSH launch method, host 10.244.73.162, and sarah credentials selected

The **Save** button was clicked. Immediately after saving, the agent log showed the SSH connection establishing and the remote environment being enumerated. However, the agent appeared offline in the Build Executor Status panel.

Screenshot: App Server 1 agent log showing SSH connection to 10.244.73.162:22 with successful authentication and remote environment dump

**Root Cause of Offline Status:** Jenkins attempted to launch the agent using the Java binary from the remote PATH. The system had Java 11 installed (`openjdk version "11.0.20.1"`), but Jenkins 2.541.2 requires Java 17 or later for agent processes.

---

### Phase 9: Upgrading Java to Version 17 on App Server 1

A second SSH session was opened to App Server 1 as `tony`:

```bash
thor@jumphost ~$ ssh tony@stapp01
```

Java 17 was installed via `yum`:

```bash
[tony@stapp01 ~]$ sudo yum install -y java-17-openjdk
```

The package manager downloaded and installed:

- `java-17-openjdk-1:17.0.18.0.8-2.el9.x86_64`
- `java-17-openjdk-headless-1:17.0.18.0.8-2.el9.x86_64`

Total download: 45 MB.

Screenshot: yum install java-17-openjdk output showing download progress for two packages at 33 MB/s

Screenshot: yum transaction output confirming java-17-openjdk and java-17-openjdk-headless installed successfully with "Complete!"

The Java binary path was verified:

```bash
[tony@stapp01 ~]$ find /usr/lib/jvm -name "java" -type f | grep "java-17"
/usr/lib/jvm/java-17-openjdk-17.0.18.0.8-2.el9.x86_64/bin/java
```

The system default Java was configured using `alternatives`:

```bash
[tony@stapp01 ~]$ sudo alternatives --config java
There are 2 programs which provide 'java'.
  Selection    Command
  1           java-11-openjdk.x86_64 (/usr/lib/jvm/java-11-openjdk-11.0.20.1.1-2.el9.x86_64/bin/java)
*+ 2           java-17-openjdk.x86_64 (/usr/lib/jvm/java-17-openjdk-17.0.18.0.8-2.el9.x86_64/bin/java)
Enter to keep the current selection[+], or type selection number: 2
```

The active Java version was confirmed:

```bash
[tony@stapp01 ~]$ java -version
openjdk version "17.0.18" 2026-01-20 LTS
OpenJDK Runtime Environment (Red_Hat-17.0.18.0.8-2) (build 17.0.18+8-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-17.0.18.0.8-2) (build 17.0.18+8-LTS, mixed mode, sharing)
```

Screenshot: alternatives --config java showing two selections with java-17 set as active, followed by java -version confirming 17.0.18

---

### Phase 10: Configuring the JavaPath on the Jenkins Agent Node

Back in the Jenkins UI, the App Server 1 node was opened and **Configure** was selected. Under the **Advanced** section of the SSH launch method, the **JavaPath** field was explicitly set to:

```
/usr/lib/jvm/java-17-openjdk-17.0.18.0.8-2.el9.x86_64/bin/java
```

The configuration form also confirmed:

| Field | Value |
|---|---|
| Remote Root Directory | /home/sarah/jenkins\_agent |
| Labels | stapp01 |
| Launch Method | Launch agents via SSH |
| Host | 10.244.73.162 |
| Credentials | sarah/\*\*\*\*\*\* (Sarah credentials for App Server 1) |
| Host Key Verification Strategy | Non verifying Verification Strategy |
| Port | 22 |
| JavaPath | /usr/lib/jvm/java-17-openjdk-17.0.18.0.8-2.el9.x86_64/bin/java |

Screenshot: Node configure page showing JavaPath explicitly set to the java-17-openjdk binary path under Advanced SSH settings

**Save** was clicked.

---

### Phase 11: Confirming Agent Connectivity

After saving the JavaPath configuration, the App Server 1 node status page confirmed the agent came online. The status page showed:

- Label: `stapp01`
- Build Executor Status: `0 of 1 executor busy`
- Projects tied to App Server 1: None

Screenshot: App Server 1 status page showing stapp01 label and 0 of 1 executor busy with no offline indicator

The Nodes list confirmed both nodes were online and reporting system data:

| Node | Architecture | Free Disk Space | Response Time |
|---|---|---|---|
| App Server 1 | Linux (amd64) | 342.31 GiB | 51ms |
| Built-In Node | Linux (amd64) | 333.01 GiB | 0ms |

Screenshot: Nodes list page showing App Server 1 and Built-In Node both online with disk space and response time metrics

---

### Phase 12: Installing the Pipeline Plugin

The Plugin Manager was opened at **Available plugins** and searched for `pipeline`. The top result, **Pipeline** (`608.v67378e9d3db_1`), was selected. This is the meta-plugin that installs the full Pipeline suite.

Screenshot: Available plugins search for "pipeline" with the Pipeline plugin checked for installation

The download progress page confirmed successful installation of all Pipeline plugin dependencies:

```
ASM API                                - Success
SCM API                                - Success
Pipeline: Step API                     - Success
Pipeline: API                          - Success
Script Security                        - Success
Pipeline: Supporting APIs              - Success
Durable Task                           - Success
Pipeline: Nodes and Processes          - Success
Pipeline: Build Step                   - Success
Pipeline: SCM Step                     - Success
Pipeline: Groovy                       - Success
Pipeline: Groovy Libraries             - Success
Plain Credentials                      - Success
Credentials Binding                    - Success
Pipeline: Model API                    - Success
Pipeline: Stage Step                   - Success
Pipeline: Job                          - Success
Pipeline: Declarative                  - Success
Pipeline                               - Success
Loading plugin extensions              - Success
```

Screenshot: Pipeline plugin download progress page (part 1) showing first batch of dependencies installed successfully

Screenshot: Pipeline plugin download progress page (part 2) showing remaining dependencies through Pipeline: Declarative and Pipeline all installed successfully

Jenkins was restarted after installation.

Screenshot: Jenkins restarting page after Pipeline plugin installation

---

### Phase 13: Resolving Dirty Working Tree Before Pipeline Runs

Before triggering the pipeline, the working tree in `/var/www/html` was found to have uncommitted local modifications (the `feature` branch had a dirty state). This would cause `git checkout master` to fail because Git refuses to overwrite locally modified files.

The working tree was stashed as `sarah` to produce a clean state:

```bash
[tony@stapp01 ~]$ sudo -u sarah git -C /var/www/html stash
Saved working directory and index state WIP on feature: b2340ae Added feature.html file
```

The status was verified:

```bash
[tony@stapp01 ~]$ sudo -u sarah git -C /var/www/html status
On branch feature
nothing to commit, working tree clean
```

The branch state was confirmed:

```bash
[tony@stapp01 ~]$ sudo -u sarah git -C /var/www/html branch
* feature
  master
```

Screenshot: Terminal showing git stash saving WIP, git status confirming clean working tree, and git branch showing feature as current branch

---

### Phase 14: Creating the devops-webapp-job Pipeline

In the Jenkins UI, **New Item** was clicked. The item name was entered as `devops-webapp-job` and the type **Pipeline** was selected (not Multibranch Pipeline).

Screenshot: New Item page with devops-webapp-job entered as item name and Pipeline type selected

The **OK** button was clicked to open the job configuration.

---

### Phase 15: Configuring the BRANCH Parameter and Pipeline Script

In the job configuration under **General**, the **This project is parameterized** checkbox was enabled. A **String Parameter** was added with:

| Field | Value |
|---|---|
| Name | BRANCH |
| Default Value | master |
| Description | Branch to deploy (master or feature) |

Screenshot: Job configuration showing String Parameter named BRANCH with default value master and description "Branch to deploy (master or feature)"

In the **Pipeline** section, **Definition** was set to **Pipeline script**. The following Declarative pipeline was entered in the script editor:

```groovy
pipeline {
    agent {
        label 'stapp01'
    }
    stages {
        stage('Deploy') {
            steps {
                script {
                    if (params.BRANCH == 'master') {
                        sh '''
                            cd /var/www/html
                            git checkout master
                            git pull origin master
                        '''
                    } else if (params.BRANCH == 'feature') {
                        sh '''
                            cd /var/www/html
                            git checkout feature
                            git pull origin feature
                        '''
                    } else {
                        ...
                    }
                }
            }
        }
    }
}
```

Screenshot: Pipeline script editor showing the conditional Deploy stage with master and feature branch logic targeting /var/www/html

**Save** was clicked. The pipeline job status page appeared showing no builds yet.

Screenshot: devops-webapp-job status page showing "No builds" with Build with Parameters and Configure options in the sidebar

---

### Phase 16: Build 1 Failure and Diagnosis

**Build with Parameters** was clicked. The BRANCH parameter showed the default value `master`. The **Build** button was clicked.

Screenshot: Build with Parameters page showing BRANCH parameter field pre-filled with "master"

Build #1 completed with `Finished: FAILURE`. The console output revealed:

```
Started by user admin
[Pipeline] Start of Pipeline
[Pipeline] node
Running on App Server 1 in /home/sarah/jenkins_agent/workspace/devops-webapp-job
[Pipeline] stage
[Pipeline] { (Deploy)
[Pipeline] script
[Pipeline] sh
+ cd /var/www/html
+ git checkout master
error: Your local changes to the following files would be overwritten by checkout:
        feature.html
        index.html
Please commit your changes or stash them before you switch branches.
Aborting
ERROR: script returned exit code 1
Finished: FAILURE
```

Screenshot: Build #1 console output showing the git checkout master error about local changes that would be overwritten

**Root Cause:** Although `git stash` was performed in Phase 13, the stash state may not have persisted, or the pipeline was triggered before the stash completed. The working tree still had uncommitted modifications on the `feature` branch that blocked the `git checkout master` operation.

**Resolution:** The stash was re-applied (as shown in Phase 13 terminal output), and the pipeline was triggered again for Build #2.

Build #1 appeared in the build history with a failure indicator:

Screenshot: devops-webapp-job status page showing Build #1 with red failure icon at 2:59 AM

---

### Phase 17: Build 2 Success with master Branch

**Build with Parameters** was clicked again. `master` was retained in the BRANCH field. The **Build** button was clicked.

Screenshot: Build with Parameters page showing BRANCH field with master value ready for Build #2

Build #2 completed with `Finished: SUCCESS`. The console output confirmed:

```
Started by user admin
[Pipeline] Start of Pipeline
[Pipeline] node
Running on App Server 1 in /home/sarah/jenkins_agent/workspace/devops-webapp-job
[Pipeline] stage
[Pipeline] { (Deploy)
[Pipeline] script
[Pipeline] sh
+ cd /var/www/html
+ git checkout master
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
+ git pull origin master
From http://gitea:3000/sarah/web_app
 * branch            master     -> FETCH_HEAD
Already up to date.
[Pipeline] End of Pipeline
Finished: SUCCESS
```

Screenshot: Build #2 console output showing successful git checkout master, git pull origin master from gitea, and Finished: SUCCESS

The job status page showed both builds:

- Build #2 at 3:03 AM with green success icon
- Build #1 at 2:59 AM with red failure icon

Screenshot: devops-webapp-job status page with Build #2 successful (green) and Build #1 failed (red) in build history

---

## Pipeline Script

The complete pipeline script used in `devops-webapp-job`:

```groovy
pipeline {
    agent {
        label 'stapp01'
    }
    stages {
        stage('Deploy') {
            steps {
                script {
                    if (params.BRANCH == 'master') {
                        sh '''
                            cd /var/www/html
                            git checkout master
                            git pull origin master
                        '''
                    } else if (params.BRANCH == 'feature') {
                        sh '''
                            cd /var/www/html
                            git checkout feature
                            git pull origin feature
                        '''
                    } else {
                        error("Invalid branch value: ${params.BRANCH}. Must be 'master' or 'feature'.")
                    }
                }
            }
        }
    }
}
```

Key design decisions:

* The `agent` block uses `label 'stapp01'` to pin execution to App Server 1 specifically, not any available node. This is critical because the deployment targets a file path that only exists on that host.
* The stage name `Deploy` is exact and case-sensitive per the task requirement.
* The pipeline is scoped to a single stage to keep the execution graph simple and the failure surface narrow.
* The `else` branch raises an explicit `error()` to prevent silent no-ops if an unexpected value is passed to BRANCH.

---

## Errors Encountered and Resolutions

### Error 1: Git Dubious Ownership

**Error:**
```
fatal: detected dubious ownership in repository at '/var/www/html'
```

**Root Cause:** Git 2.35.2+ blocks operations when the invoking user does not own the repository directory. `tony` attempted to run `git remote -v` inside `/var/www/html`, which is owned by `sarah`.

**Resolution:**
```bash
git config --global --add safe.directory /var/www/html
sudo -u sarah git config --global --add safe.directory /var/www/html
```

---

### Error 2: Jenkins Agent Offline After Node Creation

**Symptom:** App Server 1 appeared offline in the Jenkins Nodes panel immediately after registration.

**Root Cause:** Jenkins 2.541.2 requires Java 17 or later on the agent. The server had only Java 11 installed.

**Resolution:** Java 17 was installed via `sudo yum install -y java-17-openjdk`. The active JVM was switched using `sudo alternatives --config java`. The explicit JavaPath was then configured in the Jenkins node Advanced SSH settings to point to `/usr/lib/jvm/java-17-openjdk-17.0.18.0.8-2.el9.x86_64/bin/java`.

---

### Error 3: Build #1 Failure Due to Dirty Working Tree

**Error:**
```
error: Your local changes to the following files would be overwritten by checkout:
        feature.html
        index.html
Please commit your changes or stash them before you switch branches.
Aborting
```

**Root Cause:** The working copy at `/var/www/html` was on the `feature` branch with uncommitted local modifications. Git refuses to switch branches when the checkout would overwrite such files.

**Resolution:**
```bash
sudo -u sarah git -C /var/www/html stash
```

This saved the working directory state to the stash, leaving the tree clean. Build #2 then ran successfully.

---

## Best Practices Applied

**Principle of Least Privilege on the Agent Directory**

The Jenkins agent workspace (`/home/sarah/jenkins_agent`) was created and owned exclusively by `sarah`. The pipeline runs as `sarah` via SSH credentials, so no sudo escalation is required during pipeline execution. This isolates the build process from system-level changes.

**Explicit JavaPath Over PATH Resolution**

Rather than relying on the system PATH to resolve `java`, the full absolute path to the Java 17 binary was set in the Jenkins node Advanced configuration. This prevents version ambiguity when multiple JDKs coexist on the same host, as was the case here with both Java 11 and Java 17 installed.

**Non-Verifying Host Key Strategy with Documented Risk**

The `Non verifying Verification Strategy` was selected for the SSH connection to avoid requiring pre-distributed host keys in the Jenkins controller. The agent log itself surfaces the warning: `SSH Host Keys are not being verified. Man-in-the-middle attacks may be possible against this connection.` In a production environment, `Known hosts file Verification Strategy` or `Manually trusted key Verification Strategy` should be used with the host key pre-registered.

**Credentials Stored in Jenkins Credential Store**

Sarah's username and password were stored in the Jenkins global credential store with a meaningful ID (`sarah-stapp01`) and description rather than being embedded in the pipeline script. This prevents credential leakage in console output and enables centralized rotation.

**Single-Stage Pipeline with Parameterized Branching**

Using a single `Deploy` stage with an internal `script` block and conditional logic keeps the pipeline view clean and unambiguous. Both branch paths are handled in one stage rather than two separate stages, which avoids unnecessary skipped-stage clutter in the Pipeline visualization.

**Stashing Before Branch Switch**

Running `git stash` before pipeline execution ensured the working tree was clean and branch switches would not fail. In a production pipeline, the `git checkout` command should be prefixed with `git stash` within the shell block itself, or the pipeline should use `git clean -fd` to guarantee a deterministic state before each deployment.

---

## Lessons Learned

**Java Version Compatibility is a First-Class Concern**

Jenkins version upgrades carry implicit agent JVM requirements. Before registering any node, verify the Java version on the agent against the Jenkins compatibility matrix. Installing both Java 11 and Java 17 in parallel and setting the explicit `JavaPath` in the node configuration is safer than relying on `alternatives` alone, because `alternatives` changes the system default for all processes, not just the Jenkins agent.

**Git Safe Directory Must Be Set for the Executing User, Not Just the Interactive User**

Setting `safe.directory` as `tony` does not propagate to `sarah`. Any user that will run `git` commands against a directory they do not own must have the exception registered under their own global config. Since Jenkins runs pipeline shell steps as `sarah`, the safe directory registration under `sarah`'s global config is what matters for pipeline execution.

**Working Tree State Persists Across Pipeline Runs**

Unlike ephemeral container-based pipelines, running directly against a live web root means any uncommitted changes from a previous session remain on disk between pipeline runs. The pipeline or pre-flight setup must handle this deterministically, either by stashing, hard-resetting, or using `git clean -fd` before branch operations.

**Plugin Installation Order Matters**

The `SSH Build Agents` plugin depends on `SSH Credentials` and `Credentials`. Installing these in a single batch from the Available plugins page allows Jenkins to resolve and install all transitive dependencies automatically. Installing them separately in the wrong order can result in partially initialized plugin states that require additional restarts.

**Parameterized Pipelines Require a First Run to Register Parameter Definitions**

On a brand-new parameterized pipeline job, Jenkins does not expose the `Build with Parameters` UI until at least one build has been triggered or the job configuration has been saved. After saving the configuration, the `Build with Parameters` link appeared immediately in the sidebar, making the BRANCH parameter available before the first build.


















<img width="1034" height="714" alt="image" src="https://github.com/user-attachments/assets/1fa3a959-1ba3-4d49-8d8f-f7f526ebef73" />
<img width="1030" height="675" alt="image" src="https://github.com/user-attachments/assets/28033d22-0dda-4494-8bf0-9727e8ae4877" />
<img width="1036" height="779" alt="image" src="https://github.com/user-attachments/assets/ca815c67-e323-4fb7-90db-a3dc170af67c" />

<img width="1034" height="786" alt="image" src="https://github.com/user-attachments/assets/3d75f6e8-6aa0-4485-a3ba-db05b27fcb31" />
<img width="1038" height="477" alt="image" src="https://github.com/user-attachments/assets/b4fa032c-81f0-49a8-8ccf-a8a55943f347" />

<img width="1034" height="690" alt="image" src="https://github.com/user-attachments/assets/eb5f26b9-61c4-481a-afe2-374425a9f65b" />

<img width="1030" height="862" alt="image" src="https://github.com/user-attachments/assets/6083a6b2-f6ba-4759-90c8-d6742547928a" />

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/88cbee5a-b2ff-4569-bff2-080351fcf3e8" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/133f20aa-c19b-4625-882f-75876e9f473b" />
<img width="1918" height="1015" alt="image" src="https://github.com/user-attachments/assets/b71ec070-0854-4d1d-8b16-5325baff14f4" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/33d775c3-c4f9-4392-a454-6f855479f0a7" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/a128a080-4d9d-4dc8-9f70-3f3f1dc967d9" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/ebdc0587-887b-48fa-b0c0-01f942cb48dc" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/7193bea5-1e95-4410-9979-780778720d2e" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/ff34994c-6b77-4c67-957f-c52948aa9b76" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/297d4a5c-6ce7-40e0-acca-739a40ca48b4" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/c9fb9243-1c66-4466-8876-a1f5cf3c697c" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/e2990525-61f7-4172-9557-2a902cf1047c" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/39ea895e-6870-44ed-8f83-054947d46a25" />
<img width="1899" height="1018" alt="image" src="https://github.com/user-attachments/assets/55bfe741-2035-4d78-bca8-0bc3510919e4" />
<img width="1033" height="607" alt="image" src="https://github.com/user-attachments/assets/010ace13-a3f5-49b3-ab21-aa960a2aa175" />
<img width="1035" height="644" alt="image" src="https://github.com/user-attachments/assets/2b6ba855-5238-404f-8bba-d1b8394e6472" />
<img width="1036" height="749" alt="image" src="https://github.com/user-attachments/assets/43b27b47-ab21-479a-a393-6f8a50b31a5e" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/a9f75a49-fd93-4aea-8104-5dec61710d18" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/45c096af-e30c-40ad-b973-8cd36b0b3280" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/f4253150-89db-42db-bd79-71e2cff5aeec" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/94d6c961-34f8-4b10-8ce7-44d9428b30a5" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/2f4f4bdd-cf8b-44d8-83ea-11f218b16b2f" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/82a83c27-ba10-470e-9d50-16280d33c068" />
<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/d92c25cd-88f1-448a-b875-34f6f7d3795e" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/8e7a8415-9020-484a-8e03-3514ec8efb65" />
<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/5aaee61d-bb2c-42a8-a37c-2d54f4da33b0" />
<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/c58d9e3e-195f-4eb3-89a5-0ee6682f8d15" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/b0282ebd-b906-4a9d-82c7-0adcf66a0a4b" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/c2c72e7f-e220-4bf3-b095-ce23409a92fc" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/2a177628-0ced-4d71-8b43-c725d4c8cd6d" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/59e5266e-7ca5-4b9b-b7a8-c0b53ecb80cf" />
<img width="1035" height="576" alt="image" src="https://github.com/user-attachments/assets/53c715a6-abef-4ba0-b8f3-dc96f54512b4" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/732073c3-9be7-4c90-a656-4ac0a811a43c" />
<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/24734439-b829-4300-9706-1e5a0db0a66d" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/b5941096-95a0-4e6c-8dd9-cc0c06363676" />




















