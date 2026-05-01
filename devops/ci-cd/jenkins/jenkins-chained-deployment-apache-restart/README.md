# Jenkins Chained Builds: Automated Web Deployment and Service Restart via Publish Over SSH

## Table of Contents

- [Overview](#overview)
- [Architecture and Solution Design](#architecture-and-solution-design)
- [Infrastructure Context](#infrastructure-context)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: SSH Into the Jenkins Server and Generate an RSA Key Pair](#phase-1-ssh-into-the-jenkins-server-and-generate-an-rsa-key-pair)
  - [Phase 2: Copy the Public Key to App Server 1 and Verify Remote Deployment](#phase-2-copy-the-public-key-to-app-server-1-and-verify-remote-deployment)
  - [Phase 3: Log In to Jenkins and Update the Bouncy Castle Plugin](#phase-3-log-in-to-jenkins-and-update-the-bouncy-castle-plugin)
  - [Phase 4: Install the Publish Over SSH and Git Plugins](#phase-4-install-the-publish-over-ssh-and-git-plugins)
  - [Phase 5: Configure the Publish Over SSH Global Settings](#phase-5-configure-the-publish-over-ssh-global-settings)
  - [Phase 6: Create the Upstream Job datacenter-app-deployment](#phase-6-create-the-upstream-job-datacenter-app-deployment)
  - [Phase 7: Configure Source Code Management for datacenter-app-deployment](#phase-7-configure-source-code-management-for-datacenter-app-deployment)
  - [Phase 8: Configure the Build Step for datacenter-app-deployment](#phase-8-configure-the-build-step-for-datacenter-app-deployment)
  - [Phase 9: Configure the Post-Build Action to Trigger the Downstream Job](#phase-9-configure-the-post-build-action-to-trigger-the-downstream-job)
  - [Phase 10: Create the Downstream Job manage-services](#phase-10-create-the-downstream-job-manage-services)
  - [Phase 11: Configure the Trigger for manage-services](#phase-11-configure-the-trigger-for-manage-services)
  - [Phase 12: Configure the Build Step for manage-services](#phase-12-configure-the-build-step-for-manage-services)
  - [Phase 13: Verify Both Jobs Exist on the Jenkins Dashboard](#phase-13-verify-both-jobs-exist-on-the-jenkins-dashboard)
  - [Phase 14: Build Validation - Run #1](#phase-14-build-validation---run-1)
  - [Phase 15: Verify the Application is Accessible](#phase-15-verify-the-application-is-accessible)
  - [Phase 16: Build Validation - Run #2 (Idempotency Verification)](#phase-16-build-validation---run-2-idempotency-verification)
- [Build Results Summary](#build-results-summary)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Errors and Resolutions](#errors-and-resolutions)

---

## Overview

This implementation configures a Jenkins chained build pipeline for the Stratos Datacenter DevOps team. The solution automates web application deployment to App Server 1 and conditionally restarts the Apache (`httpd`) service only when the upstream deployment succeeds. Two Jenkins Freestyle jobs are created and linked in a parent-child (upstream-downstream) relationship using the **Publish Over SSH** plugin to execute remote commands over a key-authenticated SSH connection.

---

## Architecture and Solution Design

The pipeline consists of two Jenkins Freestyle jobs connected in a chained build relationship:

```
[Git Repo: sarah/web (Gitea)]
        |
        | (Git pull on App Server 1 via SSH exec)
        v
[Job 1: datacenter-app-deployment]  <-- Upstream Job
        |
        | (Post-build trigger: only if stable)
        v
[Job 2: manage-services]  <-- Downstream Job
        |
        | (SSH exec: sudo systemctl restart httpd)
        v
[App Server 1: stapp01.stratos.xfusioncorp.com]
```

The upstream job pulls the latest code from the master branch of the `web` Gitea repository into the `/var/www/html` document root on App Server 1 via SSH remote execution. If and only if the upstream build is stable, the downstream job triggers automatically and restarts the `httpd` service on the same app server.

---

## Infrastructure Context

| Component | Value |
|-----------|-------|
| Jenkins Server | `jenkins.stratos.xfusioncorp.com` |
| Jenkins Port | `8080` |
| Jenkins User | `admin` |
| App Server 1 | `stapp01.stratos.xfusioncorp.com` |
| App Server OS User | `tony` |
| Doc Root | `/var/www/html` |
| Gitea URL | `http://gitea.stratos.xfusioncorp.com:3000` |
| Gitea User | `sarah` |
| Git Repository | `sarah/web` |
| Git Branch | `master` |
| LB Port | `8091` |
| Jenkins Version | `2.541.2` |

---

## Prerequisites

* SSH access from a jump host to the Jenkins server as the `jenkins` OS user
* The `jenkins` OS user must have a home directory at `/var/lib/jenkins`
* The `tony` user on App Server 1 must have `sudo` privileges for `git` and `systemctl` commands without a password prompt (configured via `/etc/sudoers`)
* Apache (`httpd`) must already be installed and running on App Server 1
* The `/var/www/html` directory on App Server 1 must already be initialized as a local Git repository

---

## Implementation Guide

### Phase 1: SSH Into the Jenkins Server and Generate an RSA Key Pair

SSH from the jump host (`thor@jumphost`) into the Jenkins server as the `jenkins` OS user. When prompted about host authenticity, type `yes` to accept the fingerprint and add it to `known_hosts`. Authenticate with the `jenkins` user password when prompted.

```bash
thor@jumphost:~$ ssh jenkins@jenkins
```

After landing on the Jenkins server shell, generate a 2048-bit RSA key pair for the `jenkins` user with no passphrase. This key pair will be used by Jenkins to authenticate against App Server 1 over SSH.

```bash
jenkins@jenkins:~$ ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
```

The output confirms that both the private key (`/var/lib/jenkins/.ssh/id_rsa`) and the public key (`/var/lib/jenkins/.ssh/id_rsa.pub`) have been saved.

*Screenshots: SSH login to Jenkins server from jump host and RSA key pair generation output*

<img width="1039" height="807" alt="image" src="https://github.com/user-attachments/assets/ec2e3a5c-7e27-4a78-b124-5c0fd7e9f663" />
<img width="1037" height="553" alt="image" src="https://github.com/user-attachments/assets/3f75baa7-5bf6-426c-80a5-38cf167b6d83" />

---

### Phase 2: Copy the Public Key to App Server 1 and Verify Remote Deployment

Copy the generated public key to the `tony` user's `authorized_keys` on App Server 1 using `ssh-copy-id`. Disable strict host key checking to avoid interactive prompts during automation. Enter `tony`'s password when prompted.

```bash
jenkins@jenkins:~$ ssh-copy-id -o StrictHostKeyChecking=no tony@stapp01.stratos.xfusioncorp.com
```

The output confirms `Number of key(s) added: 1`. Immediately verify the key-based SSH authentication by running a compound remote command that pulls the latest code from Gitea into `/var/www/html` and restarts Apache, all within a single SSH session.

```bash
jenkins@jenkins:~$ ssh -o StrictHostKeyChecking=no tony@stapp01.stratos.xfusioncorp.com \
  "sudo git config --global --add safe.directory /var/www/html && \
   sudo git -C /var/www/html remote set-url origin http://sarah:Sarah_pass123@gitea.stratos.xfusioncorp.com:3000/sarah/web.git \
   && \
   sudo git -C /var/www/html pull origin master && \
   sudo systemctl restart httpd && echo ALL_GOOD"
```

The terminal output shows a fast-forward merge (`1 file changed, 1 insertion(+), 1 deletion(-)`) followed by `ALL_GOOD`, confirming that passwordless SSH, sudo permissions, git operations, and `httpd` restart all function correctly end to end.

Confirm the private key content is readable for the next phase by printing it:

```bash
jenkins@jenkins:~$ cat ~/.ssh/id_rsa
```

Copy the full private key content including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` header and footer lines. This will be pasted into Jenkins in a later phase.

*Screenshots: ssh-copy-id output showing 1 key added, remote deployment command output showing ALL_GOOD, and private key content printed to terminal*

<img width="1034" height="852" alt="image" src="https://github.com/user-attachments/assets/fd1d7f8d-fa37-4fdb-b5ef-73803948f003" />
<img width="1033" height="859" alt="image" src="https://github.com/user-attachments/assets/f3fa2af9-a6a8-40ae-864d-e1ad0dd1d28a" />

---

### Phase 3: Log In to Jenkins and Update the Bouncy Castle Plugin

Open the Jenkins web UI at port `8080`. Log in using username `admin` and password `Adm!n321`.

*Screenshot: Jenkins login page with admin credentials entered*

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/b168f004-680f-4563-bd83-9e7632f4700a" />

Navigate to **Manage Jenkins > Plugins > Updates**. The Updates tab shows one available update: the **Bouncy Castle API** plugin (`2.30.1.84` to `2.30.1.84-291`). This plugin provides cryptographic primitives required by SSH-related plugins. Select the checkbox next to it and click **Update**.

*Screenshot: Jenkins Plugin Manager Updates tab showing Bouncy Castle API update available*

<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/13aefd68-8fff-4372-8f86-d7f77aeac11e" />

On the Download progress page, the Bouncy Castle API plugin downloads successfully. The page shows the message `Downloaded Successfully. Will be activated during the next boot`. Enable the **Restart Jenkins when installation is complete and no jobs are running** checkbox to trigger a safe restart.

*Screenshot: Bouncy Castle API download progress showing success and restart pending*

<img width="1918" height="1020" alt="image" src="https://github.com/user-attachments/assets/9ee763fb-95bf-417a-a16a-88bc1cdd0811" />

Jenkins displays the restarting screen. The browser reloads automatically when Jenkins is ready.

*Screenshot: Jenkins restarting screen*

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/07c7fb5e-5eb1-4274-a8b8-6c401738857d" />

---

### Phase 4: Install the Publish Over SSH and Git Plugins

After Jenkins restarts, navigate to **Manage Jenkins > Plugins > Available plugins**. Search for `publish over` in the search field. The results show multiple plugins. Select the checkboxes for:

* **Git** (Source Code Management integration)
* **SSH Agent**
* **Publish Over SSH** (for sending build artifacts and executing commands over SSH)

Click **Install**.

*Screenshot: Available plugins search results showing Git, SSH Agent, and Publish Over SSH selected for installation*

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/3fb4b66a-0150-40ee-8108-3e7e193fa79e" />

The Download progress page shows all dependencies installing successfully. Key packages installed include:

* `commons-lang3 v3.x Jenkins API`
* `Credentials`, `Plain Credentials`, `SSH Credentials`, `Credentials Binding`
* `Git client`, `Git`, `SSH Agent`
* `Mina SSHD API :: Common`, `Mina SSHD API :: Core`
* `JSch dependency`
* `Publish Over SSH`
* All transitive dependencies show `Success`

*Screenshot: Download progress page showing all plugin dependencies installing with Success status (page 1)*

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/234a8e6b-3eb9-4118-baa7-84a3e2ac0e6b" />

*Screenshot: Download progress page showing remaining dependencies including Publish Over SSH all showing Success*

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/571f89a1-a1ef-4d6a-8b4a-f32a45fde2c5" />

After the installation completes, enable **Restart Jenkins when installation is complete and no jobs are running** and allow Jenkins to perform another safe restart.

*Screenshot: Jenkins restarting screen after Publish Over SSH plugin installation*

<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/c8207a28-b7de-4a4d-965c-562556e58ea8" />

---

### Phase 5: Configure the Publish Over SSH Global Settings

Navigate to **Manage Jenkins > System**. Scroll down to the **Publish over SSH** section. This is where the global SSH key and target server definitions are configured.

In the **Key** text area, paste the full RSA private key content that was copied from the Jenkins server in Phase 2. The key must include the complete `-----BEGIN OPENSSH PRIVATE KEY-----` block.

*Screenshot: Manage Jenkins System page showing Publish over SSH section with private key pasted into the Key field*

Scroll down to the **SSH Servers** section. Click **Add** to define the target server. Fill in the following fields:

| Field | Value |
|-------|-------|
| Name | `app-server-1` |
| Hostname | `stapp01.stratos.xfusioncorp.com` |
| Username | `tony` |
| Remote Directory | `/` |

Click **Test Configuration**. The result shows **Success**, confirming that Jenkins can authenticate against App Server 1 using the configured private key and reach the server over SSH.

Click **Save**.

*Screenshot: SSH Server configuration for app-server-1 showing Name, Hostname, Username, Remote Directory fields and Test Configuration showing Success*

---

### Phase 6: Create the Upstream Job datacenter-app-deployment

From the Jenkins dashboard, click **New Item**. Enter `datacenter-app-deployment` as the item name. Select **Freestyle project** as the item type. Click **OK**.

*Screenshot: New Item page with datacenter-app-deployment name entered and Freestyle project selected*

---

### Phase 7: Configure Source Code Management for datacenter-app-deployment

In the job configuration page, navigate to the **Source Code Management** section. Select **Git**. In the **Repository URL** field, enter:

```
http://sarah:Sarah_pass123@gitea.stratos.xfusioncorp.com:3000/sarah/web.git
```

Leave **Credentials** as `- none -` because the credentials are embedded directly in the repository URL. Set **Branch Specifier** to `*/master`.

*Screenshot: Source Code Management section showing Git selected, repository URL with embedded credentials, and branch set to master*

---

### Phase 8: Configure the Build Step for datacenter-app-deployment

Navigate to the **Build Steps** section. Click **Add build step** and select **Send files or execute commands over SSH**.

In the **SSH Publishers** configuration:

* **Name**: Select `app-server-1` from the dropdown (this is the server registered in the global Publish Over SSH settings)
* Leave **Source files**, **Remove prefix**, and **Remote directory** fields empty because no files are being transferred
* In the **Exec command** field, enter the following deployment script:

```bash
sudo git config --global --add safe.directory /var/www/html
sudo git -C /var/www/html remote set-url origin http://sarah:Sarah_pass123@gitea.stratos.xfusioncorp.com:3000/sarah/web.git
sudo git -C /var/www/html pull origin master
sudo systemctl restart httpd && echo ALL_GOOD
```

*Screenshot: Build Steps section showing Send files or execute commands over SSH configured with app-server-1 and exec command deployment script*

---

### Phase 9: Configure the Post-Build Action to Trigger the Downstream Job

Navigate to the **Post-build Actions** section. Click **Add post-build action** and select **Build other projects**.

In the **Projects to build** field, enter `manage-services`. Select **Trigger only if build is stable** to ensure the downstream job only runs when the upstream deployment succeeds without errors.

Note: At this point a warning appears (`No such project 'manage-services'`) because the `manage-services` job has not been created yet. This is expected and the warning resolves once the downstream job is created in the next phase.

Click **Save**.

*Screenshot: Post-build Actions section showing Build other projects configured with manage-services and Trigger only if build is stable selected, with the project-not-found warning visible*

---

### Phase 10: Create the Downstream Job manage-services

From the Jenkins dashboard, click **New Item**. Enter `manage-services` as the item name. Select **Freestyle project**. Click **OK**.

*Screenshot: New Item page with manage-services name entered and Freestyle project selected*

---

### Phase 11: Configure the Trigger for manage-services

In the `manage-services` job configuration, navigate to the **Triggers** section. Check the **Build after other projects are built** checkbox.

In the **Projects to watch** field, enter `datacenter-app-deployment`. Select **Trigger only if build is stable**.

This configures `manage-services` as a downstream job that will execute automatically whenever `datacenter-app-deployment` completes with a stable (successful) status. The **Source Code Management** section remains set to **None** because this job does not pull any source code.

*Screenshot: manage-services Triggers section showing Build after other projects are built checked, datacenter-app-deployment entered as the upstream project, and Trigger only if build is stable selected*

---

### Phase 12: Configure the Build Step for manage-services

Navigate to the **Build Steps** section of `manage-services`. Click **Add build step** and select **Send files or execute commands over SSH**.

Configure the SSH Publisher:

* **Name**: Select `app-server-1` from the dropdown
* Leave **Source files**, **Remove prefix**, and **Remote directory** fields empty
* In the **Exec command** field, enter:

```bash
sudo systemctl restart httpd
```

Click **Save**.

*Screenshot: manage-services Build Steps section showing Send files or execute commands over SSH with app-server-1 selected and sudo systemctl restart httpd as the exec command*

---

### Phase 13: Verify Both Jobs Exist on the Jenkins Dashboard

Return to the Jenkins dashboard. Confirm that both jobs appear in the job list:

* `datacenter-app-deployment`
* `manage-services`

Both jobs show `N/A` for Last Success, Last Failure, and Last Duration since they have not been executed yet.

*Screenshot: Jenkins dashboard showing both datacenter-app-deployment and manage-services jobs listed with N/A status*

---

### Phase 14: Build Validation - Run #1

Manually trigger the `datacenter-app-deployment` job by clicking **Build Now**. Open the Console Output for Build #1.

The console output confirms the following sequence:

1. Jenkins clones the remote Git repository from `http://sarah:Sarah_pass123@gitea.stratos.xfusioncorp.com:3000/sarah/web.git`
2. Git checks out commit `f2d770d74805e3d2d8a0f0b4777e3b13d5d74d82` on the `master` branch with commit message `Updated index.html file`
3. SSH connects to `[jenkins.stratos.xfusioncorp.com]` using configuration `[app-server-1]`
4. The exec command completes in `200 ms`
5. The build step reports `Build step 'Send files or execute commands over SSH' changed build result to SUCCESS`
6. Jenkins automatically triggers `manage-services` (the downstream job)
7. The upstream job finishes with `Finished: SUCCESS`

*Screenshot: datacenter-app-deployment Build #1 console output showing full execution sequence ending in SUCCESS and downstream manage-services triggered*

The `manage-services` Build #1 console output confirms:

1. The job was started by the upstream project `datacenter-app-deployment` build number `1`, originally caused by user `admin`
2. SSH connects to `[jenkins.stratos.xfusioncorp.com]` using configuration `[app-server-1]`
3. The exec command (`sudo systemctl restart httpd`) completes in `1,201 ms`
4. The build step reports `Build step 'Send files or execute commands over SSH' changed build result to SUCCESS`
5. The downstream job finishes with `Finished: SUCCESS`

*Screenshot: manage-services Build #1 console output confirming triggered by upstream and httpd restart completed successfully*

---

### Phase 15: Verify the Application is Accessible

Open the application URL on port `8091` in the browser. The page loads and displays `Welcome to KodeKloud!`, confirming that:

* The web content was successfully pulled from the Gitea repository into `/var/www/html`
* Apache is serving the content correctly after the service restart

*Screenshot: Application loaded at port 8091 displaying "Welcome to KodeKloud!"*

---

### Phase 16: Build Validation - Run #2 (Idempotency Verification)

Manually trigger a second run of `datacenter-app-deployment` to confirm the pipeline is idempotent and passes on repetitive executions.

The `datacenter-app-deployment` Build #2 console output shows:

1. Git detects the workspace already exists and fetches upstream changes from the remote repository
2. Git checks out the same commit `f2d770d74805e3d2d8a0f0b4777e3b13d5d74d82` (no new commits)
3. SSH exec completes successfully in `200 ms`
4. `manage-services` is triggered again
5. Build finishes with `Finished: SUCCESS`

*Screenshot: datacenter-app-deployment Build #2 console output showing idempotent run with SUCCESS result and manage-services triggered*

The `manage-services` Build #2 console output confirms the downstream job was triggered by `datacenter-app-deployment` build number `2` and the `httpd` restart completed successfully again in `1,201 ms`.

*Screenshot: manage-services Build #2 console output showing triggered by upstream build #2 and Finished: SUCCESS*

---

## Build Results Summary

The final state of both jobs on the Jenkins dashboard confirms the complete upstream-downstream relationship:

The `datacenter-app-deployment` job status page shows:

* Green checkmark status for the most recent build
* **Downstream Projects**: `manage-services` (with green checkmark)
* Build history: `#1` at 5:31 AM and `#2` at 5:34 AM, both successful

*Screenshot: datacenter-app-deployment status page showing Downstream Projects section with manage-services listed and two successful builds in history*

The `manage-services` job status page shows:

* Green checkmark status for the most recent build
* **Upstream Projects**: `datacenter-app-deployment` (with green checkmark)
* Build history: `#1` at 5:31 AM and `#2` at 5:34 AM, both successful

*Screenshot: manage-services status page showing Upstream Projects section with datacenter-app-deployment listed and two successful builds in history*

---

## Best Practices Applied

**Key-based SSH Authentication**: A dedicated RSA 2048-bit key pair was generated for the `jenkins` OS user and distributed to App Server 1 using `ssh-copy-id`. This eliminates password-based authentication in automated pipelines, which is a production security requirement.

**StrictHostKeyChecking Disabled Only for Initial Setup**: The `-o StrictHostKeyChecking=no` flag was used during the initial `ssh-copy-id` and verification steps. The Publish Over SSH plugin handles host key verification internally through the global server configuration.

**Sudo for All Remote Filesystem Operations**: All Git and `systemctl` commands executed on App Server 1 use `sudo` to ensure the `tony` user has the necessary privileges to write to `/var/www/html` and manage system services, avoiding permission failures in automated contexts.

**git safe.directory Configuration**: Including `sudo git config --global --add safe.directory /var/www/html` in the exec command prevents Git from refusing to operate in a directory owned by a different user. This is a required configuration for Git version 2.35.2 and later when running as `sudo`.

**Remote URL Reset on Each Run**: Setting the remote origin URL with embedded credentials on each run via `git remote set-url` ensures the pipeline is self-healing and does not depend on the state left by a previous run or a manual operation on the server.

**Downstream Trigger Set to Stable Builds Only**: The `manage-services` job is configured to trigger only when `datacenter-app-deployment` is stable. This prevents unnecessary service restarts when the deployment itself has failed, which could mask deployment errors.

**Plugin Updates Before Installation**: The Bouncy Castle API plugin was updated before installing the Publish Over SSH plugin to ensure all cryptographic dependencies were current, preventing compatibility errors during SSH key operations.

**Idempotent Build Design**: The pipeline was designed to produce the same result on every execution regardless of whether the repository has new commits. `git pull` on a repository that is already up to date exits cleanly without errors, making the pipeline safe to run repeatedly.

---

## Lessons Learned

**The order of plugin operations matters**: Updating the Bouncy Castle API plugin before installing Publish Over SSH is critical. Bouncy Castle provides the cryptographic library that SSH key handling depends on. Installing Publish Over SSH before Bouncy Castle is current can result in runtime failures during key parsing or test configuration.

**Private key format must be exact**: When pasting the private key into the Jenkins global Publish Over SSH configuration, the entire key content including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` lines must be included without trailing spaces or line breaks. Any truncation causes authentication failures during Test Configuration.

**The manage-services job must exist before saving the upstream job**: Jenkins displays a validation warning (`No such project 'manage-services'`) in the Post-build Actions section of `datacenter-app-deployment` if the downstream job does not yet exist. While Jenkins allows saving in this state, the chained trigger will not resolve until the downstream job is created. Creating the downstream job immediately after saves the upstream resolves the warning on the next save.

**Configuring the downstream trigger in both jobs**: The chained relationship is reinforced by configuring the trigger from both directions. The upstream job has a post-build action targeting the downstream job, and the downstream job has a trigger listening to the upstream job. This dual configuration ensures the relationship is correctly reflected on both job status pages and is more resilient to configuration changes.

**Test Configuration must pass before proceeding**: The **Test Configuration** button in the SSH Servers section provides immediate feedback on whether the private key, hostname, and username combination is valid. Running this test before creating any jobs catches authentication issues early and avoids silent build failures later.

**git safe.directory is a runtime requirement**: Without the `git config --global --add safe.directory /var/www/html` command, Git 2.35.2+ will refuse to operate in a directory owned by root (or a different user than the one invoking the `sudo git` command), producing an `unsafe repository` fatal error. This must be included as the first command in the exec script.

---

## Errors and Resolutions

**Error: `No such project 'manage-services'` warning in Post-build Actions**

* **Root Cause**: The `datacenter-app-deployment` job's post-build trigger was configured before the `manage-services` job was created. Jenkins performs a live validation of project names referenced in post-build actions.
* **Resolution**: This warning is expected during the setup sequence when the upstream job is configured first. The warning resolves automatically once `manage-services` is created. No functional issue results from saving the upstream job in this state.

**Error: Bouncy Castle plugin shows `Will be activated during the next boot` warning**

* **Root Cause**: Jenkins cannot hot-reload cryptographic library updates while the server is running due to classloader isolation.
* **Resolution**: The **Restart Jenkins when installation is complete and no jobs are running** checkbox was enabled to trigger a safe restart, which activated the updated plugin before the Publish Over SSH installation began.

**Error: SSH connection fails during Test Configuration after pasting the private key**

* **Root Cause (anticipated)**: This class of failure typically occurs when the private key is pasted with extra whitespace, missing headers, or when the corresponding public key has not been added to the target user's `authorized_keys` file.
* **Resolution**: The issue was avoided by verifying end-to-end SSH connectivity manually from the Jenkins OS shell using the same key before pasting it into the Jenkins UI, and by using `ssh-copy-id` to distribute the public key to App Server 1 prior to any Jenkins configuration.









<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/074bb4f3-2197-4271-b709-90c6a857749e" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/33635b1b-0c01-4802-a73f-7ff84c1bb069" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/c62e9740-ff6c-470d-abf8-1fdb92653267" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/ca0cc8df-a4f1-4cac-a8db-85e4b9464ca4" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/92ba219d-bc46-44de-9c1a-07d26e1bf06e" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/76e75834-1cbe-48fc-b8d3-5be241734ee8" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/88e38cca-4583-4c95-937a-52cc088e21ab" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/9aeb3b03-1559-4491-9a28-a6e42b4d48ac" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/bcfcef45-02fc-4cb4-9465-047f4ca2a319" />
<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/8baf8b11-8ecc-4bee-b897-f241a92034f9" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/c3384d9e-3bc0-4b80-b4d9-7a769ce1c3d3" />
<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/827a4790-91fe-4de1-bc19-7155fc236a15" />
<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/e7efe77d-4bb9-48b6-91fe-53227d4bfb1e" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/f739c1e0-eafa-46de-9a75-8c7d0284ebd1" />
<img width="1919" height="1010" alt="image" src="https://github.com/user-attachments/assets/69cbb168-accc-49b7-bf3f-2c0c3fb0a4ee" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/6a162282-5c8f-4687-8b03-0465b6052851" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/b5288e2c-20b0-4d47-b09d-2528783213b0" />











