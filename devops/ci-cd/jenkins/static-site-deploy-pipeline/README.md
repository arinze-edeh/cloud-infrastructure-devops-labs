# Jenkins CI/CD Pipeline: Automated Static Website Deployment on Nautilus App Server

A production-style implementation of a Jenkins declarative pipeline that automates the deployment of a static website from a Gitea repository to an Apache-hosted application server, with a built-in validation stage to verify successful delivery.

---

## Table of Contents

- [Overview](#overview)
- [Architecture and Infrastructure](#architecture-and-infrastructure)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [1. SSH into App Server 1 from Jump Host](#1-ssh-into-app-server-1-from-jump-host)
  - [2. Update index.html and Push to Gitea](#2-update-indexhtml-and-push-to-gitea)
  - [3. Install Java 17 and Prepare Jenkins Agent Directory](#3-install-java-17-and-prepare-jenkins-agent-directory)
  - [4. Log In to Jenkins and Install Required Plugins](#4-log-in-to-jenkins-and-install-required-plugins)
  - [5. Restart Jenkins After Plugin Installation](#5-restart-jenkins-after-plugin-installation)
  - [6. Add SSH Credentials for the Jenkins Agent](#6-add-ssh-credentials-for-the-jenkins-agent)
  - [7. Register App Server 1 as a Jenkins Agent Node](#7-register-app-server-1-as-a-jenkins-agent-node)
  - [8. Verify Agent Connectivity](#8-verify-agent-connectivity)
  - [9. Create the deploy-job Pipeline](#9-create-the-deploy-job-pipeline)
  - [10. Write the Declarative Pipeline Script](#10-write-the-declarative-pipeline-script)
  - [11. Run the Pipeline and Verify Console Output](#11-run-the-pipeline-and-verify-console-output)
- [Pipeline Script Reference](#pipeline-script-reference)
- [Best Practices Applied](#best-practices-applied)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Lessons Learned](#lessons-learned)

---

## Overview

xFusionCorp Industries required a reliable automated delivery mechanism for their static website. The solution involved:

- Updating website source content in a Gitea-hosted repository
- Registering the target application server as a permanent Jenkins agent via SSH
- Authoring a declarative Jenkins pipeline with a `Deploy` stage that syncs the repository to Apache's document root and a `Test` stage that validates live accessibility via the load balancer URL

The pipeline runs entirely on the remote agent (`stapp01`), keeping execution close to the deployment target and eliminating the need for file transfer steps.

---

## Architecture and Infrastructure

| Component | Details |
|---|---|
| Jump Host | `thor@jumphost` |
| App Server 1 | `stapp01` (user: `sarah`, IP: `10.244.195.46`) |
| Source Repository | `http://gitea:3000/sarah/web.git` |
| Document Root | `/var/www/html` |
| Jenkins Agent Root | `/home/sarah/jenkins_agent` |
| Apache Port | `8080` |
| Load Balancer URL | `http://stlb01:8091` |
| Jenkins Version | `2.541.2` |
| Java Version | `OpenJDK 17.0.18` |

---

## Prerequisites

- SSH access from the jump host to `stapp01`
- Jenkins running and accessible on port `8080`
- Gitea running and accessible at `http://gitea:3000`
- Apache (`httpd`) installed and running on `stapp01`
- The `sarah/web` repository already cloned into `/var/www/html` on `stapp01`

---

## Implementation Guide

### 1. SSH into App Server 1 from Jump Host

From the jump host, establish an SSH session to `stapp01` as the `sarah` user. On first connection, accept the host key fingerprint to add the host to the known hosts file.

```bash
ssh sarah@stapp01
```

Screenshot: SSH connection from jump host to stapp01, host key fingerprint accepted and added to known hosts

<img width="1032" height="600" alt="image" src="https://github.com/user-attachments/assets/60b049c4-b150-496e-ac16-577c121ad4f6" />

---

### 2. Update index.html and Push to Gitea

After logging in, navigate to the repository directory at `/var/www/html`. Configure the Git identity for the `sarah` user, overwrite `index.html` with the required content, stage the change, commit, and push to the `master` branch on the Gitea remote.

```bash
cd /var/www/html
git config user.email "sarah@gitea.com"
git config user.name "sarah"
echo "Welcome to xFusionCorp Industries" > index.html
cat index.html
git add index.html
git commit -m "Update index.html"
git push origin master
```

Verify Apache is reachable on the app server before proceeding:

```bash
curl -s http://localhost:8080
```

Screenshot: Terminal output showing git commit, git push to Gitea master branch, and curl confirming Apache is running on localhost:8080

<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/2117cd78-b66f-4531-a812-703255202aef" />

---

### 3. Install Java 17 and Prepare Jenkins Agent Directory

The Jenkins SSH agent requires a compatible JDK on the remote host. Install `java-17-openjdk` via `yum`, verify the installed version, and create the directory that Jenkins will use as the agent workspace root.

```bash
sudo yum install -y java-17-openjdk
java -version
mkdir -p /home/sarah/jenkins_agent
exit
```

Screenshot: yum transaction output showing java-17-openjdk and java-17-openjdk-headless installed successfully

Screenshot: Continuation of yum output confirming Complete, followed by java -version output showing OpenJDK 17.0.18 build 17.0.18+8-LTS, then logout and connection closed back to jump host


<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/2117cd78-b66f-4531-a812-703255202aef" />
<img width="1036" height="832" alt="image" src="https://github.com/user-attachments/assets/3865b9f0-9e1b-47ed-ae42-6a1d606d9c3f" />
<img width="1034" height="541" alt="image" src="https://github.com/user-attachments/assets/3cee3a82-253a-400f-8e81-7e2337eb6408" />

---

### 4. Log In to Jenkins and Install Required Plugins

Open the Jenkins UI and authenticate with the `admin` account.

Screenshot: Jenkins login page with username admin entered and password field populated

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/2bb2f541-b27c-4109-9398-ef2916aaa082" />

Navigate to **Manage Jenkins > Plugins > Available plugins** and search for `credentials bindi`. Select all four of the following plugins and click **Install**:

- **Git** (5.10.1)
- **SSH Build Agents** (3.1097.v868116049892)
- **Pipeline** (608.v67378e9d3db_1)
- **Credentials Binding** (720.v3f6decef43ea_)

Screenshot: Jenkins Plugins page showing Git, SSH Build Agents, Pipeline, and Credentials Binding all selected for installation

<img width="1917" height="1013" alt="image" src="https://github.com/user-attachments/assets/f4dfe348-7c93-4098-bfa2-ceac4e299053" />

---

### 5. Restart Jenkins After Plugin Installation

Monitor the **Download progress** page until all plugin components report `Success`, then enable the **Restart Jenkins when installation is complete and no jobs are running** checkbox to trigger a safe restart.

Screenshot: Download progress page showing all plugin dependencies including commons-lang3, Pipeline Step API, SSH Credentials, Credentials Binding, Git, SSH Build Agents, Pipeline API, Pipeline Groovy, Pipeline Declarative, and all supporting libraries reporting Success

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/61104daa-9553-40f2-af73-a9226020e54f" />

Screenshot: Continuation of Download progress page showing additional pipeline components and the Credentials Binding final entry all reporting Success, with the Restart Jenkins checkbox visible at the bottom

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/b95ec107-e19a-410f-aa6d-41311c71462d" />

<img width="1917" height="1023" alt="image" src="https://github.com/user-attachments/assets/54815be9-1be3-48fe-b40e-7549f5d68d55" />

Screenshot: Jenkins restarting screen with the spinning indicator and message confirming the browser will reload automatically when Jenkins is ready

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/3489d1be-fbc1-4b92-9f2f-1499963b91ef" />

---

### 6. Add SSH Credentials for the Jenkins Agent

After Jenkins restarts, navigate to **Manage Jenkins > Credentials > System > Global credentials** and click **Add Credentials**. Configure the credential as follows:

| Field | Value |
|---|---|
| Kind | Username with password |
| Scope | Global |
| Username | `sarah` |
| Password | `Sarah_pass123` |
| ID | `sarah-stapp01` |
| Description | `Sarah - App Server 1` |

Click **Create**.

Screenshot: Add Username with password dialog showing all fields populated with sarah credentials, ID sarah-stapp01, and description Sarah - App Server 1

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/a3a78471-50f3-4952-96fd-3ec7dc5c87d5" />

Screenshot: Global credentials page confirming the sarah-stapp01 credential has been created and is listed with description Sarah - App Server 1

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/9ab55a73-fcff-462f-94aa-e251c95dbbbd" />

---

### 7. Register App Server 1 as a Jenkins Agent Node

Navigate to **Manage Jenkins > Nodes** and click **New Node**. Enter `App Server 1` as the node name, select **Permanent Agent**, and click **Create**.

Screenshot: New node dialog with App Server 1 entered as the node name and Permanent Agent selected

On the node configuration page, fill in the following fields and click **Save**:

| Field | Value |
|---|---|
| Name | `App Server 1` |
| Number of executors | `1` |
| Remote root directory | `/home/sarah/jenkins_agent` |
| Labels | `stapp01` |
| Usage | Use this node as much as possible |
| Launch method | Launch agents via SSH |
| Host | `stapp01` |
| Credentials | `sarah/****** (Sarah - App Server 1)` |
| Host Key Verification Strategy | Non verifying Verification Strategy |
| Availability | Keep this agent online as much as possible |

Screenshot: Node configuration form showing Name App Server 1, remote root directory /home/sarah/jenkins_agent, label stapp01, launch method Launch agents via SSH, and host stapp01

Screenshot: Lower portion of node configuration form showing SSH credentials set to sarah-stapp01 and Host Key Verification Strategy set to Non verifying Verification Strategy

---

### 8. Verify Agent Connectivity

Navigate to **Manage Jenkins > Nodes**, confirm `App Server 1` appears in the node list, then click the node and open its **Log** view. Confirm the agent connected successfully.

Screenshot: Nodes page showing App Server 1 listed alongside the Built-In Node

Screenshot: Agent log showing SSH connection opened to stapp01:22, authentication successful, remote environment loaded, sftp client started, remoting.jar copied (1,407,915 bytes), and agent process started at /home/sarah/jenkins_agent

Screenshot: Continuation of agent log showing the Jenkins Remoting channel started, remoting version 3352.v17a_fb_4b_2773f, SSHLauncher communication protocol confirmed, and the final line Agent successfully connected and online

---

### 9. Create the deploy-job Pipeline

From the Jenkins dashboard, click **New Item**. Enter `deploy-job` as the item name, select **Pipeline** (not Multibranch Pipeline), and click **OK**.

Screenshot: New Item page with deploy-job entered as the item name and Pipeline type selected

---

### 10. Write the Declarative Pipeline Script

In the pipeline configuration page, scroll to the **Pipeline** section. Set the **Definition** to **Pipeline script** and enter the following Groovy pipeline directly into the script editor. Click **Save**.

Screenshot: Pipeline configuration page showing the complete Groovy script in the editor with both Deploy and Test stages visible

The full pipeline script is documented in the [Pipeline Script Reference](#pipeline-script-reference) section below.

---

### 11. Run the Pipeline and Verify Console Output

From the `deploy-job` status page, click **Build Now**. Build `#1` appears in the Builds panel with a green success indicator.

Screenshot: deploy-job status page showing build number 1 completed successfully at 8:18 PM

Click on build `#1` and open **Console Output** to inspect execution. Confirm both stages completed and the final line reads `Finished: SUCCESS`.

Screenshot: Console output for build 1 showing: pipeline started by admin, running on App Server 1 in /home/sarah/jenkins_agent/workspace/deploy-job, Deploy stage executing git fetch and reset to origin/master with HEAD at a4fce20, Test stage executing curl against http://stlb01:8091 returning Welcome to xFusionCorp Industries, FILE_CONTENT matching WEB_CONTENT, Validation Success: Website matches Repository, and pipeline finishing with Finished: SUCCESS

---

## Pipeline Script Reference

```groovy
pipeline {
    agent { label 'stapp01' }

    stages {
        stage('Deploy') {
            steps {
                script {
                    def giteaUrl = "http://sarah:Sarah_pass123@gitea:3000/sarah/web.git"
                    sh """
                        echo 'Sarah_pass123' | sudo -S -H chown -R sarah:sarah /var/www/html
                        git config --global --add safe.directory /var/www/html
                        cd /var/www/html
                        git remote set-url origin ${giteaUrl}
                        git fetch origin master
                        git reset --hard origin/master
                        echo 'Sarah_pass123' | sudo -S chmod -R 755 /var/www/html
                    """
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    sh """
                        WEB_CONTENT=\$(curl -sL http://stlb01:8091)
                        FILE_CONTENT=\$(cat /var/www/html/index.html)
                        echo "Web says: \$WEB_CONTENT"
                        echo "File says: \$FILE_CONTENT"
                        if [ "\$WEB_CONTENT" == "\$FILE_CONTENT" ]; then
                            echo "Validation Success: Website matches Repository"
                        else
                            echo "Validation Failure: Website does not match Repository"
                            exit 1
                        fi
                    """
                }
            }
        }
    }
}
```

**Stage breakdown:**

**Deploy**
- Sets the Gitea remote URL with embedded credentials
- Uses `sudo chown` to ensure `sarah` owns the document root before git operations
- Marks `/var/www/html` as a safe directory to prevent Git ownership errors
- Runs `git fetch` followed by `git reset --hard origin/master` to force the working tree to match the remote state exactly, discarding any local drift
- Restores `755` permissions on the document root after the sync

**Test**
- Issues a `curl -sL` against the load balancer URL `http://stlb01:8091` to retrieve live web content
- Reads the local `index.html` from disk
- Compares both strings; if they match, prints `Validation Success` and exits cleanly; if they differ, prints `Validation Failure` and exits with code `1` to fail the build

---

## Best Practices Applied

**Agent-side execution:** The entire pipeline runs on the `stapp01` agent node rather than the built-in controller. This keeps deployment logic close to the target system and avoids unnecessary file transfers over the network.

**`git reset --hard` over `git pull`:** Using `git fetch` followed by `git reset --hard origin/master` guarantees the working directory is an exact mirror of the remote branch, even if local changes have drifted or if a previous partial deployment left the tree in an inconsistent state.

**Safe directory configuration:** Adding `/var/www/html` as a Git safe directory prevents the `dubious ownership` error that arises when the Jenkins agent process user does not match the directory owner. This is applied with `--global` scope to persist across pipeline runs.

**Credential isolation:** SSH credentials for the agent are stored as a Jenkins global credential object (`sarah-stapp01`) rather than hardcoded in the pipeline. The Gitea URL embeds credentials only within the `def giteaUrl` variable scoped to the script block.

**Non-verifying host key strategy:** For this environment, `Non verifying Verification Strategy` was selected during node registration to avoid SSH host key mismatch failures when the controller first connects to the agent. In a production environment with stricter security requirements, a known-hosts file or manually trusted key strategy should be used instead.

**Validation stage with exit code enforcement:** The `Test` stage does not merely print the curl output; it compares the live response against the file system state and explicitly calls `exit 1` on mismatch, which causes Jenkins to mark the build as failed. This ensures the pipeline provides a meaningful signal rather than a false green.

**Plugin restart discipline:** After installing plugins, Jenkins was restarted through the **Restart Jenkins when installation is complete and no jobs are running** mechanism rather than a hard kill. This is the safe restart path that allows in-progress builds to complete before the service restarts.

---

## Errors Encountered and Resolutions

**Git dubious ownership error**

When the pipeline first attempted to run git commands inside `/var/www/html`, Git rejected the operation because the directory was owned by a different user than the process executing the command.

Resolution: Added `git config --global --add safe.directory /var/www/html` as the first git operation inside the Deploy stage. This registers the directory as trusted for the duration of the agent session and eliminates the ownership check failure.

**SSH host key verification failure on agent connect**

On the first connection attempt from Jenkins controller to the `stapp01` agent, the SSH handshake failed because no known host entry existed.

Resolution: Set **Host Key Verification Strategy** to **Non verifying Verification Strategy** in the agent node configuration. This allowed the initial connection to succeed. The known hosts entry was then populated in subsequent connections.

**Jenkins UI unresponsive after restart**

After triggering the Jenkins restart from the plugin update center, the browser UI became unresponsive for approximately 30 to 60 seconds.

Resolution: Refreshed the browser tab manually after waiting for the service to fully restart. The UI returned to the login page and normal operation resumed after re-authenticating.

---

## Lessons Learned

**Matching Java version between controller and agent matters.** Jenkins remoting requires a compatible JVM on the agent. Installing `java-17-openjdk` on `stapp01` before registering the node prevented the common `NoClassDefFoundError` or `UnsupportedClassVersionError` failures that occur when the agent JVM is too old.

**`git reset --hard` is the correct deployment primitive.** A `git pull` leaves room for merge conflicts and uncommitted local changes to interfere with the deployment. Resetting hard to the remote ref is deterministic and idempotent, which are exactly the properties needed in a CI/CD deploy step.

**The Test stage should reflect production traffic, not localhost.** Curling `http://stlb01:8091` rather than `http://localhost:8080` validates that the content is reachable through the actual load balancer path that end users traverse. A localhost check would miss routing issues at the load balancer layer.

**Plugin installation order does not matter, but restart timing does.** All four plugins (Git, SSH Build Agents, Pipeline, Credentials Binding) can be selected and installed in a single batch. However, attempting to configure nodes or write pipelines before the restart completes will result in missing form fields or unrecognized pipeline syntax. Always restart before proceeding to configuration steps.

**Credential scope affects agent visibility.** Credentials must be created with **Global** scope to be selectable during agent node configuration. Credentials created with narrower scope will not appear in the SSH credentials dropdown on the node configuration page.












<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/13734f23-2fa9-496a-a4a8-e68d3f1e1bd1" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/77c0c4b5-58b6-4c12-803b-d8934fae09ea" />
<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/3ea7015a-a5e8-43fa-8396-c257ea745733" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/dc75120c-97db-4f40-bbf2-a9cce37df162" />
<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/32857dbd-8949-43d0-b7e7-dde7509add7b" />
<img width="1919" height="1008" alt="image" src="https://github.com/user-attachments/assets/741404bf-85fc-4ec2-b541-2cc86143d013" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/5403722d-0972-487d-bc2c-6bda14a719da" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/4cfb0a87-4629-4dec-a428-89ddba7c2516" />
<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/30b32547-7eaa-4df1-8f8a-7043c68d8c92" />
<img width="1915" height="1013" alt="image" src="https://github.com/user-attachments/assets/6324483c-6b85-44e3-99ec-b12be6d91563" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/9d441531-0ab9-4b18-aa62-390e23915872" />








