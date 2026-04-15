# Jenkins CI/CD Server Installation and Initial Configuration

> **Platform:** KodeKloud / Nautilus Lab Environment
> **Category:** CI/CD Infrastructure
> **Difficulty:** Intermediate
> **Jenkins Version:** 2.541.3
> **OS:** Ubuntu 24.04.4 LTS

---

## Table of Contents

* [Overview](#overview)
* [Architecture and Context](#architecture-and-context)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1: SSH into the Jenkins Server](#step-1-ssh-into-the-jenkins-server)
  * [Step 2: Update Package Index](#step-2-update-package-index)
  * [Step 3: Install Java Runtime Environment](#step-3-install-java-runtime-environment)
  * [Step 4: Add the Jenkins GPG Signing Key](#step-4-add-the-jenkins-gpg-signing-key)
  * [Step 5: Register the Jenkins APT Repository](#step-5-register-the-jenkins-apt-repository)
  * [Step 6: Resolve GPG Signature Verification Failure](#step-6-resolve-gpg-signature-verification-failure)
  * [Step 7: Install Jenkins via APT](#step-7-install-jenkins-via-apt)
  * [Step 8: Start Jenkins and Verify Service Status](#step-8-start-jenkins-and-verify-service-status)
  * [Step 9: Retrieve the Initial Admin Password](#step-9-retrieve-the-initial-admin-password)
  * [Step 10: Unlock Jenkins via Web UI](#step-10-unlock-jenkins-via-web-ui)
  * [Step 11: Install Suggested Plugins](#step-11-install-suggested-plugins)
  * [Step 12: Create the First Admin User](#step-12-create-the-first-admin-user)
  * [Step 13: Configure the Jenkins Instance URL](#step-13-configure-the-jenkins-instance-url)
  * [Step 14: Confirm Jenkins is Ready](#step-14-confirm-jenkins-is-ready)
* [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [References](#references)

---

## Overview

This document captures the end-to-end installation and initial configuration of a Jenkins CI/CD automation server on an Ubuntu 24.04 host within a multi-node lab environment. The task was initiated as part of the DevOps CI/CD pipeline setup for **xFusionCorp Industries**, following a formal infrastructure request from the DevOps team.

The objectives of this implementation were:

* Install Jenkins on the designated `jenkins` server using the `apt` package manager
* Start the Jenkins service using the `service` command
* Complete the web-based Getting Started wizard to create the first administrative user with the specified credentials

**Admin credentials provisioned:**

| Field | Value |
|---|---|
| Username | `theadmin` |
| Password | `Admin321` |
| Full Name | `John` |
| Email | `john@jenkins.stratos.xfusioncorp.com` |

---

## Architecture and Context

```
[jumphost (thor)] --SSH--> [jenkins server (10.244.164.63)]
                                     |
                              Ubuntu 24.04 LTS
                              Jenkins 2.541.3
                              OpenJDK 17 JRE
                                     |
                        [Jenkins UI on port 8080]
```

The lab environment consists of a jump host (`jumphost`) from which all server access is proxied. The `jenkins` node is an Ubuntu 24.04 minimal installation reachable at `10.244.164.63`. Access is authenticated using the `root` user with the password `S3curePass`.

---

## Prerequisites

* SSH access from `thor@jumphost` to `root@jenkins`
* Internet connectivity from the Jenkins node to reach `pkg.jenkins.io` and `archive.ubuntu.com`
* A modern browser to access the Jenkins web UI on port 8080

---

## Implementation Guide

### Step 1: SSH into the Jenkins Server

From the jump host, initiate an SSH connection to the Jenkins node. Accept the ED25519 host key fingerprint on first connection.

```bash
ssh root@jenkins
```

When prompted, type `yes` to permanently add the host to `known_hosts`, then enter the password `S3curePass`.

*Screenshot: SSH login to jenkins node from jumphost*

<img width="1030" height="654" alt="image" src="https://github.com/user-attachments/assets/23ff453d-1879-47cc-933b-0bfd6ac33ad0" />

---

### Step 2: Update Package Index

Refresh the local APT package cache to ensure the latest package metadata is available before installing dependencies.

```bash
sudo apt update
```

The output confirms all configured repositories were reached successfully, including `download.docker.com`, `archive.ubuntu.com`, and the existing `pkg.jenkins.io` stub. At this stage, `65 packages` were identified as upgradable.

*Screenshot: apt update output confirming repository sync*

<img width="1031" height="793" alt="image" src="https://github.com/user-attachments/assets/9eac248e-77cf-4829-94be-d86566a72d65" />

---

### Step 3: Install Java Runtime Environment

Jenkins requires Java 17 or later. Install OpenJDK 17 JRE along with `fontconfig`, which is a required runtime dependency for font rendering within the Jenkins UI.

```bash
sudo apt install fontconfig openjdk-17-jre -y
```

This pulled `109` newly installed packages totalling `123 MB` of archives, including OpenJDK 17 JRE headless, Mesa graphics libraries, and all associated GTK and X11 dependencies. Post-install, `update-alternatives` automatically set `/usr/lib/jvm/java-17-openjdk-amd64/bin/java` as the default `java` binary.

*Screenshots: apt install output showing OpenJDK 17 installation completing*

<img width="1032" height="852" alt="image" src="https://github.com/user-attachments/assets/64e3da6d-385e-4f46-81db-9f2a170aecdf" />
<img width="1032" height="852" alt="image" src="https://github.com/user-attachments/assets/c64d4d02-68b4-4aa0-9f3e-72126380e2c7" />
<img width="1037" height="860" alt="image" src="https://github.com/user-attachments/assets/c10ce0ce-38bb-430b-908d-f8c281ca537f" />

---

### Step 4: Add the Jenkins GPG Signing Key

Download and store the Jenkins repository signing key in the recommended `/usr/share/keyrings/` location. This key is used by APT to verify the integrity of downloaded Jenkins packages.

```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
```

*Screenshots: The file `jenkins-keyring.asc` (3.1 KB) was saved successfully at `07:52:53 UTC`.*

<img width="1029" height="501" alt="image" src="https://github.com/user-attachments/assets/c713a79d-b988-45ea-b8a3-1d7e1a11af77" />

---

### Step 5: Register the Jenkins APT Repository

Write the Jenkins stable repository entry into `/etc/apt/sources.list.d/jenkins.list`, referencing the downloaded signing key via the `signed-by` directive.

```bash
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```
*Screenshots:*

<img width="1035" height="461" alt="image" src="https://github.com/user-attachments/assets/73539e62-72fc-4709-86d5-243bdbc435c2" />

Then refresh the package index:

```bash
sudo apt update
```

---

### Step 6: Resolve GPG Signature Verification Failure

**This step documents the primary error encountered and the multi-attempt resolution sequence.** See also: [Errors Encountered and Resolutions](#errors-encountered-and-resolutions).

The `apt update` following repository registration repeatedly failed with:

```
NO_PUBKEY 7198F4B714ABFC68
```

**Attempt 1:** Used the deprecated `apt-key adv` method to import the key from `keyserver.ubuntu.com`. The key was imported into the legacy `trusted.gpg` keyring, but the error persisted on subsequent `apt update` runs because the `signed-by` directive in `jenkins.list` still pointed to the `.asc` file rather than the legacy keyring.

```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 7198F4B714ABFC68
```

**Attempt 2:** Re-downloaded the key using `curl` and overwrote the existing `.asc` file, then re-wrote the `jenkins.list` entry. The same `NO_PUBKEY` error persisted.

```bash
sudo curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
```

**Attempt 3 (Successful):** Converted the ASCII-armored key to a binary GPG format and placed it in `/etc/apt/trusted.gpg.d/`, then updated the `jenkins.list` to reference the repo without the `signed-by` clause, relying on the system-trusted keyring instead.

```bash
sudo wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/jenkins.gpg

echo "deb https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
```

The final `apt update` succeeded. A deprecation warning appeared noting that the key was stored in the legacy `trusted.gpg` keyring, but the repository was now resolvable and Jenkins packages became available.

*Screenshot: apt update succeeding after GPG trust resolution*

<img width="1040" height="444" alt="image" src="https://github.com/user-attachments/assets/19e729aa-6ac6-4ff2-b48b-f657ce8fce83" />

---

### Step 7: Install Jenkins via APT

With the repository properly authenticated, install the `jenkins` package.

```bash
sudo apt install jenkins -y
```
*Screenshots:*

<img width="1037" height="858" alt="image" src="https://github.com/user-attachments/assets/dfe7e626-9511-441e-8ad7-b9695fed38d6" />

APT downloaded `jenkins 2.541.3` (95.8 MB) from `pkg.jenkins.io` and installed it. The post-install hook attempted to start the Jenkins service, but it was blocked by the lab environment's `policy-rc.d` restriction (`invoke-rc.d: policy-rc.d denied execution of start`). This is expected behavior in containerized and minimal Ubuntu environments. The `systemd` service unit was correctly symlinked at `/etc/systemd/system/multi-user.target.wants/jenkins.service`.

---

### Step 8: Start Jenkins and Verify Service Status

Start the Jenkins service explicitly using the `service` command, as required by the task specification.

```bash
sudo service jenkins start
```

Expected output:
```
* Starting Jenkins Automation Server jenkins
Setting up max open files limit to 8192
                                                                [ OK ]
```

Confirm the service is running:

```bash
sudo service jenkins status
```

Expected output:
```
* jenkins is running
```

*Screenshot: service jenkins status confirming running state*

<img width="1030" height="607" alt="image" src="https://github.com/user-attachments/assets/96cbf244-ba9f-4b0e-84f8-3ac2b78a89b1" />

---

### Step 9: Retrieve the Initial Admin Password

Jenkins generates a one-time initial admin password during first startup and writes it to a secrets file. Read this value to unlock the web UI.

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Output:
```
29eacffb14c146cf84d13dc3b0e7a24f
```

*Screenshot:*

<img width="1032" height="657" alt="image" src="https://github.com/user-attachments/assets/bddc6372-10bb-49e7-b478-1961491f1c53" />

This value is a single-use unlock token and should be treated as a temporary credential. It is superseded once the first admin user is created.

---

### Step 10: Unlock Jenkins via Web UI

Open a browser and navigate to the Jenkins UI on port 8080. The KodeKloud lab provides a proxied URL in the format:

```
https://8080-port-<lab-id>.labs.kodekloud.com/
```

On the **Unlock Jenkins** page, paste the initial admin password obtained in Step 9 into the **Administrator password** field, then click **Continue**.

*Screenshot: Unlock Jenkins page with administrator password entered*

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/04fe6d34-390f-48e5-a2b7-e24e793660e9" />

---

### Step 11: Install Suggested Plugins

On the **Customize Jenkins** page, select **Install suggested plugins**. Jenkins will download and install the community-recommended plugin set, including:

* Pipeline, Git, GitHub Branch Source, Pipeline Graph View
* Folders, Timestamper, Workspace Cleanup, Build Timeout
* OWASP Markup Formatter, Credentials Binding, Ant, Gradle
* Matrix Authorization Strategy, SSH Build Agents, Mailer
* Email Extension, Dark Theme, LDAP

Plugin installation proceeds automatically with a progress view showing per-plugin installation status.

*Screenshots: Getting Started plugin installation progress screen*

<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/0f386859-15f7-42db-bf80-60ab60df90fc" />
<img width="1918" height="1014" alt="image" src="https://github.com/user-attachments/assets/9cdea902-03f1-4012-ba54-b17ed555c9db" />

---

### Step 12: Create the First Admin User

On the **Create First Admin User** page, fill in the required fields using the credentials specified in the task brief:

| Field | Value |
|---|---|
| Username | `theadmin` |
| Password | `Admin321` |
| Confirm password | `Admin321` |
| Full name | `John` |
| E-mail address | `john@jenkins.stratos.xfusioncorp.com` |

Click **Save and Continue**.

*Screenshot: Create First Admin User form filled with theadmin credentials*

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/8dc413bf-6749-497f-a4b1-54f047741bae" />

---

### Step 13: Configure the Jenkins Instance URL

On the **Instance Configuration** page, Jenkins auto-populates the URL from the current browser request. Confirm the pre-filled URL:

```
https://8080-port-<lab-id>.labs.kodekloud.com/
```

Click **Save and Finish** to persist the instance URL.

*Screenshot: Instance Configuration page with Jenkins URL pre-populated*

---

### Step 14: Confirm Jenkins is Ready

The final Getting Started screen displays **Jenkins is ready!** with the message:

```
Your Jenkins setup is complete.
```

Click **Start using Jenkins** to land on the Jenkins dashboard.

*Screenshot: Jenkins is ready confirmation screen*

*Screenshot: Jenkins dashboard - Welcome to Jenkins home page*

---

## Errors Encountered and Resolutions

### Error 1: GPG Signature Verification Failure (`NO_PUBKEY 7198F4B714ABFC68`)

**Symptom:**

```
W: GPG error: https://pkg.jenkins.io/debian-stable binary/ Release:
   The following signatures couldn't be verified because the public key is not available:
   NO_PUBKEY 7198F4B714ABFC68
```

**Root Cause:**

The ASCII-armored `.asc` key file downloaded from `pkg.jenkins.io` was not being accepted by the APT `signed-by` mechanism in this Ubuntu 24.04 minimal environment. The existing pre-registered Jenkins repository stub in the base image conflicted with the new `signed-by` entry, and the key format was not being correctly parsed for signature validation.

**Resolution:**

The key was converted from ASCII-armored format to binary GPG format using `gpg --dearmor` and placed in `/etc/apt/trusted.gpg.d/jenkins.gpg`. The `jenkins.list` entry was updated to reference the repository without a `signed-by` directive, allowing APT to resolve the key from the system-trusted keyring directory instead.

```bash
sudo wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/jenkins.gpg

echo "deb https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

**Note:** A deprecation warning about the legacy `trusted.gpg` keyring appeared in the final `apt update` output. This is cosmetic in lab environments; in production, the proper fix is to ensure the `.asc` key is correctly formatted and the `signed-by` path is valid.

---

### Error 2: Jenkins Service Auto-Start Blocked by policy-rc.d

**Symptom:**

```
invoke-rc.d: policy-rc.d denied execution of start.
```

**Root Cause:**

Ubuntu minimal and Docker-based environments use a `policy-rc.d` script that blocks automatic service starts during package installation to prevent unexpected state changes in containerized systems.

**Resolution:**

Started the service manually after package installation completed:

```bash
sudo service jenkins start
```

This is the expected workflow in these environments and is consistent with the task requirement to start Jenkins using the `service` command.

---

## Best Practices Applied

* **Signed package repositories:** The Jenkins APT repository was authenticated using a cryptographic signing key before package installation, preventing supply chain compromise via unsigned packages.
* **Dedicated keyrings directory:** The GPG key was placed in `/etc/apt/trusted.gpg.d/` rather than appended to the legacy `/etc/apt/trusted.gpg`, following the modern APT keyring management approach.
* **Explicit service management:** The `service` command was used for start and status verification rather than relying on package post-install hooks, providing deterministic control.
* **Initial password rotation:** The one-time initial admin password was used solely to unlock the setup wizard; the permanent admin credentials were set through the first-run wizard, effectively rotating the bootstrap credential.
* **Principle of least necessary plugins:** The suggested plugin set was used rather than installing arbitrary plugins, keeping the initial attack surface minimal while providing full pipeline capabilities.

---

## Lessons Learned

* **The `signed-by` APT directive requires correctly formatted key files.** In Ubuntu 24.04, the `wget -O` approach to saving `.asc` keys can result in files that the `signed-by` parser rejects silently. The reliable method is to pipe through `gpg --dearmor` and write a binary `.gpg` file to `/etc/apt/trusted.gpg.d/`.

* **`apt-key adv` is deprecated and unreliable for `signed-by` scenarios.** Running `apt-key adv --recv-keys` imports a key into the legacy global keyring, which does not satisfy a `signed-by` clause pointing to a specific file path. These two mechanisms are mutually exclusive; mixing them creates confusion and persistent verification failures.

* **`policy-rc.d` auto-start denials are expected in minimal Ubuntu environments.** They are not errors in the traditional sense. Always plan for explicit `service start` commands when working in containerized or lab-grade Ubuntu nodes.

* **Jenkins 2.541.3 (LTS) setup wizard is browser-driven.** All post-install configuration including admin user creation, plugin selection, and URL binding must be completed through the UI before the instance is considered operational. CLI-based configuration via `jenkins-cli.jar` is possible but was not in scope for this task.

* **Initial admin password has a fixed path.** The file `/var/lib/jenkins/secrets/initialAdminPassword` is always present after first startup regardless of install method. Its content is a random 32-character hex string generated at runtime.

---

## References

* [Jenkins Official Installation Docs for Debian/Ubuntu](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)
* [Jenkins LTS Release Line](https://www.jenkins.io/download/lts/)
* [APT Secure Apt Key Management (Ubuntu)](https://help.ubuntu.com/community/SecureApt)
* [OpenJDK 17 on Ubuntu 24.04](https://openjdk.org/projects/jdk/17/)
* [KodeKloud Nautilus Lab Platform](https://kodekloud.com)




<img width="1034" height="783" alt="image" src="https://github.com/user-attachments/assets/dd878ce5-447f-4ed2-a135-bae071bd020c" />
<img width="1040" height="668" alt="image" src="https://github.com/user-attachments/assets/49d0e645-8d8c-47dc-9a01-54dd8001ac1a" />
<img width="1033" height="581" alt="image" src="https://github.com/user-attachments/assets/873707d1-813a-47b6-99e8-0543b9604027" />
<img width="1036" height="556" alt="image" src="https://github.com/user-attachments/assets/8ef42b88-bc5e-49c0-be62-242108df0f31" />
<img width="1037" height="458" alt="image" src="https://github.com/user-attachments/assets/f315b786-0bf9-4ce5-a6f4-4e080b3d57a8" />
<img width="1035" height="554" alt="image" src="https://github.com/user-attachments/assets/c2cbaf51-02fa-4880-b1c5-bf059f6898b2" />


<img width="1032" height="544" alt="image" src="https://github.com/user-attachments/assets/559d6aff-eb44-44c5-ac4b-b1f82dabe1a9" />



<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/297cf368-1a20-43bc-9e8a-b25bea3dcfaf" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/6328a263-ed8e-45a3-b2ce-24cdc3dfde9c" />
<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/512836ba-82dd-45b2-8384-d6636815b67f" />


