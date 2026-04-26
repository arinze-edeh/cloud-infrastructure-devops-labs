# Jenkins SSH Build Agent Configuration Across Multi-Node Stratos Datacenter

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Environment Reference](#environment-reference)
- [Implementation](#implementation)
  - [Phase 1: CLI Setup on the Jenkins Controller](#phase-1-cli-setup-on-the-jenkins-controller)
    - [Step 1: SSH into the Jenkins Controller](#step-1-ssh-into-the-jenkins-controller)
    - [Step 2: Generate the SSH Key Pair](#step-2-generate-the-ssh-key-pair)
    - [Step 3: Distribute the Public Key to All App Servers](#step-3-distribute-the-public-key-to-all-app-servers)
    - [Step 4: Install Java 17 and Create Jenkins Workspace Directories](#step-4-install-java-17-and-create-jenkins-workspace-directories)
    - [Step 5: Validate Passwordless SSH Connectivity](#step-5-validate-passwordless-ssh-connectivity)
    - [Step 6: Export the Private Key as Base64](#step-6-export-the-private-key-as-base64)
  - [Phase 2: Jenkins UI Configuration](#phase-2-jenkins-ui-configuration)
    - [Step 7: Log In to the Jenkins Web UI](#step-7-log-in-to-the-jenkins-web-ui)
    - [Step 8: Update the Bouncy Castle API Plugin](#step-8-update-the-bouncy-castle-api-plugin)
    - [Step 9: Restart Jenkins After the Update](#step-9-restart-jenkins-after-the-update)
    - [Step 10: Install the SSH Build Agents Plugin](#step-10-install-the-ssh-build-agents-plugin)
    - [Step 11: Restart Jenkins After Plugin Installation](#step-11-restart-jenkins-after-plugin-installation)
    - [Step 12: Add SSH Credentials for App Server 1 (tony)](#step-12-add-ssh-credentials-for-app-server-1-tony)
    - [Step 13: Add SSH Credentials for App Server 2 (steve)](#step-13-add-ssh-credentials-for-app-server-2-steve)
    - [Step 14: Add SSH Credentials for App Server 3 (banner)](#step-14-add-ssh-credentials-for-app-server-3-banner)
    - [Step 15: Verify All Three Credentials](#step-15-verify-all-three-credentials)
    - [Step 16: Register App Server 1 as a Permanent Agent Node](#step-16-register-app-server-1-as-a-permanent-agent-node)
    - [Step 17: Configure App Server 1 Node Details](#step-17-configure-app-server-1-node-details)
    - [Step 18: Register App Server 2 as a Permanent Agent Node](#step-18-register-app-server-2-as-a-permanent-agent-node)
    - [Step 19: Configure App Server 2 Node Details](#step-19-configure-app-server-2-node-details)
    - [Step 20: Register App Server 3 as a Permanent Agent Node](#step-20-register-app-server-3-as-a-permanent-agent-node)
    - [Step 21: Configure App Server 3 Node Details](#step-21-configure-app-server-3-node-details)
    - [Step 22: Validate All Nodes Are Online](#step-22-validate-all-nodes-are-online)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Technologies Used](#technologies-used)

---

## Project Overview

This project documents the end-to-end configuration of three Stratos Datacenter application servers as permanent SSH build agent nodes within a Jenkins CI/CD controller environment. The setup enables Jenkins to distribute and execute pipeline workloads across all three application servers over a secure SSH channel authenticated via RSA key pairs.

---

## Problem Statement

The Nautilus DevOps team deployed a new Jenkins controller instance in the Stratos Datacenter to serve as the central CI/CD and automation hub. With three application servers available in the environment, the team needed to register all of them as Jenkins SSH build agent nodes so that Jenkins could offload job execution to these servers. No agents were configured, no SSH trust existed between Jenkins and the app servers, and no Java runtime was installed on the app servers, all of which are prerequisites for Jenkins agent operation.

---

## Solution Architecture

The solution follows a two-phase approach:

**Phase 1 (CLI):** Establish the SSH trust chain from the Jenkins controller to each application server by generating an RSA key pair on the controller, distributing the public key to all three servers, installing Java 17 on each server, creating the Jenkins workspace directory on each server, and validating passwordless authentication.

**Phase 2 (Jenkins UI):** Install and update the required Jenkins plugins (Bouncy Castle API, SSH Build Agents), store the private key as a credential for each server user, and register each application server as a permanent agent node with the correct label, remote root directory, host, and launch method.

```
Jenkins Controller (jenkins / 10.244.195.5)
        |
        |-- SSH (key-based) --> stapp01 (tony@10.244.81.15)    [App_server_1 | label: stapp01]
        |-- SSH (key-based) --> stapp02 (steve@10.244.244.178) [App_server_2 | label: stapp02]
        |-- SSH (key-based) --> stapp03 (banner@10.244.234.222)[App_server_3 | label: stapp03]
```

---

## Prerequisites

* SSH access to the Jenkins controller from the jump host
* Sudo privileges for each app server user (tony, steve, banner)
* Internet connectivity on the app servers for package installation
* Jenkins accessible at port 8080 via browser

---

## Environment Reference

| Component       | Hostname  | IP Address      | OS              | User   |
|-----------------|-----------|-----------------|-----------------|--------|
| Jenkins Controller | jenkins | 10.244.195.5   | Ubuntu 24.04    | jenkins |
| App Server 1    | stapp01   | 10.244.81.15    | AlmaLinux 9     | tony   |
| App Server 2    | stapp02   | 10.244.244.178  | AlmaLinux 9     | steve  |
| App Server 3    | stapp03   | 10.244.234.222  | AlmaLinux 9     | banner |

| Jenkins Node Name | Label   | Remote Root Directory    | Credential ID |
|-------------------|---------|--------------------------|---------------|
| App_server_1      | stapp01 | /home/tony/jenkins       | tony-key      |
| App_server_2      | stapp02 | /home/steve/jenkins      | steve-key     |
| App_server_3      | stapp03 | /home/banner/jenkins     | banner-key    |

---

## Implementation

### Phase 1: CLI Setup on the Jenkins Controller

#### Step 1: SSH into the Jenkins Controller

From the jump host, establish an SSH session to the Jenkins controller using the `jenkins` user:

```bash
ssh jenkins@jenkins
```

Accept the host key fingerprint when prompted. The session authenticates with a password and lands in the Jenkins home directory at `/var/lib/jenkins`.


>Screenshot: SSH session established to the Jenkins controller from the jump host, showing the Ubuntu 24.04 welcome banner and the jenkins@jenkins prompt

<img width="1032" height="738" alt="image" src="https://github.com/user-attachments/assets/cff5c215-7089-4fab-8bf1-cbee3bd1a56c" />

---

#### Step 2: Generate the SSH Key Pair

Generate a 4096-bit RSA key pair on the Jenkins controller. The key pair will be used exclusively for authenticating Jenkins to each application server. The private key is stored in the Jenkins home directory under `.ssh/`, and no passphrase is set to allow unattended agent launches.

```bash
ssh-keygen -t rsa -b 4096 -C "jenkins-slave-key" -f ~/.ssh/jenkins_slave_rsa -N ""
```

**Expected output:**
```
Your identification has been saved in /var/lib/jenkins/.ssh/jenkins_slave_rsa
Your public key has been saved in /var/lib/jenkins/.ssh/jenkins_slave_rsa.pub
The key fingerprint is:
SHA256:neljaW5bgUmhPCs5NAQwLzW8WNuuw+7c5GY08lfymRQ jenkins-slave-key
```

The `.ssh/` directory is created automatically since it did not previously exist.

Screenshot:

<img width="1033" height="643" alt="image" src="https://github.com/user-attachments/assets/21f93c43-1d3d-4869-8647-cdbf4eca5f6c" />

---

#### Step 3: Distribute the Public Key to All App Servers

Copy the public key to each application server using `ssh-copy-id`. The `-f` flag forces the copy even if the key may already be present. Each server prompts for a password on first connection and adds the Jenkins controller to its known hosts.

**App Server 1 (stapp01 / tony):**

```bash
ssh-copy-id -f -i ~/.ssh/jenkins_slave_rsa.pub tony@stapp01
```

Accept the host key fingerprint when prompted, then enter tony's password. Output confirms one key added.

**App Server 2 (stapp02 / steve):**

```bash
ssh-copy-id -f -i ~/.ssh/jenkins_slave_rsa.pub steve@stapp02
```

Accept the host key fingerprint, then enter steve's password.

**App Server 3 (stapp03 / banner):**

```bash
ssh-copy-id -f -i ~/.ssh/jenkins_slave_rsa.pub banner@stapp03
```

Accept the host key fingerprint, then enter banner's password.

---

#### Step 4: Install Java 17 and Create Jenkins Workspace Directories

Jenkins agents require a compatible Java runtime. Install `java-17-openjdk` on each server and create the remote root directory that Jenkins will use as the agent workspace. All commands are executed over SSH from the Jenkins controller using the password-authenticated session (the passwordless key will be used afterward).

**App Server 1:**

```bash
ssh tony@stapp01 "sudo dnf install -y java-17-openjdk && sudo mkdir -p /home/tony/jenkins && sudo chown tony:tony /home/tony/jenkins"
```

**App Server 2:**

```bash
ssh steve@stapp02 "sudo dnf install -y java-17-openjdk && sudo mkdir -p /home/steve/jenkins && sudo chown steve:steve /home/steve/jenkins"
```

**App Server 3:**

```bash
ssh banner@stapp03 "sudo dnf install -y java-17-openjdk && sudo mkdir -p /home/banner/jenkins && sudo chown banner:banner /home/banner/jenkins"
```

Each server installs `java-17-openjdk` version `1:17.0.18.0.8-2.el9` along with its headless dependency (~45 MB total). The `mkdir` and `chown` commands create and assign ownership of the workspace directory to the respective user.


>Screenshots: Terminal output showing successful Java 17 installation and directory creation on all three app servers, with "Complete!" confirmation for each dnf transaction

<img width="1033" height="655" alt="image" src="https://github.com/user-attachments/assets/8f6c061e-5f15-446a-bd79-91236440daf3" />
<img width="1040" height="869" alt="image" src="https://github.com/user-attachments/assets/45c9ef7d-5451-4076-bbe3-741a0f9552c1" />
<img width="1033" height="862" alt="image" src="https://github.com/user-attachments/assets/ff1aabf2-4756-433f-b300-015804a9c0bd" />
<img width="1033" height="410" alt="image" src="https://github.com/user-attachments/assets/27015800-d82e-4326-a945-98b6388b265d" />
<img width="1138" height="871" alt="image" src="https://github.com/user-attachments/assets/76d686c1-9d09-400c-bba5-ae87a1ab6a7a" />
<img width="1134" height="655" alt="image" src="https://github.com/user-attachments/assets/ef916c4d-ad92-42c3-bba8-c65612974e7f" />
<img width="1135" height="426" alt="image" src="https://github.com/user-attachments/assets/e89a0873-16d4-4330-90eb-759ce3b16fe3" />
<img width="1134" height="827" alt="image" src="https://github.com/user-attachments/assets/8412f702-7ea0-4848-a378-9a73cc86cb65" />
<img width="1133" height="654" alt="image" src="https://github.com/user-attachments/assets/95c5cb66-152c-418c-935b-f01b0eb885e0" />


---

#### Step 5: Validate Passwordless SSH Connectivity

Verify that the RSA key pair works correctly for passwordless authentication and that Java is accessible on each server:

```bash
ssh -i ~/.ssh/jenkins_slave_rsa tony@stapp01 "java -version && echo 'tony OK'"
ssh -i ~/.ssh/jenkins_slave_rsa steve@stapp02 "java -version && echo 'steve OK'"
ssh -i ~/.ssh/jenkins_slave_rsa banner@stapp03 "java -version && echo 'banner OK'"
```

**Expected output for all three servers:**
```
openjdk version "17.0.18" 2026-01-20 LTS
OpenJDK Runtime Environment (Red_Hat-17.0.18.0.8-2) (build 17.0.18+8-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-17.0.18.0.8-2) (build 17.0.18+8-LTS, mixed mode, sharing)
tony OK
...
steve OK
...
banner OK
```

>Screenshots: All three servers respond without a password prompt, confirming that key-based authentication is fully operational.

<img width="1137" height="427" alt="image" src="https://github.com/user-attachments/assets/008ad924-5947-43f0-b904-d469de3211b4" />

---

#### Step 6: Export the Private Key as Base64

Export the private key as a single-line base64 string. This is used for pasting the raw PEM content into the Jenkins credentials UI. The `-w 0` flag disables line wrapping to produce a continuous string.

```bash
cat ~/.ssh/jenkins_slave_rsa | base64 -w 0
```

The output is a long base64-encoded string representing the full OpenSSH private key. Copy this to a secure buffer for use in the Jenkins credential configuration steps.

>Screenshots: 

<img width="1133" height="865" alt="image" src="https://github.com/user-attachments/assets/9f5a308d-351c-4013-80f0-59abf946b020" />

---

### Phase 2: Jenkins UI Configuration

#### Step 7: Log In to the Jenkins Web UI

Open a browser and navigate to the Jenkins web interface. Log in using the provided credentials.

* **Username:** admin
* **Password:** Adm!n321


>Screenshot: Jenkins login page showing the "Sign in to Jenkins" form with "admin" entered in the username field


<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/bd9c1191-62bf-4ba9-8ae0-4a3a0c218db9" />

---

#### Step 8: Update the Bouncy Castle API Plugin

Before installing the SSH Build Agents plugin, apply the available update for the Bouncy Castle API plugin, which is a cryptography library dependency required for SSH key operations.

Navigate to: **Manage Jenkins > Plugins > Updates**

The update list shows one pending update: `bouncycastle API 2.30.1.84-291.v9f17b_21896e2`. Select the checkbox and click **Update**.


>Screenshots: Jenkins Plugin Manager Updates tab showing the Bouncy Castle API plugin selected for update with a health score of 100

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/4375afdc-b33e-4ae0-a9d9-e575bf8129e8" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/a1953827-a141-4324-a40c-3941aab347a6" />

---

#### Step 9: Restart Jenkins After the Update

After the Bouncy Castle API update is applied, Jenkins triggers a restart. The browser displays the "Jenkins is restarting" page and automatically reloads when the service comes back online.

>Screenshot: Jenkins restarting splash screen with the spinning indicator and "Your browser will reload automatically when Jenkins is ready" message

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/12965a3f-0350-40e9-b6f7-e8a49e5d43cf" />

---

#### Step 10: Install the SSH Build Agents Plugin

After Jenkins restarts, install the SSH Build Agents plugin, which provides the "Launch agents via SSH" launch method required for the node configuration.

Navigate to: **Manage Jenkins > Plugins > Available plugins**

Search for `SSH build agent`. Select **SSH Build Agents 3.1097.v868116049892** and click **Install**.

>Screenshot: Jenkins Available Plugins tab with "SSH build agent" search results showing SSH Build Agents plugin selected with a health score of 100

<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/0beb6df8-0fd8-4bd7-8472-d0fb077b621f" />

The download progress page confirms successful installation of all dependency plugins: EDDSA API, Gson API, Trilead API, commons-lang3 v3.x Jenkins API, Ionicons API, Structs, commons-text API, Credentials, Variant, SSH Credentials, SSH Build Agents, and Loading plugin extensions. All items show a green checkmark with "Success".

```
Screenshot: Jenkins plugin download progress page showing all dependencies installed successfully with green checkmarks
```

---

#### Step 11: Restart Jenkins After Plugin Installation

After the SSH Build Agents plugin and all dependencies install successfully, Jenkins restarts again. Wait for the browser to reload automatically.

```
Screenshot: Jenkins restarting splash screen appearing a second time following the SSH Build Agents plugin installation
```

---

#### Step 12: Add SSH Credentials for App Server 1 (tony)

Navigate to: **Manage Jenkins > Credentials > System > Global credentials (unrestricted)**

Click **Add Credentials**, select **SSH Username with private key**, and click **Next**.

Configure the credential as follows:

* **Scope:** Global (Jenkins, nodes, items, all child items, etc)
* **ID:** tony-key
* **Description:** (leave blank)
* **Username:** tony
* **Private Key:** Enter directly, then paste the full OpenSSH private key content (the raw PEM text beginning with `-----BEGIN OPENSSH PRIVATE KEY-----`)

Click **Create**.

```
Screenshot: Jenkins "Add SSH Username with private key" credential form showing ID "tony-key", username "tony", and the private key field populated with the OpenSSH PEM content
```

---

#### Step 13: Add SSH Credentials for App Server 2 (steve)

Click **Add Credentials** again. Select **SSH Username with private key** and configure as follows:

* **Scope:** Global
* **ID:** steve-key
* **Username:** steve
* **Private Key:** Enter directly, paste the same private key

Click **Create**.

```
Screenshot: Jenkins credential form for "steve-key" showing username "steve" and the private key field populated, with "tony-key / tony" already visible in the credentials list background
```

---

#### Step 14: Add SSH Credentials for App Server 3 (banner)

Click **Add Credentials** again. Select **SSH Username with private key** and configure as follows:

* **Scope:** Global
* **ID:** banner-key
* **Username:** banner
* **Private Key:** Enter directly, paste the same private key

Click **Create**.

```
Screenshot: Jenkins credential form for "banner-key" showing username "banner" and the private key field populated, with "tony-key" and "steve-key" already visible in the credentials list background
```

---

#### Step 15: Verify All Three Credentials

After all three credentials are created, the Global credentials page lists all entries:

| ID          | Username |
|-------------|----------|
| tony-key    | tony     |
| steve-key   | steve    |
| banner-key  | banner   |

```
Screenshot: Jenkins Global credentials page showing three SSH credentials listed: tony-key (tony), steve-key (steve), and banner-key (banner)
```

---

#### Step 16: Register App Server 1 as a Permanent Agent Node

Navigate to: **Manage Jenkins > Nodes > New Node**

Enter the node name `App_server_1`, select **Permanent Agent**, and click **Create**.

```
Screenshot: Jenkins "New node" page with "App_server_1" entered as the node name and "Permanent Agent" selected
```

---

#### Step 17: Configure App Server 1 Node Details

On the node configuration page, fill in the following fields:

* **Name:** App_server_1
* **Number of Executors:** 1
* **Remote root directory:** /home/tony/jenkins
* **Labels:** stapp01
* **Usage:** Use this node as much as possible
* **Launch method:** Launch agents via SSH
  * **Host:** stapp01
  * **Credentials:** tony (select the `tony-key` credential)
  * **Host Key Verification Strategy:** Non verifying Verification Strategy
* **Availability:** Keep this agent online as much as possible

Click **Save**.

```
Screenshot: Jenkins node configuration form for App_server_1 showing remote root directory "/home/tony/jenkins", label "stapp01", host "stapp01", credentials set to "tony", and Host Key Verification Strategy set to "Non verifying Verification Strategy"
```

---

#### Step 18: Register App Server 2 as a Permanent Agent Node

Navigate to: **Manage Jenkins > Nodes > New Node**

Enter the node name `App_server_2`, select **Permanent Agent**, and click **Create**.

```
Screenshot: Jenkins "New node" page with "App_server_2" entered as the node name and "Permanent Agent" selected
```

---

#### Step 19: Configure App Server 2 Node Details

On the node configuration page, fill in the following fields:

* **Name:** App_server_2
* **Number of Executors:** 1
* **Remote root directory:** /home/steve/jenkins
* **Labels:** stapp02
* **Usage:** Use this node as much as possible
* **Launch method:** Launch agents via SSH
  * **Host:** stapp02
  * **Credentials:** steve (select the `steve-key` credential)
  * **Host Key Verification Strategy:** Non verifying Verification Strategy
* **Availability:** Keep this agent online as much as possible

Click **Save**.

```
Screenshot: Jenkins node configuration form for App_server_2 showing remote root directory "/home/steve/jenkins", label "stapp02", host "stapp02", credentials set to "steve", and Host Key Verification Strategy set to "Non verifying Verification Strategy"
```

---

#### Step 20: Register App Server 3 as a Permanent Agent Node

Navigate to: **Manage Jenkins > Nodes > New Node**

Enter the node name `App_server_3`, select **Permanent Agent**, and click **Create**.

```
Screenshot: Jenkins "New node" page with "App_server_3" entered as the node name and "Permanent Agent" selected
```

---

#### Step 21: Configure App Server 3 Node Details

On the node configuration page, fill in the following fields:

* **Name:** App_server_3
* **Number of Executors:** 1
* **Remote root directory:** /home/banner/jenkins
* **Labels:** stapp03
* **Usage:** Use this node as much as possible
* **Launch method:** Launch agents via SSH
  * **Host:** stapp03
  * **Credentials:** banner (select the `banner-key` credential)
  * **Host Key Verification Strategy:** Non verifying Verification Strategy
* **Availability:** Keep this agent online as much as possible

Click **Save**.

```
Screenshot: Jenkins node configuration form for App_server_3 showing remote root directory "/home/banner/jenkins", label "stapp03", host "stapp03", credentials set to "banner", and Host Key Verification Strategy set to "Non verifying Verification Strategy"
```

---

#### Step 22: Validate All Nodes Are Online

Navigate to: **Manage Jenkins > Nodes**

All three nodes, along with the Built-In Node, are listed in the Nodes table. All show their architecture as `Linux (amd64)`, clock status as `In sync`, and valid response times, confirming they are online and accepting connections from the Jenkins controller.

| Node Name    | Architecture  | Clock Difference | Free Disk Space | Response Time |
|--------------|---------------|------------------|-----------------|---------------|
| App_server_1 | Linux (amd64) | In sync          | 339.99 GiB      | 23ms          |
| App_server_2 | Linux (amd64) | In sync          | 339.30 GiB      | 39ms          |
| App_server_3 | Linux (amd64) | In sync          | 337.40 GiB      | 91ms          |
| Built-In Node | Linux (amd64) | In sync          | 336.93 GiB      | 0ms           |

```
Screenshot: Jenkins Nodes overview page listing App_server_1, App_server_2, App_server_3, and Built-In Node, all showing "In sync" clock status and active response times, confirming all agents are online
```

---

## Best Practices Applied

* **Dedicated key pair per role:** A single RSA 4096-bit key pair was generated specifically for Jenkins agent authentication, keeping it isolated from any user personal keys.
* **No passphrase on the agent key:** Passphrase-free keys are standard practice for automated Jenkins agent launches, since Jenkins cannot interactively enter a passphrase at connection time.
* **`-f` flag on ssh-copy-id:** Ensures idempotent key distribution even if a partial or conflicting key state exists.
* **Ownership-correct workspace directories:** The `chown` step ensures the Jenkins agent process, running as the OS user, has full write access to its workspace without requiring elevated permissions during build execution.
* **Non verifying Host Key Verification Strategy:** Selected deliberately in a controlled Stratos Datacenter environment where host key management is handled at the network/provisioning layer. In production environments with stronger security requirements, use "Known hosts file" or "Manually provided key" strategies instead.
* **Credential-per-user model:** Each app server user received its own Jenkins credential entry (tony-key, steve-key, banner-key), enabling per-node audit trails and the ability to rotate individual credentials without affecting the others.
* **Plugin update before installation:** Updating the Bouncy Castle API plugin before installing SSH Build Agents ensures cryptographic compatibility and avoids runtime dependency conflicts.
* **Jenkins restart between each plugin operation:** Restarting after updates and after installation ensures a clean plugin load cycle and prevents state inconsistencies.

---

## Lessons Learned

* **Java must be installed before agent registration, not after.** Jenkins will attempt to connect and launch the agent at the moment the node is saved. If Java is absent, the connection succeeds at the SSH level but the agent process fails to start immediately. Installing Java first eliminates this failure mode entirely.
* **The SSH key must be the raw PEM content, not base64.** The Jenkins credentials UI for "SSH Username with private key" expects the actual PEM-formatted private key (starting with `-----BEGIN OPENSSH PRIVATE KEY-----`). The base64 export step is useful for inspection but the raw key text is what gets pasted into the Jenkins form.
* **`ssh-copy-id` accepts the first connection via password only.** Subsequent connections in the same session reuse the established known_hosts entry. This is why the host key fingerprint prompt appears only once per server even though multiple commands are run against it.
* **Plugin dependency chains can be large.** Installing SSH Build Agents pulled in 10 additional dependency plugins. Reviewing the download progress page confirms all dependencies installed cleanly before proceeding.
* **Bouncy Castle API must be current for RSA 4096 keys to parse correctly.** An outdated Bouncy Castle library can silently fail to parse modern OpenSSH private key formats. Updating it as the first plugin operation prevents cryptographic parsing errors when adding SSH credentials.
* **"Non verifying Verification Strategy" is acceptable for controlled environments.** In a datacenter where all node IPs are known and network access is restricted, this strategy removes the operational burden of pre-populating known_hosts on the Jenkins controller. In public-cloud or zero-trust environments, switch to a stricter strategy.

---

## Technologies Used

| Technology | Role |
|---|---|
| Jenkins 2.541.2 | CI/CD controller and agent orchestration |
| SSH Build Agents Plugin 3.1097 | SSH-based agent launch method |
| Bouncy Castle API Plugin 2.30.1.84 | Cryptographic library for SSH key parsing |
| OpenSSH (RSA 4096) | Key-based authentication between controller and agents |
| Java 17 OpenJDK (Red Hat 17.0.18) | Jenkins agent runtime on AlmaLinux 9 app servers |
| Ubuntu 24.04 LTS | Jenkins controller operating system |
| AlmaLinux 9 | App server operating system |
| dnf | Package manager for Java installation on app servers |





<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/d663e3b4-ba75-42ab-aa89-bcd8e2d7419b" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/a0944062-aa48-4d68-97e4-72eea77c3dbf" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/816646f7-3f47-4333-b534-e9f4a8f63910" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/eaf222f9-60a8-4ee9-a2c4-e3a6b61a8141" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/41376449-ae6c-4243-9bfc-c6327b0af61d" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/c95c787d-58e7-4257-b0f0-47f216ad8a5c" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/ad105f60-72b4-4b8c-8f3a-77e3125942f5" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/8603dac8-ea68-4b7a-9512-ee91d3a560d9" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/22c55f0e-7ad2-41c2-9583-01c04b7e9c06" />

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/37ba46b5-ccde-43ab-8a80-cc42c09de134" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/4813aed9-9452-409b-ab63-264bfaa4ef47" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/bc0ed2d7-ee44-4491-9a9e-9983178fa2df" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/c15e67b4-8384-40ab-ba7a-35510c66ac61" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/ef69db2d-3500-464f-8985-1a43f686b87e" />
<img width="1919" height="1010" alt="image" src="https://github.com/user-attachments/assets/aa06e58c-cff5-4d3f-bfd1-f7f1700d80a1" />
<img width="1919" height="1009" alt="image" src="https://github.com/user-attachments/assets/830e9638-93cc-4043-ae21-7ee4c13e6a53" />
