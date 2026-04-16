# Jenkins Plugin Installation: Git and GitLab Plugins

> **Domain:** CI/CD Infrastructure | **Platform:** Jenkins on Ubuntu 24.04 LTS | **Environment:** KodeKloud Nautilus

---

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Architecture and Context](#architecture-and-context)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1: SSH into the Jenkins Server](#step-1-ssh-into-the-jenkins-server)
  * [Step 2: Verify Jenkins Process and File System Layout](#step-2-verify-jenkins-process-and-file-system-layout)
  * [Step 3: Access the Jenkins Web UI](#step-3-access-the-jenkins-web-ui)
  * [Step 4: Log In to Jenkins](#step-4-log-in-to-jenkins)
  * [Step 5: Navigate to the Plugin Manager](#step-5-navigate-to-the-plugin-manager)
  * [Step 6: Explore the Updates Tab](#step-6-explore-the-updates-tab)
  * [Step 7: Search and Select Git and GitLab Plugins](#step-7-search-and-select-git-and-gitlab-plugins)
  * [Step 8: Initiate Plugin Installation](#step-8-initiate-plugin-installation)
  * [Step 9: Monitor Download Progress](#step-9-monitor-download-progress)
  * [Step 10: Enable Restart on Completion](#step-10-enable-restart-on-completion)
  * [Step 11: Wait for Jenkins to Restart](#step-11-wait-for-jenkins-to-restart)
  * [Step 12: Log Back In After Restart](#step-12-log-back-in-after-restart)
  * [Step 13: Verify Installed Plugins](#step-13-verify-installed-plugins)
* [Validation](#validation)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Reference](#reference)

---

## Overview

This document details the end-to-end process of installing the **Git** and **GitLab** plugins on a freshly provisioned Jenkins server. These plugins are foundational dependencies for source-code-driven CI/CD pipelines, enabling Jenkins to clone repositories, respond to GitLab webhook triggers, and display build results directly within the GitLab UI.

The implementation was performed by SSH-ing into the Jenkins host from a jump host, verifying the running Jenkins process, accessing the Jenkins web interface, and installing both plugins through the Plugin Manager with a safe restart to apply the changes.

---

## Problem Statement

The Nautilus DevOps team provisioned a new Jenkins server intended for CI/CD workloads. Before any pipeline jobs can be configured, the server requires the **Git** plugin (for source code management) and the **GitLab** plugin (for build trigger integration) to be installed. Without these plugins, Jenkins cannot interact with Git repositories or GitLab webhooks.

---

## Architecture and Context

```
Jump Host (thor@jumphost)
        |
        | SSH (jenkins@jenkins)
        v
Jenkins Server (Ubuntu 24.04 LTS)
  - Jenkins 2.541.2
  - Running on port 8080
  - Jenkins Home: /var/lib/jenkins
  - Java process: /usr/bin/java -Djava.awt.headless=true -jar /usr/share/java/jenkins.war
```

| Attribute | Value |
|---|---|
| Jenkins Version | 2.541.2 |
| Host OS | Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-94-generic x86\_64) |
| Jenkins Port | 8080 |
| Jenkins Home | /var/lib/jenkins |
| Plugins Installed | Git 5.10.1, GitLab 1.9.13 |
| Access Method | SSH from jump host |

---

## Prerequisites

* SSH access to the jump host as `thor`
* Jenkins server reachable at hostname `jenkins` from the jump host
* Jenkins web UI accessible on port 8080
* Admin credentials for Jenkins: username `admin`, password `Adm!n321`
* Outbound internet access from the Jenkins host to the Jenkins Update Center

---

## Implementation Guide

### Step 1: SSH into the Jenkins Server

From the jump host, open an SSH connection to the Jenkins server using the `jenkins` user account.

```bash
ssh jenkins@jenkins
```

Accept the host key fingerprint when prompted by typing `yes`. The server will add `jenkins` to the list of known hosts and authenticate the session.

**Expected output:**

```
The authenticity of host 'jenkins (10.244.29.234)' can't be established.
ED25519 key fingerprint is SHA256:tyX2fFBjS0A5pNlRaih+266sGG9JD0LpsFdIAX9iMzA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'jenkins' (ED25519) to the list of known hosts.
jenkins@jenkins's password:
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-94-generic x86_64)
```

Screenshot: SSH session established from thor@jumphost to jenkins@jenkins

<img width="1023" height="828" alt="image" src="https://github.com/user-attachments/assets/86b31d37-81d6-4328-a4c7-25935d751b4d" />

---

### Step 2: Verify Jenkins Process and File System Layout

Confirm that the Jenkins process is running and inspect the Jenkins home directory to understand the current state of the installation.

```bash
ps aux | grep jenkins
```

**Expected output (abbreviated):**

```
jenkins  118  0.0  1.1 9183100 756656 ?  Sl  01:01  0:26 /usr/bin/java -Djava.awt.headless=true -jar /usr/share/java/jenkins.war --webroot=/var/cache/jenkins/war --httpPort=8080
```

This confirms Jenkins is running as the `jenkins` user on port 8080.

Next, list the Jenkins home directory to confirm the installation is intact:

```bash
ls /var/lib/jenkins/
```

**Expected output:**

```
config.xml                                        nodeMonitors.xml
hudson.model.UpdateCenter.xml                     plugins
identity.key.enc                                  secret.key
jenkins.install.InstallUtil.lastExecVersion       secret.key.not-so-secret
jenkins.install.UpgradeWizard.state               secrets
jenkins.model.JenkinsLocationConfiguration.xml    updates
jenkins.security.apitoken.ApiTokenPropertyConfiguration.xml  userContent
jenkins.telemetry.Correlator.xml                  users
jobs
```

Screenshot: ps aux output showing Jenkins Java process on port 8080 and ls /var/lib/jenkins directory listing

<img width="1036" height="484" alt="image" src="https://github.com/user-attachments/assets/1024e223-0b0c-4ad2-82c7-ace82a469a15" />

---

### Step 3: Access the Jenkins Web UI

Open a browser and navigate to the Jenkins web interface on port 8080 at the lab-provided URL.

```
https://8080-port-7gzi2utfjsrkiazu.labs.kodekloud.com
```

The Jenkins login page should appear.

Screenshot: Jenkins login page rendered in browser at port 8080

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/36123f00-f40a-40fe-bec6-231dde1b14b3" />

---

### Step 4: Log In to Jenkins

Enter the administrator credentials on the Jenkins sign-in form and click **Sign in**.

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `Adm!n321` |

Screenshot: Jenkins Sign in form with username admin populated and password entered

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/36123f00-f40a-40fe-bec6-231dde1b14b3" />

After successful authentication, the Jenkins dashboard loads and displays the **Welcome to Jenkins!** landing page with no jobs configured.

Screenshot: Jenkins dashboard showing Welcome to Jenkins and empty job list

<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/8e928821-b890-4c13-88ac-6c8a654d2e2e" />

---

### Step 5: Navigate to the Plugin Manager

From the Jenkins dashboard, navigate to the Plugin Manager:

1. Click the gear icon (settings) in the top-right navigation bar to open **Manage Jenkins**
2. Under the **System Configuration** section, click **Plugins**

Screenshot: Manage Jenkins page with Plugins option highlighted under System Configuration

<img width="1919" height="953" alt="image" src="https://github.com/user-attachments/assets/9b995cc5-0a26-4e2f-88e0-ad0f8ec83957" />

This opens the Plugin Manager at:

```
/manage/pluginManager/
```

---

### Step 6: Explore the Updates Tab

The Plugin Manager opens on the **Updates** tab by default, displaying plugins with available updates. At this point, one update is listed for the **Bouncycastle API** library plugin (version 2.30.1.84-291.v9f17b\_21896e2), which is a dependency used by other plugins.

Screenshot: Plugin Manager Updates tab showing Bouncycastle API with one pending update

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/cddaa567-f471-45a7-adf8-1cf566f4e60e" />

>This tab is informational at this stage. Proceed to the **Available plugins** tab to install the required plugins.

---

### Step 7: Search and Select Git and GitLab Plugins

Click **Available plugins** in the left-hand navigation panel. In the search bar, type `gitlab` to filter the available plugin list.

From the filtered results, select the checkboxes for both:

* **Git** (version 5.10.1) -- integrates Git source code management with Jenkins (tagged: `git`, `Source Code Management`)
* **GitLab** (version 1.9.13) -- allows GitLab to trigger Jenkins builds and display results in the GitLab UI (tagged: `Build Triggers`)

Screenshot: Available plugins tab with "gitlab" search query showing Git and GitLab plugins both checked for installation

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/9d559427-fdf6-4b1e-a127-36e2ed1e023a" />

**Note:** Although the search term is `gitlab`, the Git plugin also appears in the results and must be explicitly selected in addition to the GitLab plugin.

---

### Step 8: Initiate Plugin Installation

With both **Git** and **GitLab** checkboxes selected, click the **Install** button in the top-right of the plugin list.

Jenkins will redirect to the **Download progress** page and begin resolving and downloading all required plugin dependencies.

---

### Step 9: Monitor Download Progress

The Download progress page displays the installation status for each plugin and its transitive dependencies in real time. All items should resolve to **Success**.

**Dependency chain installed (partial list):**

| Plugin | Status |
|---|---|
| commons-lang3 v3.x Jenkins API | Success |
| Ionicons API | Success |
| Structs | Success |
| Pipeline: Step API | Success |
| Credentials | Success |
| SSH Credentials | Success |
| Credentials Binding | Success |
| SCM API | Success |
| Git client | Success |
| Gson API | Success |
| Mailer | Success |
| **Git** | **Success** |
| Pipeline: API | Success |
| Pipeline: Supporting APIs | Success |
| Font Awesome API | Success |
| Bootstrap 5 API | Success |
| JUnit | Success |
| Matrix Project | Success |
| Joda Time API | Success |
| Pipeline: Job | Success |
| **GitLab** | **Success** |
| Loading plugin extensions | Success |

Screenshot: Download progress page showing first set of dependencies all resolving to Success

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/845c836d-c51d-4eaf-9a46-14465eeaa7a4" />

Screenshot: Download progress page showing second set of dependencies including Git resolving to Success

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/e9af244b-cb08-4504-84c8-fc5b1f7924c9" />

Screenshot: Download progress page showing final entries including GitLab and Loading plugin extensions resolving to Success

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/0facbe68-e7a2-4164-8f81-472f380265f0" />

---

### Step 10: Enable Restart on Completion

At the bottom of the Download progress page, check the box labeled:

> **Restart Jenkins when installation is complete and no jobs are running**

This triggers a safe restart of Jenkins once all plugin downloads complete and no active builds are running, ensuring the new plugins are fully loaded into the Jenkins runtime.

Screenshot: Download progress page bottom section showing Restart Jenkins checkbox being enabled

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/0facbe68-e7a2-4164-8f81-472f380265f0" />

---

### Step 11: Wait for Jenkins to Restart

Jenkins will initiate a restart. The browser will display the Jenkins restarting screen with the message:

> **Jenkins is restarting**
> Your browser will reload automatically when Jenkins is ready.

Do not close the browser tab. The page will automatically redirect to the login screen once Jenkins finishes restarting.

Screenshot: Jenkins restarting splash screen with loading indicator and Safe Restart note

<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/20510d18-4818-41e9-8b9c-c164899f2878" />

---

### Step 12: Log Back In After Restart

Once Jenkins finishes restarting, the browser automatically redirects to the login page. Enter the same administrator credentials to re-authenticate.

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `Adm!n321` |

Screenshot: Jenkins login page reappearing after restart with admin credentials being entered

---

### Step 13: Verify Installed Plugins

After logging back in, navigate to **Manage Jenkins > Plugins > Installed plugins**. In the search bar, type `git` to filter the installed plugin list and confirm both target plugins are present and enabled.

**Confirmed installed plugins:**

| Plugin | Version | Health | Enabled |
|---|---|---|---|
| Git client plugin | 6.6.0 | 100 | Yes |
| Git plugin | 5.10.1 | 100 | Yes |
| GitLab Plugin | 1.9.13 | 97 | Yes |

Screenshot: Installed plugins tab filtered by "git" showing Git client plugin, Git plugin, and GitLab Plugin all enabled and healthy

Both the **Git** and **GitLab** plugins are confirmed installed, enabled, and healthy. The task is complete.

---

## Validation

The following criteria confirm successful completion of this implementation:

* Jenkins process confirmed running on port 8080 via `ps aux`
* Jenkins home directory `/var/lib/jenkins` verified intact with expected files and directories
* Git plugin (version 5.10.1) visible in Installed plugins with health score 100 and enabled status
* GitLab Plugin (version 1.9.13) visible in Installed plugins with health score 97 and enabled status
* Jenkins restarted cleanly with no errors and the login page reappeared as expected

---

## Best Practices Applied

* **Safe restart strategy:** Rather than forcing an immediate restart, the "Restart Jenkins when installation is complete and no jobs are running" option was selected. This approach is non-disruptive and safe for production environments with active builds.

* **Server-side verification before UI work:** Before accessing the web UI, the Jenkins process was verified via `ps aux` and the home directory was inspected via `ls`. This step establishes a ground-truth baseline and is a sound habit when working on shared or inherited servers.

* **Targeted plugin search:** Searching for `gitlab` in the Available plugins tab surfaces both the Git and GitLab plugins in a single view, reducing navigation overhead and the risk of missing a required plugin.

* **Post-installation verification:** Navigating to Installed plugins and filtering by name after the restart confirms the plugins are not just downloaded but actively enabled in the Jenkins runtime, which is the definitive confirmation of a successful installation.

* **Dependency awareness:** The Download progress page was monitored to completion rather than navigating away prematurely. Plugin installations in Jenkins involve transitive dependency graphs, and any mid-chain failure would prevent the target plugin from loading correctly.

---

## Lessons Learned

* **The Git plugin does not appear in searches for "git" alone on the Available plugins tab if it is already bundled or pre-selected.** Searching for `gitlab` surfaces it alongside the GitLab plugin, which is a more reliable search strategy when installing both together.

* **Jenkins plugin installation resolves and downloads a substantial dependency tree.** For the GitLab plugin alone, over 30 transitive dependencies were downloaded. In environments with restricted outbound internet access or proxy requirements, this step can fail silently or partially. Always monitor the Download progress page to completion.

* **The restart is required for plugins to become active.** Plugins downloaded without a restart will appear in the installed list but may not be loaded by the Jenkins runtime. Always trigger the restart, wait for the login page to reappear, and log back in before performing any validation.

* **Health scores below 100 are not failures.** The GitLab plugin reports a health score of 97, which reflects community rating, not a broken state. The enabled toggle and absence of error banners are the authoritative indicators of plugin health post-install.

* **SSH access to the Jenkins host is a valuable diagnostic layer.** Verifying the process and file system before touching the UI ensures you are working on the correct server and that the Jenkins installation is in a known good state before making changes.

---

## Reference

| Resource | Description |
|---|---|
| [Jenkins Plugin Index](https://plugins.jenkins.io/) | Official plugin registry with documentation and changelogs |
| [Git Plugin](https://plugins.jenkins.io/git/) | Git SCM integration for Jenkins |
| [GitLab Plugin](https://plugins.jenkins.io/gitlab-plugin/) | GitLab webhook and build trigger integration |
| [Jenkins Managing Plugins](https://www.jenkins.io/doc/book/managing/plugins/) | Official documentation for plugin lifecycle management |
| `/var/lib/jenkins/plugins/` | On-disk location of all installed Jenkins plugins |












<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/976eca49-5fa0-4c5a-90b0-8c2f11f88a8e" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/943394e4-b2e5-405d-a468-0888b2bae290" />






