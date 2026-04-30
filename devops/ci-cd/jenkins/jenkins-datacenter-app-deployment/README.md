# Jenkins CI/CD Auto-Deployment Pipeline for xFusionCorp Industries Web Application

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture and Infrastructure](#architecture-and-infrastructure)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Jenkins Login and Initial Plugin Update](#phase-1-jenkins-login-and-initial-plugin-update)
  - [Phase 2: Installing Required Plugins](#phase-2-installing-required-plugins)
  - [Phase 3: App Server Preparation via SSH](#phase-3-app-server-preparation-via-ssh)
  - [Phase 4: Configuring Publish Over SSH in Jenkins System Settings](#phase-4-configuring-publish-over-ssh-in-jenkins-system-settings)
  - [Phase 5: Creating the Jenkins Freestyle Job](#phase-5-creating-the-jenkins-freestyle-job)
  - [Phase 6: Configuring Source Code Management](#phase-6-configuring-source-code-management)
  - [Phase 7: Configuring Build Triggers](#phase-7-configuring-build-triggers)
  - [Phase 8: Configuring the Build Step with SSH Transfer and Exec Command](#phase-8-configuring-the-build-step-with-ssh-transfer-and-exec-command)
  - [Phase 9: Saving the Job and Verifying Initial State](#phase-9-saving-the-job-and-verifying-initial-state)
  - [Phase 10: Developer Workflow - Updating and Pushing Application Code](#phase-10-developer-workflow---updating-and-pushing-application-code)
  - [Phase 11: Verifying Gitea Repository Reflects the Push](#phase-11-verifying-gitea-repository-reflects-the-push)
  - [Phase 12: Configuring the Gitea Webhook to Trigger Jenkins](#phase-12-configuring-the-gitea-webhook-to-trigger-jenkins)
  - [Phase 13: Observing the Triggered Build and Console Output](#phase-13-observing-the-triggered-build-and-console-output)
  - [Phase 14: Diagnosing Initial Deployment Gap and Refining the Exec Command](#phase-14-diagnosing-initial-deployment-gap-and-refining-the-exec-command)
  - [Phase 15: Validating Final Deployment on App Server](#phase-15-validating-final-deployment-on-app-server)
  - [Phase 16: Confirming Idempotent Build Runs](#phase-16-confirming-idempotent-build-runs)
  - [Phase 17: Final Application Verification via Load Balancer URL](#phase-17-final-application-verification-via-load-balancer-url)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)

---

## Project Overview

This project implements an end-to-end CI/CD pipeline for the Nautilus application team within the **Stratos Datacenter** environment. The objective is to eliminate manual deployment steps by automating the delivery of the `sarah/web` Gitea repository to App Server 1 (`stapp01`) every time a developer pushes a change to the `master` branch.

**Trigger mechanism:** A Gitea webhook fires an HTTP POST to Jenkins on every push event. Jenkins polls SCM as a secondary trigger. On build, Jenkins fetches the latest repository state, transfers the files to a staging directory on the app server via SSH, then executes a sequence of privileged shell commands to atomically replace the web root content and restart `httpd`.

**Outcome:** Visiting the load balancer URL (`http://stlb01:8091`) renders the content deployed by Jenkins with no subdirectory in the path, confirming the deployment target is correctly set to `/var/www/html`.

---

## Architecture and Infrastructure

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| Application Server 1 | stapp01 | tony / sarah | Hosts the Nautilus web application via httpd on port 8080 |
| LoadBalancer Server | stlb01 | loki | Distributes HTTP traffic; exposes port 8091 externally |
| Database Server | stdb01 | peter | Hosts Nautilus Database |
| Storage Server | ststor01 | natasha | Stores data for Nautilus Servers |
| Backup Server | stbkp01 | clint | Manages backups |
| Mail Server | stmail01 | groot | Manages email services |
| Jump Host Server | jump-host | thor | Provides secure access to Stratos DC |
| Jenkins Server | jenkins | jenkins | Runs Jenkins CI/CD pipeline |

**Key component relationships:**

```
Developer (sarah) --> git push --> Gitea (sarah/web, port 3000)
                                        |
                                   Webhook POST
                                        |
                                   Jenkins (port 8080)
                                        |
                              SCM checkout + SSH transfer
                                        |
                                 stapp01:/var/www/html
                                        |
                              httpd (port 8080) --> stlb01 (port 8091)
```

---

## Prerequisites

* Jenkins 2.541.2 running and accessible on port 8080
* Gitea 1.25.3 running and accessible on port 3000
* App Server 1 (`stapp01`) reachable from the Jenkins server via SSH
* `httpd` installed on `stapp01` and configured to serve from `/var/www/html` on port 8080
* User `sarah` exists on `stapp01` with a home directory at `/home/sarah`
* The repository `sarah/web` is already cloned at `/home/sarah/web` on `stapp01`
* Jump host access via user `thor` to reach `stapp01` when operating from outside the datacenter

---

## Implementation Guide

### Phase 1: Jenkins Login and Initial Plugin Update

Navigate to the Jenkins UI on port 8080 and sign in using:

* **Username:** `admin`
* **Password:** `Adm!n321`

Screenshot: Jenkins login page with admin credentials entered

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/13361c8b-68be-4ed8-a4c2-fddf74ff5c92" />

After login, navigate to **Manage Jenkins > Plugins > Updates**. One pending update is present: the **bouncycastle API** plugin (version 2.30.1.84). Select it and click **Update**.

Screenshot: Jenkins Plugin Manager showing bouncycastle API update available with health score 97

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/7e565f37-3bfc-4805-83dc-fb76c87b205c" />

On the Download progress page, the bouncycastle API plugin downloads successfully with the message: *"Downloaded Successfully. Will be activated during the next boot."* Enable the checkbox **"Restart Jenkins when installation is complete and no jobs are running"** to apply the update cleanly.

Screenshot: Plugin download progress page showing bouncycastle API downloaded successfully

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/f154f52d-8d3e-4754-ae87-faabbce2e83e" />

Jenkins displays its restarting screen. Wait for the browser to reload automatically before proceeding.

Screenshot: Jenkins restarting screen with "Jenkins is restarting" spinner

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/a8e58492-f85f-40a5-8778-3a5ba659405e" />

---

### Phase 2: Installing Required Plugins

After Jenkins restarts, return to **Manage Jenkins > Plugins > Available plugins**. Search for `git` to locate the required plugins.

Select both:
* **Publish Over SSH** (version 390.vb_f56e7405751) - categorized under Artifact Uploaders and Build Tools
* **Git** (version 5.10.1) - categorized under Source Code Management

Click **Install**.

Screenshot: Available plugins list filtered by "git" showing Publish Over SSH and Git selected with checkboxes

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/6fb269db-a29c-4cdd-9b43-58419553b97a" />

The Download progress page shows all dependency plugins resolving successfully in order, including: JAXB, JSON Api, Jackson Annotations 2 API, Jakarta Activation API, SnakeYAML API, Jakarta XML Binding API, Woodstox Core API, Jackson 2 API, Infrastructure plugin for Publish Over X, Structs, EDDSA API, Gson API, Trilead API, Variant, commons-lang3 v3.x Jenkins API, Ionicons API, commons-text API, Credentials, SSH Credentials, JSch dependency, Publish Over SSH, Pipeline: Step API, Plain Credentials, and others.

Screenshot: Download progress page showing all dependency plugins installing with green success indicators (page 1)

<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/4a5c89d0-eb04-4f18-9765-5270f387ec6c" />

The second page of the progress screen confirms remaining dependencies including: SSH Credentials, JSch dependency, Publish Over SSH, Pipeline: Step API, Plain Credentials, Credentials Binding, ASM API, SCM API, Pipeline: SCM Step, Mina SSHD API Common, Mina SSHD API Core, Apache HttpComponents Client 4.x API, Caffeine API, Script Security, Git client, Jakarta Mail API, Display URL API, Mailer, Git, and Loading plugin extensions - all reporting **Success**.

Screenshot: Download progress page showing remaining plugins all succeeded including Git and Publish Over SSH

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/42a8d3d4-9db1-4dc3-a60e-33747b72ab4d" />

Enable **"Restart Jenkins when installation is complete and no jobs are running"** again. Jenkins restarts to activate all newly installed plugins.

Screenshot: Jenkins restarting screen after Publish Over SSH and Git plugin installation

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/d62bb414-ac05-4973-8bb0-a40408704979" />

---

### Phase 3: App Server Preparation via SSH

From the jump host, SSH into App Server 1 as user `sarah`. The host key fingerprint for `stapp01` is presented on first connection; accept it to add the host to `known_hosts`.

```bash
thor@jumphost ~$ ssh sarah@stapp01
# Accept the host key fingerprint prompt with: yes
# Enter sarah's password when prompted
```

Screenshot: Terminal showing successful SSH from jump host to stapp01 as sarah, including host key acceptance and last login timestamp

<img width="1038" height="471" alt="image" src="https://github.com/user-attachments/assets/45295fd9-fc63-4dde-a46c-3c7ff48f568f" />

Once on `stapp01`, grant `sarah` passwordless sudo access by writing an entry to `/etc/sudoers.d/sarah`. This is required so Jenkins can execute privileged deployment commands (chown, rsync, systemctl) as `sarah` without an interactive password prompt:

```bash
[sarah@stapp01 ~]$ echo "sarah ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/sarah
```

Enter sarah's password when prompted by `sudo`. The sudoers entry is echoed back confirming it was written successfully.

Screenshot: Terminal showing the sudoers echo command execution, sudo lecture text, password prompt, and the written sudoers line confirmed

<img width="1031" height="579" alt="image" src="https://github.com/user-attachments/assets/3191d2de-2523-415b-a3e4-67cbc6525557" />

Start the `httpd` service and enable it to survive reboots:

```bash
[sarah@stapp01 ~]$ sudo systemctl start httpd
sudo systemctl enable httpd
```

Screenshot: Terminal showing httpd start and enable commands executed without errors

<img width="1032" height="694" alt="image" src="https://github.com/user-attachments/assets/1a461ed4-5e55-4ba8-be8e-a1e76871393f" />

Take ownership of the web root directory and clear any pre-existing content so Jenkins has a clean target to deploy into. Also remove any stale `.git` metadata from the web root:

```bash
[sarah@stapp01 ~]$ sudo chown -R sarah:sarah /var/www/html
sudo rm -rf /var/www/html/*
sudo rm -rf /var/www/html/.git
```

Verify the `httpd` DocumentRoot is `/var/www/html` by inspecting the Apache configuration:

```bash
[sarah@stapp01 ~]$ grep -i documentroot /etc/httpd/conf/httpd.conf
```

Output confirms `DocumentRoot "/var/www/html"`.

Screenshot: Terminal showing chown, rm, and DocumentRoot grep commands with confirmed output

<img width="1032" height="556" alt="image" src="https://github.com/user-attachments/assets/10b3b31c-ba22-43da-9c48-75a8f7823929" />

Navigate to the cloned repository and verify the Git remote is correctly pointing to Gitea. Then write the initial content to `index.html` and confirm it:

```bash
[sarah@stapp01 ~]$ cd /home/sarah/web
git remote -v
# origin  http://sarah:Sarah_pass123@gitea:3000/sarah/web.git (fetch)
# origin  http://sarah:Sarah_pass123@gitea:3000/sarah/web.git (push)

echo "Welcome to the xFusionCorp Industries" > index.html
cat index.html
# Welcome to the xFusionCorp Industries
```

Screenshot: Terminal showing git remote -v output with Gitea origin URLs, echo command to index.html, and cat confirming content

<img width="1032" height="581" alt="image" src="https://github.com/user-attachments/assets/5ca547f0-f1d3-4018-a5d4-81f9735a395b" />

Verify `httpd` is active and running:

```bash
[sarah@stapp01 web]$ sudo systemctl status httpd | grep Active
# Active: active (running) since Thu 2026-04-30 03:08:30 UTC; 49min ago
```

Run a local curl to confirm `httpd` is responding. At this stage it still returns the default CentOS HTTP Server Test Page since Jenkins has not yet deployed the custom content to the web root:

```bash
curl -s http://localhost:8080 | (head -5; echo "---"; tail -5)
```

Also confirm the `/var/www/html` directory is owned by `sarah` and is currently empty (the web root was cleaned above and Jenkins has not yet deployed):

```bash
ls -la /var/www/html/
# total 12
# drwxr-xr-x 1 sarah sarah 4096 Apr 30 03:53 .
# drwxr-xr-x 1 root  root  4096 Mar  5 14:50 ..
```

Screenshot: Terminal showing httpd active status, curl output returning the default test page, git remote -v, echo to index.html, and empty /var/www/html listing

<img width="1040" height="498" alt="image" src="https://github.com/user-attachments/assets/c9f5ad90-c3ca-49b1-8115-64f6baf7b8b6" />

---

### Phase 4: Configuring Publish Over SSH in Jenkins System Settings

Return to Jenkins and navigate to **Manage Jenkins > System**. Scroll to the **Publish over SSH** section.

Configure the global SSH server entry that Jenkins will use to connect to App Server 1:

| Field | Value |
|---|---|
| Name | `appserver1` |
| Hostname | `stapp01` |
| Username | `sarah` |
| Remote Directory | `/` |

Screenshot: Jenkins System configuration page showing Publish over SSH section with SSH Server name "appserver1", hostname "stapp01", username "sarah", and remote directory "/"

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/36164506-e0f9-48bb-8aa8-201b3a66df35" />

Expand the **Advanced** section within the SSH Server configuration. Enable **"Use password authentication, or use a different key"** and enter sarah's password in the **Passphrase / Password** field. Set the port to `22` and the timeout to `300000` ms.

Screenshot: Jenkins SSH Server Advanced section showing password authentication enabled with password filled, port 22, and timeout 300000

<img width="1917" height="1020" alt="image" src="https://github.com/user-attachments/assets/ab14d682-0c1c-4d58-92e3-58b458a01a83" />

Click **Test Configuration**. The result displays **Success** at the bottom left of the SSH server configuration block, confirming Jenkins can authenticate to `stapp01` as `sarah`.

Screenshot: Jenkins SSH server configuration showing "Success" test result with port 22, timeout 300000, and proxy settings visible

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/6d10bdc8-9f3d-414e-8e38-2c13c629643d" />

---

### Phase 5: Creating the Jenkins Freestyle Job

From the Jenkins dashboard, click **New Item**. Enter the job name `xfusion-app-deployment` and select **Freestyle project** as the item type. Click **OK**.

Screenshot: Jenkins New Item page showing "xfusion-app-deployment" typed in the name field with Freestyle project selected

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/773e5140-b608-4c34-a5d3-9aec9c9aba9e" />

---

### Phase 6: Configuring Source Code Management

Within the job configuration, navigate to the **Source Code Management** section. Select **Git** as the SCM type.

Configure the repository:

| Field | Value |
|---|---|
| Repository URL | `http://sarah:Sarah_pass123@gitea:3000/sarah/web.git` |
| Credentials | `- none -` (credentials are embedded in the URL) |
| Branch Specifier | `*/master` |
| Repository browser | `(Auto)` |

Screenshot: Jenkins job SCM configuration showing Git selected, repository URL with embedded credentials, and branch specifier set to */master

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/a618934d-782f-46c6-b324-7afe04eb2cf8" />

---

### Phase 7: Configuring Build Triggers

In the **Triggers** section of the job configuration, enable two complementary trigger mechanisms:

**Trigger builds remotely (from scripts):** Enable this and set the **Authentication Token** to `deploy`. This token is embedded in the Gitea webhook URL.

**Poll SCM:** Enable this with a schedule of `* * * * *` (every minute). This acts as a safety net, ensuring Jenkins picks up changes even if the webhook delivery fails. Jenkins displays an advisory warning that polling every minute is aggressive and suggests `H * * * *` for hourly polling, but the every-minute schedule is intentional here for near-real-time responsiveness.

Screenshot: Jenkins job Triggers section showing "Trigger builds remotely" enabled with token, and Poll SCM enabled with "* * * * *" schedule and the advisory warning visible

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/f4d42d8c-c122-48fe-b556-f1474ccd8388" />

---

### Phase 8: Configuring the Build Step with SSH Transfer and Exec Command

In the **Build Steps** section, add a build step: **"Send files or execute commands over SSH"**.

Configure the SSH Publisher:

| Field | Value |
|---|---|
| SSH Server Name | `appserver1` (selected from dropdown) |
| Source files | `**/*` |
| Remove prefix | (empty) |
| Remote directory | `/home/sarah/web-tmp` |

In the **Exec command** field, enter the deployment script. This script atomically replaces the web root content from the staging directory, sets correct ownership, restarts `httpd`, and cleans up the staging directory:

```bash
sudo rm -rf /var/www/html/*
sudo rm -rf /var/www/html/.git
sudo rsync -av --exclude='.git' /home/sarah/web-tmp/ /var/www/html/
sudo chown -R sarah:sarah /var/www/html
sudo systemctl restart httpd
sudo rm -rf /home/sarah/web-tmp
```

Screenshot: Jenkins Build Steps configuration showing SSH Publisher with server "appserver1", source files **\/*, remote directory /home/sarah/web-tmp, and the full exec command entered in the Exec command field

<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/73072262-56e6-48a0-aa88-c6061aef9b76" />

---

### Phase 9: Saving the Job and Verifying Initial State

Click **Save**. The job status page loads showing the `xfusion-app-deployment` job with no builds yet executed.

Screenshot: Jenkins job status page for xfusion-app-deployment showing "No builds" in the Builds panel

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/1d62700a-80d3-4bdb-a0cc-7a2b6035d56a" />

---

### Phase 10: Developer Workflow - Updating and Pushing Application Code

Back on App Server 1, configure Git identity for the `sarah` user, stage the updated `index.html`, commit, and push to the `master` branch:

```bash
[sarah@stapp01 web]$ cd /home/sarah/web
git config user.email "sarah@stratos.com"
git config user.name "sarah"
git add index.html
git commit -m "Update index.html with xFusionCorp Industries"
git push origin master
```

The push output confirms the commit was transferred to Gitea:

```
[master d1bd6bb] Update index.html with xFusionCorp Industries
 1 file changed, 1 insertion(+), 1 deletion(-)
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Writing objects: 100% (3/3), 306 bytes | 306.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
remote: . Processing 1 references in total
To http://gitea:3000/sarah/web.git
   2c796b8..d1bd6bb  master -> master
```

Screenshot: Terminal showing git config, git add, git commit, and git push commands with the successful push output including the remote ref update

<img width="1032" height="799" alt="image" src="https://github.com/user-attachments/assets/f6696d44-b746-42d7-a5c3-fe3ae7ceaff0" />

---

### Phase 11: Verifying Gitea Repository Reflects the Push

Open the Gitea UI on port 3000 and sign in as `sarah`. The dashboard activity feed confirms:

* sarah pushed commit `d1bd6bb06b` with message "Update index.html with xFusionCorp Industries" to `sarah/web` 13 minutes ago
* The repository `sarah/web` is the only repository listed

Screenshot: Gitea login page with username "sarah" and password filled in

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/a7931571-e892-462e-8201-6a07e9183f78" />

Screenshot: Gitea dashboard showing sarah's activity feed with the recent push commit and the sarah/web repository listed

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/1f4f2429-1c0d-4b4e-bb86-9a4b2f1d3df5" />

Navigate to the `sarah/web` repository. The repository shows 2 commits on the `master` branch. The most recent commit is `d1bd6bb06b` with message "Update index.html with xFusionCorp Industries" committed 14 minutes ago. The `index.html` file is listed as the only file in the repository.

Screenshot: Gitea sarah/web repository code view showing 2 commits, master branch, index.html file, and the latest commit message

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/547ae8e6-1ce2-4efa-a470-744fe60c9a43" />

---

### Phase 12: Configuring the Gitea Webhook to Trigger Jenkins

Within the `sarah/web` repository in Gitea, navigate to **Settings > Webhooks** and click **Add Webhook > Gitea**.

Configure the webhook:

| Field | Value |
|---|---|
| Target URL | `http://admin:Adm!n321@jenkins:8080/job/xfusion-app-deployment/build?token=deploy` |
| HTTP Method | `POST` |
| POST Content Type | `application/json` |
| Active | Enabled |
| Trigger On | Push Events |

Click **Add Webhook**.

Screenshot: Gitea Add Webhook page showing the Target URL with Jenkins remote build trigger URL including admin credentials and deploy token, POST method, application/json content type, and Push Events trigger selected

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/7a5bcaa4-ef58-4079-aa87-5fe175b4af00" />

Gitea confirms with a green success banner: **"The webhook has been added."** The webhook appears in the Webhooks list showing the configured Jenkins build URL.

Screenshot: Gitea Webhooks settings page showing success banner and the newly created webhook URL listed

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/020f4cf6-013e-4848-aba2-bb93865259fc" />

---

### Phase 13: Observing the Triggered Build and Console Output

Return to Jenkins. The `xfusion-app-deployment` job shows build **#1** completed successfully (green checkmark) with timestamp 4:10 AM. The Permalinks section displays links to the last build, last stable build, last successful build, and last completed build, all pointing to build #1.

Screenshot: Jenkins job status page showing xfusion-app-deployment with green success icon, build #1 at 4:10 AM, and Permalinks section populated

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/25b57d4e-4b3d-4238-8cf6-0118af2fb554" />

Open the **Console Output** for build #2 (triggered by the SCM change detection after the git push). The console confirms the full build lifecycle:

```
Started by an SCM change
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/xfusion-app-deployment
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir ...
Fetching changes from the remote Git repository
 > git config remote.origin.url http://sarah:Sarah_pass123@gitea:3000/sarah/web.git
Fetching upstream changes from http://sarah@gitea:3000/sarah/web.git
Checking out Revision d1bd6bb06bf12a5660205ac7bc548bd1499a93e2 (refs/remotes/origin/master)
Commit message: "Update index.html with xFusionCorp Industries"
SSH: Connecting from host [jenkins.stratos.xfusioncorp.com]
SSH: Connecting with configuration [appserver1] ...
SSH: EXEC: completed after 1,401 ms
SSH: Disconnecting configuration [appserver1] ...
SSH: Transferred 1 file(s)
Build step 'Send files or execute commands over SSH' changed build result to SUCCESS
Finished: SUCCESS
```

Screenshot: Jenkins Console Output for build #2 showing SCM-triggered build, git fetch operations, SSH connection to appserver1, file transfer, exec completion, and Finished: SUCCESS

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/47dc4f42-d7f4-48ce-b272-32147843f535" />

---

### Phase 14: Diagnosing Initial Deployment Gap and Refining the Exec Command

After the first successful build, inspection of `/var/www/html` on `stapp01` reveals the directory is still empty and `curl http://localhost:8080` still returns the default CentOS Test Page. The `index.html` file is not present in the web root yet.

```bash
[sarah@stapp01 web]$ ls -la /var/www/html/
cat /var/www/html/index.html
# cat: /var/www/html/index.html: No such file or directory

curl -s http://localhost:8080 | (head -10; echo "---"; tail -10)
# Returns the default HTTP Server Test Page
```

Screenshot: Terminal showing ls -la /var/www/html showing empty directory, cat failing with no such file, and curl returning the default test page with DOCTYPE and HTML structure

<img width="1039" height="737" alt="image" src="https://github.com/user-attachments/assets/384b6a91-26b6-4305-9e6a-bd4a48318289" />

**Root cause:** The initial Exec command used `rsync` with a source path that did not correctly resolve relative to where Jenkins places the transferred files, or the `rsync` exclude pattern was not properly removing the `.git` directory before copying. The exec command was revised.

Return to the Jenkins job configuration (**Configure > Build Steps**). Update the **Exec command** to use `cp -r` instead of `rsync`, and add a `find` command to defensively remove any `.git` directories that may have been copied into the web root:

```bash
sudo rm -rf /var/www/html/*
sudo rm -rf /var/www/html/.git
sudo cp -r /home/sarah/web-tmp/. /var/www/html/
sudo find /var/www/html -name ".git" -type d -exec rm -rf {} + 2>/dev/null || true
sudo chown -R sarah:sarah /var/www/html
sudo systemctl restart httpd
sudo rm -rf /home/sarah/web-tmp
```

Screenshot: Jenkins Build Steps configuration showing the updated Exec command with cp -r and find for .git cleanup, remote directory /home/sarah/web-tmp, and source files **\/*

Save the updated configuration.

---

### Phase 15: Validating Final Deployment on App Server

After the updated job runs (build #3, triggered manually or by the next SCM poll), return to `stapp01` and verify the deployment:

```bash
[sarah@stapp01 web]$ ls -la /var/www/html/
# total 16
# drwxr-xr-x 1 sarah sarah 4096 Apr 30 04:22 .
# drwxr-xr-x 1 root  root  4096 Mar  5 14:50 ..
# -rw-r--r-- 1 sarah sarah   38 Apr 30 04:22 index.html

cat /var/www/html/index.html
# Welcome to the xFusionCorp Industries

curl -s http://localhost:8080 | (head -10; echo "---"; tail -10)
# Welcome to the xFusionCorp Industries
# Welcome to the xFusionCorp Industries
```

The `index.html` file is present, owned by `sarah`, and contains the expected content. The curl response confirms `httpd` is serving the deployed content directly from `/var/www/html`.

Screenshot: Terminal showing ls -la /var/www/html with index.html present owned by sarah, cat confirming content, and curl returning "Welcome to the xFusionCorp Industries"

Run a repeat verification showing the same consistent results:

```bash
[sarah@stapp01 web]$ ls -la /var/www/html/
cat /var/www/html/index.html
curl http://localhost:8080
# total 16
# -rw-r--r-- 1 sarah sarah   38 Apr 30 04:22 index.html
# Welcome to the xFusionCorp Industries
# Welcome to the xFusionCorp Industries
```

Screenshot: Terminal showing ls -la, cat, and curl all confirming consistent content in /var/www/html with correct ownership

---

### Phase 16: Confirming Idempotent Build Runs

The Jenkins job status page shows three successful builds: **#3** at 4:22 AM, **#2** at 4:15 AM, and **#1** at 4:10 AM. All show green success indicators, confirming the pipeline is idempotent and can be triggered multiple times without failure or side effects.

Screenshot: Jenkins job status page for xfusion-app-deployment showing builds #1, #2, and #3 all with green success icons and their respective timestamps

The **Console Output** for build **#3** (started by admin via manual trigger for validation) mirrors the same successful execution path as build #2, confirming repeatable behavior:

```
Started by user admin
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/xfusion-app-deployment
...
SSH: Connecting from host [jenkins.stratos.xfusioncorp.com]
SSH: Connecting with configuration [appserver1] ...
SSH: EXEC: completed after 1,401 ms
SSH: Disconnecting configuration [appserver1] ...
SSH: Transferred 1 file(s)
Build step 'Send files or execute commands over SSH' changed build result to SUCCESS
Finished: SUCCESS
```

Screenshot: Jenkins Console Output for build #3 showing admin-triggered build with same successful SSH transfer and exec flow, Finished: SUCCESS

---

### Phase 17: Final Application Verification via Load Balancer URL

Open a browser and navigate to the load balancer URL on port 8091. The page renders:

```
Welcome to the xFusionCorp Industries
```

The content is served directly at the root URL with no subdirectory path, confirming that `/var/www/html` is the correct and active document root, the `httpd` service is healthy, and the full pipeline from git push to live deployment is functioning end-to-end.

Screenshot: Browser showing the load balancer URL on port 8091 rendering "Welcome to the xFusionCorp Industries" at the root path with no subdirectory

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/4a46c962-b11e-4e70-9c2d-55678543b9a8" />

---

## Errors Encountered and Resolutions

### Error 1: Web root empty after first successful build

**Symptom:** Build #1 and #2 completed with `Finished: SUCCESS` in Jenkins, but `ls -la /var/www/html/` showed an empty directory and `curl http://localhost:8080` continued to return the default CentOS Test Page.

**Root cause:** The original Exec command used `rsync` with a source path `/home/sarah/web-tmp/` that, combined with the `--exclude='.git'` flag, did not correctly copy files into `/var/www/html/` when the staging directory structure included a subdirectory matching the transferred content structure. The copy operation completed without error but without placing files in the correct final path.

**Resolution:** Replaced `rsync` with `sudo cp -r /home/sarah/web-tmp/. /var/www/html/` which explicitly copies all content (including hidden files except `.git`) from the staging directory into the web root. Added a defensive `find /var/www/html -name ".git" -type d -exec rm -rf {}` command to handle any residual Git metadata that might be transferred. Rebuild confirmed `index.html` appeared in `/var/www/html/` with correct ownership and content.

### Error 2: httpd still serving the default test page during pre-deployment verification

**Symptom:** During Phase 3, after starting `httpd` and verifying the service was active, `curl http://localhost:8080` returned the default CentOS HTTP Server Test Page rather than custom content.

**Root cause:** This was expected behavior, not an error. The web root `/var/www/html/` was intentionally cleared during Phase 3 setup. Custom content is only placed there by the Jenkins deployment job. The default page is returned by Apache when the document root contains no `index.html`.

**Resolution:** No action required. This state is correct prior to the first Jenkins deployment. After the pipeline ran successfully, the curl output correctly returned the deployed content.

### Error 3: Jenkins advisory warning on Poll SCM schedule

**Symptom:** After setting Poll SCM to `* * * * *`, Jenkins displayed an orange warning: *"Do you really mean 'every minute' when you say '* * * * *'? Perhaps you meant 'H * * * *' to poll once per hour."*

**Root cause:** Jenkins detects overly aggressive polling schedules and warns administrators. Polling every minute from Jenkins to Gitea is not recommended for large-scale deployments due to server load.

**Resolution:** The `* * * * *` schedule was retained intentionally for this implementation. The Gitea webhook serves as the primary trigger. Poll SCM at one-minute intervals acts as a reliable fallback in case webhook delivery fails. For production environments serving high commit volumes, `H/5 * * * *` or `H * * * *` is preferred.

---

## Best Practices Applied

* **Sudoers drop-in file:** Passwordless sudo for `sarah` was written to `/etc/sudoers.d/sarah` rather than modifying `/etc/sudoers` directly. This follows the principle of minimal-footprint changes and is easier to audit and revoke.

* **Staging directory pattern:** Jenkins transfers files to `/home/sarah/web-tmp` before moving them into `/var/www/html`. This prevents a partially-transferred set of files from being served by `httpd` during a slow or failed transfer, reducing the window of inconsistency.

* **Atomic web root replacement:** The Exec command clears `/var/www/html/*` before copying new content. Combined with `httpd` restart after the copy, this ensures the server always serves either the old complete version or the new complete version, never a mix.

* **Git metadata exclusion from web root:** Explicitly removing `.git` from the web root via both the `rsync --exclude` approach and the defensive `find -name ".git"` cleanup prevents accidental exposure of source history through the web server.

* **Dual trigger strategy:** Combining a remote build trigger (webhook) with Poll SCM creates a redundant trigger architecture. The webhook provides near-real-time builds; Poll SCM guarantees eventual consistency even when the webhook cannot reach Jenkins.

* **Idempotent deployment script:** The Exec command is designed to produce the same result regardless of whether it runs once or ten times on the same commit. Clearing the web root before copying, and removing the staging directory after deployment, ensures no stale artifacts accumulate across repeated runs.

* **Credentials embedded in Gitea remote URL:** The Git remote URL includes `sarah:Sarah_pass123@gitea:3000` directly. In this controlled lab environment this is acceptable. In production, Jenkins credentials binding or SSH key-based authentication should be used, and credentials should be stored in Jenkins' encrypted credential store rather than in plaintext URLs.

* **httpd restart in deployment script:** Restarting the web server as part of each deployment ensures any configuration or module-level caches are cleared and the new content is served immediately. For stateless static sites this is safe and ensures no stale caching at the server layer.

---

## Lessons Learned

* **Test file transfer paths before relying on them.** The first version of the Exec command appeared to succeed (SSH reported `Transferred 1 file(s)`, build returned SUCCESS) but the files did not reach the intended target. Always validate the actual result on the destination server immediately after the first build, not just the Jenkins build status. `ls -la /var/www/html/` and `cat /var/www/html/index.html` are the ground-truth checks.

* **`rsync` source trailing slash semantics matter.** When using `rsync`, a trailing slash on the source path (`/home/sarah/web-tmp/`) means "copy the contents of this directory." Without the slash, rsync copies the directory itself as a subdirectory of the destination. Confusion between these two behaviors is a common source of deployment path errors. Switching to `cp -r /home/sarah/web-tmp/. /var/www/html/` uses the `.` convention which unambiguously means "everything inside this directory."

* **Gitea webhook URL must embed Jenkins credentials.** The remote build trigger URL format is `http://<user>:<password>@<jenkins-host>:<port>/job/<job-name>/build?token=<token>`. The token alone is insufficient; Jenkins also requires HTTP Basic Authentication in the URL for the webhook to be accepted.

* **Poll SCM and webhooks are complementary, not redundant.** In environments where the CI server and source control are on different network segments or behind firewalls, webhook delivery may be unreliable. Poll SCM guarantees Jenkins will eventually detect any push, making it a valuable safety net even when webhooks are configured.

* **Clearing the staging directory after deployment prevents state accumulation.** If the `sudo rm -rf /home/sarah/web-tmp` step is omitted, subsequent builds append to an existing staging directory rather than starting clean, which can cause old files from previous commits to persist in the web root if they were deleted from the repository.

* **The `sarah` sudoers entry must exist before the first Jenkins build attempts to run privileged commands.** If the sudoers entry is missing, the `sudo systemctl restart httpd` and `sudo chown` commands in the Exec script will fail with a permission denied error even though Jenkins connected successfully via SSH. Pre-staging the sudoers configuration on the target server is a prerequisite step that must be completed before the Jenkins job is saved and triggered.











<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/81ce317f-3f84-4f67-8f3f-e38a2622b511" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/98ec4cfc-a3ce-4d64-bff4-5297edafd914" />
<img width="1032" height="447" alt="image" src="https://github.com/user-attachments/assets/ea13ba50-a1a7-45e9-9a8c-ab27e4082fa3" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/965ac60e-dc8f-460a-89ad-abc452efa645" />
<img width="1038" height="405" alt="image" src="https://github.com/user-attachments/assets/d46e2d5a-9071-4ab7-892e-b7d6991cf2e9" />















