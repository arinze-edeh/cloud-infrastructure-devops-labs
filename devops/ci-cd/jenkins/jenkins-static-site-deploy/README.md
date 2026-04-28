# Jenkins CI/CD Pipeline: Automated Static Website Deployment to Nautilus App Server

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture and Infrastructure Context](#architecture-and-infrastructure-context)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: App Server Preparation](#phase-1-app-server-preparation)
  - [Phase 2: Jenkins Login and Plugin Update](#phase-2-jenkins-login-and-plugin-update)
  - [Phase 3: Additional Plugin Installation and Restart](#phase-3-additional-plugin-installation-and-restart)
  - [Phase 4: SSH Credential Configuration for Agent](#phase-4-ssh-credential-configuration-for-agent)
  - [Phase 5: Jenkins Agent Node Registration](#phase-5-jenkins-agent-node-registration)
  - [Phase 6: Gitea Username Credential Configuration](#phase-6-gitea-username-credential-configuration)
  - [Phase 7: Gitea Repository Verification](#phase-7-gitea-repository-verification)
  - [Phase 8: Jenkins Pipeline Job Creation and Initial Configuration](#phase-8-jenkins-pipeline-job-creation-and-initial-configuration)
  - [Phase 9: Pipeline Debugging and Resolution](#phase-9-pipeline-debugging-and-resolution)
  - [Phase 10: rsync Installation and Successful Build](#phase-10-rsync-installation-and-successful-build)
- [Pipeline Script](#pipeline-script)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)

---

## Project Overview

**Problem Statement**

The development team at xFusionCorp Industries required a repeatable, automated deployment mechanism to serve a static website via the Nautilus App Server. Manual file placement was not acceptable for a production workflow. The solution needed to integrate source control (Gitea), a Jenkins controller with a remote agent, and a declarative pipeline that deployed web content directly to the Apache document root on App Server 1.

**Solution Summary**

A Jenkins declarative pipeline job named `xfusion-webapp-job` was built and configured to:

* Run exclusively on a registered Jenkins agent node (`App Server 1`, labeled `stapp01`)
* Clone the `web_app` repository from the internal Gitea instance using stored credentials
* Synchronize the checked-out workspace content to the Apache document root at `/var/www/html` using `rsync`, excluding the `.git` directory
* Serve the static website content at the load balancer root URL with no subdirectory nesting

The agent workspace is isolated at `/home/sarah/jenkins_agent` to ensure the Jenkins build workspace never pollutes the live document root directly.

---

## Architecture and Infrastructure Context

| Component | Detail |
|---|---|
| Jenkins Controller | Port 8080, accessible via browser |
| Jenkins Agent | App Server 1 (`stapp01`), IP `10.244.13.20` |
| Agent Root Directory | `/home/sarah/jenkins_agent` |
| Web Document Root | `/var/www/html` |
| Source Repository | `sarah/web_app` on internal Gitea (port 3000) |
| Repository Branch | `master` |
| Deployment Tool | `rsync` version 3.2.5 |
| Java Runtime on Agent | OpenJDK 17 (`java-17-openjdk`) |
| OS on App Server 1 | CentOS Stream 9 |
| Web Server | Apache HTTP Server, port 8080 |

---

## Prerequisites

* SSH access to App Server 1 as `tony` with sudo privileges
* Jenkins accessible at port 8080 (credentials: `admin` / `Adm!n321`)
* Gitea accessible at port 3000 (credentials: `sarah` / `Sarah_pass123`)
* The `web_app` repository already cloned under `/var/www/html` on App Server 1
* Apache already installed and running on port 8080

---

## Implementation Guide

### Phase 1: App Server Preparation

All preparation on App Server 1 was performed over SSH from the jump host before any Jenkins configuration was touched.

**Step 1.1: SSH into App Server 1**

```bash
thor@jumphost ~$ ssh tony@stapp01
```

**Step 1.2: Install Java 17 and Set as Default**

The Jenkins agent requires a compatible Java runtime. Java 17 was installed because the SSH Build Agents plugin requires a modern JVM on the remote host. Java 11 was already present, so `alternatives --config java` was run immediately after installation to make Java 17 the active selection.

```bash
sudo yum install -y java-17-openjdk
sudo alternatives --config java
```

At the alternatives prompt, two options were displayed:

```
  Selection    Command
-----------------------------------------------
   1           java-11-openjdk.x86_64 (/usr/lib/jvm/java-11-openjdk-11.0.20.1.1-2.el9.x86_64/bin/java)
*+ 2           java-17-openjdk.x86_64 (/usr/lib/jvm/java-17-openjdk-17.0.18.0.8-2.el9.x86_64/bin/java)
```

Selection `2` was entered to activate `java-17-openjdk`. Version was verified immediately after:

```bash
java -version
# openjdk version "17.0.18" 2026-01-20 LTS
# OpenJDK Runtime Environment (Red_Hat-17.0.18.0.8-2) (build 17.0.18+8-LTS)
# OpenJDK 64-Bit Server VM (Red_Hat-17.0.18.0.8-2) (build 17.0.18+8-LTS, mixed mode, sharing)
```

**Step 1.3: Create the Agent Working Directory and Set Document Root Permissions**

```bash
sudo mkdir -p /home/sarah/jenkins_agent
sudo chown -R sarah:sarah /home/sarah/jenkins_agent
sudo chown -R sarah:sarah /var/www/html
sudo chmod -R 755 /var/www/html
```

This ensures the `sarah` user, under whom the Jenkins agent process runs, has full write access to both the isolated agent workspace and the Apache document root.

**Step 1.4: Confirm the Active Git Branch in the Document Root**

```bash
sudo -u sarah git -C /var/www/html branch
# * master
```

This confirmed the repository cloned at `/var/www/html` was tracking the `master` branch.

**Step 1.5: Check the Server IP Address**

```bash
hostname -I
# 10.244.13.20 172.12.0.1
```

The primary IP `10.244.13.20` was noted for use as the Jenkins agent host address during node registration.

**Step 1.6: Generate an SSH Key Pair for the `sarah` User**

The Jenkins agent connects to App Server 1 via SSH using key-based authentication. A dedicated 4096-bit RSA key pair was generated under the `sarah` account.

```bash
sudo su - sarah
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

The private key was then read in full so its content could be pasted into the Jenkins credential form:

```bash
cat ~/.ssh/id_rsa
```

After capturing the private key output, the `sarah` session was exited to return to the `tony` user:

```bash
exit
```

**Step 1.7: Confirm the Git Remote URL**

Back as `tony`, the remote URL configured in the cloned repository was confirmed:

```bash
sudo -u sarah git -C /var/www/html remote -v
# origin  http://sarah:Sarah_pass123@gitea:3000/sarah/web_app.git (fetch)
# origin  http://sarah:Sarah_pass123@gitea:3000/sarah/web_app.git (push)
```

This confirmed the internal Gitea hostname and the repository path to be used in the pipeline.

*Screenshots: Terminal session on stapp01 showing the complete sequence: Java 17 installation output, alternatives config selection, java -version confirmation, directory creation and permission commands, git branch output showing master, hostname -I output, SSH key generation output, authorized_keys setup, cat of the private key, exit back to tony, and git remote -v output*

<img width="1035" height="859" alt="image" src="https://github.com/user-attachments/assets/9f8a8c79-cae7-4f9b-ba85-924d90c900ce" />
<img width="1031" height="862" alt="image" src="https://github.com/user-attachments/assets/1a9f9ea9-6313-43ce-900a-6742a3ca0350" />
<img width="1029" height="377" alt="image" src="https://github.com/user-attachments/assets/82ce8966-998a-433d-abac-a12caf404090" />
<img width="1031" height="448" alt="image" src="https://github.com/user-attachments/assets/3211aa85-b61e-4b60-aa96-04631d270b8e" />
<img width="1034" height="518" alt="image" src="https://github.com/user-attachments/assets/ca41f345-03c2-4f69-8111-0e4251c3a791" />
<img width="1036" height="497" alt="image" src="https://github.com/user-attachments/assets/842f88bb-d3b8-4968-97f9-40f9354e259b" />
<img width="1029" height="863" alt="image" src="https://github.com/user-attachments/assets/6ed1de45-f77b-4167-a87d-c2eff498e82b" />
<img width="1035" height="745" alt="image" src="https://github.com/user-attachments/assets/a562c024-d06e-47f5-a93e-0e473badd260" />
<img width="1055" height="882" alt="image" src="https://github.com/user-attachments/assets/2ab8bef3-c683-4a69-8268-66933355afa9" />
<img width="1033" height="382" alt="image" src="https://github.com/user-attachments/assets/76a799a3-29b1-404e-a09d-3d3d078d3a47" />

---

### Phase 2: Jenkins Login and Plugin Update

**Step 2.1: Log In to Jenkins**

The Jenkins UI was accessed at port 8080. The login page was presented and credentials were entered: username `admin`, password `Adm!n321`.

*Screenshot: Jenkins Sign in page with admin entered in the username field and password filled in*

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/2b3b5697-71d7-4994-bb0d-6b932dc0d10c" />

**Step 2.2: Navigate to Plugin Manager and Update Available Plugins**

Navigating to **Manage Jenkins > Plugins > Updates** revealed one plugin pending an update: `bouncycastle API`, currently at installed version `2.30.1.82-277.v70ca_0b_877184` with an update available to `2.30.1.84-291.v9f17b_21896e2`. The plugin was selected using its checkbox and the **Update** button was clicked.

*Screenshot: Jenkins Plugins Updates page showing the bouncycastle API plugin selected with its checkbox for update*

<img width="1919" height="1011" alt="image" src="https://github.com/user-attachments/assets/34cf9db7-d9b6-4c09-befa-e3f7af9f9206" />

**Step 2.3: Plugin Download and Restart**

The download progress page confirmed the bouncycastle API plugin downloaded successfully with the message "Downloaded Successfully. Will be activated during the next boot." The "Restart Jenkins when installation is complete and no jobs are running" checkbox was checked to trigger a safe restart.

*Screenshot: Plugin download progress page showing bouncycastle API downloaded successfully with restart Jenkins checkbox available*

<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/7a168e0a-46cf-4572-8bae-9b47a3e7e012" />

Jenkins entered its restart sequence.

*Screenshot: Jenkins restarting screen displaying "Jenkins is restarting" with the browser reload message*

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/94245ba3-90d9-4a21-9f7e-9a556ecc871f" />

---

### Phase 3: Additional Plugin Installation and Restart

After Jenkins came back online following the bouncycastle update restart, the plugins required for SSH agent connectivity, credentials management, pipeline execution, and Git integration were installed.

**Step 3.1: Search for and Select Plugins**

Navigate to **Manage Jenkins > Plugins > Available plugins** and search for `git`. Four plugins were selected using their checkboxes:

* **SSH Build Agents** - allows launching agents over SSH using a Java implementation of the SSH protocol
* **Credentials** - allows storing credentials in Jenkins for use by other plugins and pipelines
* **Pipeline** - the full suite of plugins that enables pipeline-as-code orchestration with stages, parallel work, and multi-agent support
* **Git** - integrates Git with Jenkins for source code checkout

The **Install** button was clicked to begin batch installation.

*Screenshot: Available plugins page with git searched, showing SSH Build Agents, Credentials, Pipeline, and Git all checked for installation*

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/6a7f4c7a-ea40-40f3-b71b-b16f8d197de6" />

**Step 3.2: Monitor Plugin Installation Progress**

All plugin dependencies resolved and installed successfully across multiple pages of download progress output.

First batch: EDDSA API, Gson API, Trilead API, commons-lang3 v3.x Jenkins API, Ionicons API, Structs, commons-text API, Credentials, Variant, SSH Credentials, SSH Build Agents, ASM API, SCM API, Pipeline Step API, Pipeline API, Pipeline Milestone Step — all marked Success.

*Screenshot: Plugin download progress page showing the first batch of dependencies all marked Success*

<img width="1898" height="1013" alt="image" src="https://github.com/user-attachments/assets/2a0a468f-bfcb-4d6c-8346-2d711d62051b" />

Second batch: Pipeline Milestone Step, Caffeine API, Script Security, Pipeline Supporting APIs, Durable Task, Pipeline Nodes and Processes, Pipeline Build Step, Pipeline SCM Step, Folders, Pipeline Groovy, Pipeline Groovy Libraries, Plain Credentials, Credentials Binding, Joda Time API, JAXB, JSON Api, Jackson Annotations 2 API, Jakarta Activation API, SnakeYAML API, Jakarta XML Binding API, Woodstox Core API — all marked Success.

*Screenshot: Plugin download progress page showing the second batch of dependencies all marked Success*

<img width="1894" height="1020" alt="image" src="https://github.com/user-attachments/assets/b17d40d4-0256-41ff-bd2c-1bb1cf9c70ff" />

Third batch: Jackson Annotations 2 API, Jakarta Activation API, SnakeYAML API, Jakarta XML Binding API, Woodstox Core API, Jackson 2 API, Pipeline Model API, Pipeline Stage Step, Pipeline Job, Pipeline Declarative Extension Points API, Jakarta Mail API, Display URL API, Mailer, Branch API, Pipeline Multibranch, Pipeline Stage Tags Metadata, Pipeline Input Step, Apache HttpComponents Client 4.x API, Pipeline Basic Steps, Pipeline Declarative, Pipeline — all marked Success.

*Screenshot: Plugin download progress page showing the third batch of dependencies all marked Success*

<img width="1893" height="1015" alt="image" src="https://github.com/user-attachments/assets/18d4ee1c-fec0-41ec-bf75-e6e91d166c55" />

Final batch: Mina SSHD API Common, Mina SSHD API Core, Git client, Git, Loading plugin extensions — all marked Success. The "Restart Jenkins when installation is complete and no jobs are running" checkbox was checked.

*Screenshot: Plugin download progress final page showing Mina SSHD API Common, Mina SSHD API Core, Git client, Git, and Loading plugin extensions all marked Success with restart option available*

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/cd20a89e-bd91-479c-bc68-341ec05604ae" />

Jenkins entered its second restart cycle.

*Screenshot: Jenkins restarting screen after the full plugin suite installation*

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/61745b6e-c8dd-41c9-ad4e-f28ce7c427cd" />

---

### Phase 4: SSH Credential Configuration for Agent

After Jenkins restarted and came back online, the first credential was added: the SSH private key for the `sarah` user on App Server 1. This credential is used by the Jenkins controller to open the SSH connection to the agent node.

**Step 4.1: Navigate to Credentials and Open Add Credentials**

Navigate to **Manage Jenkins > Credentials > System > Global credentials (unrestricted)** and click **Add Credentials**. The Add Credentials dialog appeared showing the available credential type options.

**Step 4.2: Select SSH Username with Private Key**

**SSH Username with private key** was selected (highlighted in blue) and **Next** was clicked.

*Screenshot: Add Credentials dialog with SSH Username with private key option highlighted and selected*

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/a57c98ff-eae1-4eb1-8753-ca46bcd7a8b5" />

**Step 4.3: Fill in SSH Credential Details**

The SSH credential form was completed with the following values:

* **ID:** `sarah-stapp01`
* **Description:** (left blank)
* **Username:** `sarah`
* **Private Key:** Enter directly was selected. The full content of `/home/sarah/.ssh/id_rsa`, including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` headers and all key body lines, was pasted into the Key field.
* **Passphrase:** (left blank, as the key was generated without a passphrase using `-N ""`)

The **Create** button was clicked.

*Screenshot: Add SSH Username with private key form showing ID as sarah-stapp01, username as sarah, Enter directly selected, and the private key content pasted into the Key text area with the BEGIN and END markers visible*

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/7359943e-47e8-499b-b68a-725edfac5865" />

**Step 4.4: Verify SSH Credential Saved**

The Global credentials page confirmed the `sarah-stapp01` SSH credential was saved and appeared in the credentials list.

*Screenshot: Global credentials page showing the sarah-stapp01 SSH credential entry for user sarah*

<img width="1919" height="1011" alt="image" src="https://github.com/user-attachments/assets/1cba644a-1fdc-449b-909f-00b29582c7cd" />

---

### Phase 5: Jenkins Agent Node Registration

With the SSH credential in place, the App Server 1 agent node was registered in Jenkins.

**Step 5.1: Create a New Node**

Navigate to **Manage Jenkins > Nodes > New Node**.

* **Node name:** `App Server 1`
* **Type:** Permanent Agent

The **Create** button was clicked.

*Screenshot: New node page with App Server 1 entered as the node name and Permanent Agent type selected*

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/7afae84c-aef0-4aed-84ec-78ec63a3f7e6" />

**Step 5.2: Configure the Agent Node**

On the node configuration page, all required fields were filled in:

| Field | Value |
|---|---|
| Name | `App Server 1` |
| Remote root directory | `/home/sarah/jenkins_agent` |
| Labels | `stapp01` |
| Usage | Use this node as much as possible |
| Launch method | Launch agents via SSH |
| Host | `10.244.13.20` |
| Credentials | `sarah` (the `sarah-stapp01` SSH credential) |
| Host Key Verification Strategy | Non verifying Verification Strategy |

The **Save** button was clicked.

*Screenshot: Agent node configuration page showing all fields populated: name App Server 1, remote root directory /home/sarah/jenkins_agent, label stapp01, Launch agents via SSH selected, host 10.244.13.20, sarah credential selected, and Non verifying Verification Strategy set*

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/27dae380-5bca-402c-9733-9f79d55e66c9" />

**Step 5.3: Verify Agent Connectivity**

After saving, the Nodes list confirmed `App Server 1` came online alongside the Built-In Node. Both nodes showed architecture as Linux (amd64), clock in sync, available disk space, and response time data, confirming the SSH connection from the Jenkins controller to App Server 1 was established successfully.

*Screenshot: Jenkins Nodes page showing both App Server 1 and Built-In Node listed as online with architecture, disk space, and response time columns populated*

<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/b10af8ee-2723-45ca-a8bb-17350c75684d" />

---

### Phase 6: Gitea Username Credential Configuration

After the agent node was confirmed online, the second credential was added: the Gitea username and password for `sarah`. This credential is used by the pipeline's `git` step to authenticate against the Gitea server when cloning the `web_app` repository during each build.

**Step 6.1: Open Add Credentials Again**

From **Manage Jenkins > Credentials > System > Global credentials (unrestricted)**, the **Add Credentials** button was clicked. The credential type selection dialog appeared again.

*Screenshot: Add Credentials dialog appearing with Username with password, SSH Username with private key, Secret file, Secret text, and Certificate options listed*

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/f5730c83-c235-40e6-b680-185c642047f9" />

**Step 6.2: Select Username with Password**

**Username with password** was selected and **Next** was clicked.

**Step 6.3: Fill in Gitea Credential Details**

The form was completed with the following values:

* **Scope:** Global (Jenkins, nodes, items, all child items, etc.)
* **Username:** `sarah`
* **Password:** `Sarah_pass123`
* **ID:** `gitea-sarah`
* **Description:** (left blank)

The **Create** button was clicked.

*Screenshot: Add Username with password form showing scope as Global, username as sarah, password entered, and ID as gitea-sarah*

<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/9323cf00-5b6b-4c15-8980-2d93ac0a9e83" />

**Step 6.4: Verify Both Credentials Are Present**

The Global credentials page now listed both credentials:

* `sarah-stapp01` (SSH Username with private key for `sarah`)
* `gitea-sarah` (Username with password for `sarah`)

*Screenshot: Global credentials page listing both sarah-stapp01 SSH credential and gitea-sarah username/password credential*

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/d85789ce-dbc6-4048-8e81-ce3a32c9aa89" />

---

### Phase 7: Gitea Repository Verification

With credentials configured and the agent confirmed online, the Gitea repository was inspected to confirm its contents and structure before writing the pipeline.

**Step 7.1: Sign In to Gitea**

The Gitea UI was accessed at port 3000. Username `sarah` and password `Sarah_pass123` were entered and **Sign In** was clicked.

*Screenshot: Gitea Sign In page with sarah entered as the username and password filled in*

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/227feaca-032f-47da-a083-ddf150cd26af" />

**Step 7.2: Review the Dashboard and Confirm the Repository**

On the Gitea dashboard for `sarah`, the activity feed confirmed:

* `sarah` created repository `sarah/web_app` 26 minutes ago
* `sarah` pushed to `master` at `sarah/web_app` 26 minutes ago with commit `d81ae80e64` "Added index.html file"
* `sarah` created branch `master` in `sarah/web_app` 26 minutes ago

The Repositories sidebar confirmed `sarah/web_app` as the only repository under this account.

*Screenshot: Gitea dashboard for sarah showing the activity feed with web_app repository creation, push, and branch creation events, and web_app listed in the repositories sidebar*

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/5cb0a45e-974a-4758-ba29-4690522979e0" />

**Step 7.3: Inspect the Repository Contents**

Navigating into `sarah/web_app` confirmed the repository contained one file: `index.html`, committed 27 minutes prior with the message "Added index.html file". The repository was on the `master` branch, had 1 commit, 1 branch, 0 tags, and the codebase was 100% HTML at 23 KiB.

*Screenshot: sarah/web_app repository page on Gitea showing index.html as the only file on the master branch with 1 commit*

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/7975919f-d801-4baf-9c13-50146c4b68a1" />

---

### Phase 8: Jenkins Pipeline Job Creation and Initial Configuration

**Step 8.1: Create the Pipeline Job**

From the Jenkins dashboard, **New Item** was selected.

* **Item name:** `xfusion-webapp-job`
* **Type:** **Pipeline** (the standard Pipeline type, explicitly not Multibranch Pipeline)

The **OK** button was clicked.

*Screenshot: New Item page with xfusion-webapp-job entered as the item name and Pipeline type selected*

**Step 8.2: Configure the Pipeline Script**

On the job configuration page, the **Pipeline** section was opened. The **Definition** was left as **Pipeline script** and the following Jenkinsfile was entered in the Script editor. At this stage, the external Gitea URL (the same URL used in the browser to access Gitea from outside the cluster) was used in the `url` field:

```groovy
pipeline {
    agent {
        label 'stapp01'
    }

    stages {
        stage('Deploy') {
            steps {
                git branch: 'master',
                    credentialsId: 'gitea-sarah',
                    url: 'http://3000-port-yfmswuop6s46fvp5.labs.kodekloud.com/sarah/web_app.git'

                sh '''
                    rsync -av --delete --exclude='.git' \
                    ${WORKSPACE}/ /var/www/html/
                '''
            }
        }
    }
}
```

**Use Groovy Sandbox** was checked. The **Save** button was clicked.

*Screenshot: Pipeline configuration page showing the Groovy script with agent label stapp01, the Deploy stage, git step referencing gitea-sarah credential and the external Gitea URL, and the rsync shell command*

**Step 8.3: Initial Job Status Page**

After saving, the `xfusion-webapp-job` status page was displayed with no builds yet recorded and an empty Permalinks section.

*Screenshot: xfusion-webapp-job status page showing no builds yet and empty Permalinks section*

The first build was triggered using **Build Now**.

---

### Phase 9: Pipeline Debugging and Resolution

#### Error 1: Git Clone Failure via External Gitea URL (Build #1)

**Symptom**

Build #1 was dispatched to App Server 1 and ran at `/home/sarah/jenkins_agent/workspace/xfusion-webapp-job`. The pipeline entered the `Deploy` stage and the `git` step attempted to clone from the external URL. The attempt failed immediately with a network error:

```
Running on App Server 1 in /home/sarah/jenkins_agent/workspace/xfusion-webapp-job
[Pipeline] { (Deploy)
[Pipeline] git
The recommended git tool is: NONE
using credential gitea-sarah
Cloning the remote Git repository
ERROR: Error cloning remote repo 'origin'
stderr: fatal: unable to access
'http://3000-port-yfmswuop6s46fvp5.labs.kodekloud.com/sarah/web_app.git/':
Recv failure: Connection reset by peer
```

*Screenshot: Build #1 console output top section showing the pipeline running on App Server 1, entering the Deploy stage, and the git clone failing with "Recv failure: Connection reset by peer"*

The console continued with the full stack trace from the Git client plugin and concluded:

```
ERROR: Error cloning remote repo 'origin'
ERROR: Maximum checkout retry attempts reached, aborting
Finished: FAILURE
```

*Screenshot: Build #1 console output bottom section showing the stack trace tail, the repeated ERROR lines, and Finished: FAILURE*

**Root Cause**

The pipeline script referenced the external browser-facing URL for the Gitea instance. When the pipeline ran on App Server 1 (the agent), this external hostname was unreachable from within the cluster network. The agent needed to reach Gitea via the internal service hostname.

**Resolution**

The pipeline configuration was reopened and the Gitea URL was corrected from the external hostname to the internal cluster hostname:

```groovy
url: 'http://gitea:3000/sarah/web_app.git'
```

*Screenshot: Pipeline configure page showing the updated Groovy script with the corrected internal URL http://gitea:3000/sarah/web_app.git replacing the external URL on line 11*

The job was saved and additional builds were triggered.

#### Error 2: rsync Binary Not Found on Agent (Build #3)

**Symptom**

With the corrected internal Gitea URL, build #3 successfully authenticated against Gitea using the `gitea-sarah` credential and cloned the repository, checking out revision `d81ae80e6413ca40b5a928ae4308b8bdaeb0454a` from `master`. However, the subsequent `sh` step that invoked `rsync` failed with exit code 127:

```
[Pipeline] sh
+ rsync -av --delete --exclude=.git
  /home/sarah/jenkins_agent/workspace/xfusion-webapp-job/ /var/www/html/
/home/sarah/jenkins_agent/workspace/xfusion-webapp-job@tmp/durable-627c25cc/script.sh.copy:
line 2: rsync: command not found
[Pipeline] End of Pipeline
ERROR: script returned exit code 127
Finished: FAILURE
```

*Screenshot: Build #3 console output showing the git checkout of master succeeding with commit d81ae80e64, then the rsync sh step failing with "rsync: command not found" and exit code 127*

**Root Cause**

`rsync` is not included in the default CentOS Stream 9 installation. The agent's shell environment could not locate the binary, and the shell returned exit code 127 (command not found).

**Resolution**

`rsync` was installed on App Server 1 by returning to the terminal session as `tony`:

```bash
[tony@stapp01 ~]$ sudo yum install -y rsync
```

The package `rsync-3.2.5-5.el9.x86_64` was downloaded from the `baseos` repository (407 kB) and installed successfully.

*Screenshot: Terminal output on stapp01 showing the yum install rsync transaction: package details, download, installation, and "Complete!" confirmation*

Installation was verified:

```bash
[tony@stapp01 ~]$ rsync --version
# rsync  version 3.2.5  protocol version 31
```

*Screenshot: Terminal output showing rsync --version confirming version 3.2.5 with protocol version 31 on stapp01*

---

### Phase 10: rsync Installation and Successful Build

With `rsync` installed on the agent, **Build Now** was triggered from the `xfusion-webapp-job` status page to run build #4.

**Build #4 Console Output**

```
Started by user admin
[Pipeline] Start of Pipeline
[Pipeline] node
Running on App Server 1 in /home/sarah/jenkins_agent/workspace/xfusion-webapp-job
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Deploy)
[Pipeline] git
The recommended git tool is: NONE
using credential gitea-sarah
Fetching changes from the remote Git repository
Checking out Revision d81ae80e6413ca40b5a928ae4308b8bdaeb0454a (refs/remotes/origin/master)
Commit message: "Added index.html file"
  > git rev-parse --resolve-git-dir /home/sarah/jenkins_agent/workspace/xfusion-webapp-job/.git
  > git config remote.origin.url http://gitea:3000/sarah/web_app.git
Fetching upstream changes from http://gitea:3000/sarah/web_app.git
  > git --version # timeout=10
  > git --version # 'git version 2.52.0'
using GIT_ASKPASS to set credentials
  > git fetch --tags --force --progress -- http://gitea:3000/sarah/web_app.git
    +refs/heads/*:refs/remotes/origin/* # timeout=10
  > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
  > git config core.sparsecheckout # timeout=10
  > git checkout -f d81ae80e6413ca40b5a928ae4308b8bdaeb0454a # timeout=10
  > git branch -a -v --no-abbrev # timeout=10
  > git branch -D master # timeout=10
  > git checkout -b master d81ae80e6413ca40b5a928ae4308b8bdaeb0454a # timeout=10
  > git rev-list --no-walk d81ae80e6413ca40b5a928ae4308b8bdaeb0454a # timeout=10
[Pipeline] sh
+ rsync -av --delete --exclude=.git
  /home/sarah/jenkins_agent/workspace/xfusion-webapp-job/ /var/www/html/
sending incremental file list
./
index.html

sent 165 bytes  received 38 bytes  406.00 bytes/sec
total size is 35  speedup is 0.17
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

`index.html` was transferred to `/var/www/html/` and the pipeline completed end to end with no errors.

**Build History**

The `xfusion-webapp-job` status page showed build #4 with a green success indicator. The three preceding failed builds (#1, #2, #3) were recorded beneath it in the Builds panel.

*Screenshot: xfusion-webapp-job status page showing build #4 with green success checkmark and builds #1, #2, #3 with failure markers in the Builds panel*

*Screenshot: Build #4 full console output showing git authentication using gitea-sarah credential, all git checkout commands, rsync transferring index.html to /var/www/html, 165 bytes sent, and Finished: SUCCESS at the bottom*

---

## Pipeline Script

The final working pipeline script used in `xfusion-webapp-job`:

```groovy
pipeline {
    agent {
        label 'stapp01'
    }

    stages {
        stage('Deploy') {
            steps {
                git branch: 'master',
                    credentialsId: 'gitea-sarah',
                    url: 'http://gitea:3000/sarah/web_app.git'

                sh '''
                    rsync -av --delete --exclude='.git' \
                    ${WORKSPACE}/ /var/www/html/
                '''
            }
        }
    }
}
```

**Script Breakdown**

* `agent { label 'stapp01' }` pins all pipeline execution to the registered App Server 1 node, identified by the `stapp01` label assigned during node registration
* `stage('Deploy')` is the single stage name, defined exactly as required and case-sensitive
* The `git` step references the `gitea-sarah` credential ID and the internal Gitea hostname `gitea:3000`, which is reachable from App Server 1 within the cluster network
* `rsync -av --delete --exclude='.git'` synchronizes the checked-out workspace to `/var/www/html/`, removing any stale files not present in the current commit and excluding the `.git` directory so version control metadata is never served by Apache

---

## Errors and Resolutions

| Build | Error | Root Cause | Resolution |
|---|---|---|---|
| #1 | `fatal: unable to access ... Recv failure: Connection reset by peer` | Pipeline used the external browser-facing Gitea URL, which is unreachable from the agent within the cluster network | Updated the `url` in the pipeline script to the internal hostname `http://gitea:3000/sarah/web_app.git` |
| #2 | Same git connection error | Build #2 was triggered before the URL correction was applied and confirmed | Pipeline configuration corrected; URL changed to internal hostname |
| #3 | `rsync: command not found` (exit code 127) | `rsync` is not installed by default on CentOS Stream 9; the binary was absent on the agent | Installed `rsync` via `sudo yum install -y rsync` on stapp01, confirmed at version 3.2.5 |
| #4 | None | Both the internal URL and `rsync` were in place | Build succeeded; `index.html` deployed to `/var/www/html/` |

---

## Key Decisions

**SSH credential created before node registration**

The SSH private key credential for `sarah` was added to Jenkins before the agent node was registered. This sequencing was deliberate: the credential must already exist to be selectable in the node configuration form. Creating it first avoids having to return and re-edit the node after registration.

**Gitea username/password credential added after node registration**

The `gitea-sarah` username/password credential was added as a second step, after the agent node was already confirmed online. This credential is only needed at pipeline runtime, not at node registration time, so it was added at the point where pipeline creation was the next step.

**Agent workspace separate from document root**

The Jenkins agent root directory was set to `/home/sarah/jenkins_agent` rather than `/var/www/html`. The Jenkins workspace, build metadata, durable task files, and temporary shell scripts created during execution are all contained within the agent root. Promotion of content to the document root happens only through the explicit `rsync` step with controlled inclusions.

**rsync with `--delete` and `--exclude='.git'`**

Using `rsync --delete` ensures that files removed from the repository are also removed from the document root on the next build, keeping deployed content exactly in sync with source control. Excluding `.git` prevents version control metadata from being exposed through the Apache-served directory.

**Internal Gitea hostname over external URL**

The external lab URL used to access Gitea in the browser is routed through an external proxy and is not reachable from within the cluster. The internal hostname `gitea:3000` resolves correctly from App Server 1 and is the appropriate address for agent-side Git operations.

**Non verifying Host Key Verification Strategy**

In this controlled internal environment with a known, fixed agent IP, Non verifying was used to avoid the bootstrapping complexity of pre-distributing or manually accepting the agent's SSH host key. In a production environment with infrastructure of unknown provenance, a known hosts file or manually trusted key verification strategy would be the correct choice.

---

## Best Practices Applied

* Java 17 installed and explicitly activated via `alternatives --config java` before agent registration, establishing a specific known runtime version rather than relying on the system default
* SSH key authentication used for controller-to-agent connectivity, as the SSH Build Agents plugin requires key-based authentication rather than password-based SSH
* `.ssh` directory and `authorized_keys` file permissions set to `700` and `600` respectively, satisfying OpenSSH security requirements before any connection attempt
* Agent root directory created under `sarah`'s home path, scoping all build artifacts to the intended user account and keeping them separate from system paths
* Apache document root ownership assigned to `sarah` prior to any pipeline run, so the `rsync` write operation succeeds without requiring elevated privileges at runtime
* Credentials stored in the Jenkins credential store and referenced by meaningful, descriptive IDs (`sarah-stapp01`, `gitea-sarah`) rather than hardcoded values in the pipeline script
* Both plugin update cycles (bouncycastle API update and full plugin batch install) used the safe restart checkbox option, avoiding interruption of any in-progress builds
* All four required plugins installed in a single batch operation, reducing the total number of restart cycles from four to one for this phase

---

## Lessons Learned

**Network reachability must be validated from the agent, not from the browser**

The Gitea URL that resolves correctly in a browser does not automatically resolve from a Jenkins agent inside the cluster. The external hostname is routed through an environment-specific proxy and is inaccessible from App Server 1. The correct practice is to verify that the intended Git URL is reachable from the agent directly (for example with `curl` or `git ls-remote` run on the agent host) before triggering the first pipeline build.

**Agent tool dependencies must be audited before writing the pipeline**

`rsync` is not present by default on CentOS Stream 9. The pipeline assumed the binary would be available on the agent and failed with exit code 127 on the first successful git checkout. Checking for required tool binaries on the agent host (for example with `which rsync` or `rpm -q rsync`) before finalizing the pipeline script would have prevented this failure and the associated rebuild cycle.

**Credential creation order is dictated by which component consumes each credential first**

The SSH credential must exist before the agent node is registered because the node configuration form requires it as a required field. The Gitea credential must exist before the first pipeline build but has no dependency on the node configuration step. Planning credential creation to match each credential's point of first consumption eliminates the need to revisit completed configuration steps.

**The agent workspace and the document root are distinct concerns and must be treated as such**

Jenkins writes internal files during builds: durable task scripts, workspace lock files, and temporary shell copies. If the agent root were set directly to `/var/www/html`, these files would be served by Apache during and after builds. Isolating the agent workspace and using `rsync` with explicit exclusions to copy only the intended content to the document root is the correct architectural separation.

**Plugin installations should be batched to minimize restart cycles**

Installing SSH Build Agents, Credentials, Pipeline, and Git together in one operation required only a single restart. Installing them individually would have required four restarts. In environments where Jenkins restarts interrupt running builds, minimizing restart cycles reduces operational risk.











<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/52ff2dab-49ed-4860-bea4-3267e3284089" />
<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/bcfee91d-65eb-43c9-9e31-83cf82b5c187" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/022c9b35-fd16-4080-a780-fdd0302ddcc7" />
<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/8009108e-7a66-4deb-a382-2d99e8a6e178" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/2dbb6cd9-4608-46a8-bc5a-7b8854d91c70" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/df47363f-1e7a-4d02-83ae-3ff62f06ca25" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/3a174886-c5e2-4da7-af1a-33b505a06242" />
<img width="1038" height="858" alt="image" src="https://github.com/user-attachments/assets/e66ee1f0-b2c4-467a-879f-ecdf9d8392c0" />
<img width="1033" height="790" alt="image" src="https://github.com/user-attachments/assets/4f351478-021a-4465-a8c9-3a5f5cb99ace" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/88455d35-6f61-4551-a50f-dae6ad2b1a17" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/964afdda-5eae-4b61-99b3-c512e31352fd" />


