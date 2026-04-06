# Apache Tomcat Deployment on CentOS Stream 9: Java Web Application on a Custom Port

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment and Prerequisites](#environment-and-prerequisites)
- [Tools and Technologies](#tools-and-technologies)
- [Architecture Summary](#architecture-summary)
- [Deployment Steps](#deployment-steps)
  - [Step 1: SSH Access to App Server](#step-1-ssh-access-to-app-server)
  - [Step 2: System Update and Java Installation](#step-2-system-update-and-java-installation)
  - [Step 3: Install Apache Tomcat](#step-3-install-apache-tomcat)
  - [Step 4: Configure Tomcat Port](#step-4-configure-tomcat-port)
  - [Step 5: Start and Enable Tomcat Service](#step-5-start-and-enable-tomcat-service)
  - [Step 6: Transfer the WAR Artifact](#step-6-transfer-the-war-artifact)
  - [Step 7: Deploy the Application](#step-7-deploy-the-application)
  - [Step 8: Restart Tomcat to Apply Changes](#step-8-restart-tomcat-to-apply-changes)
  - [Step 9: Verify Application Availability](#step-9-verify-application-availability)
- [Outcome](#outcome)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Tags](#tags)

---

## Overview

This document details the end-to-end deployment of a Java-based web application (`ROOT.war`) on **Apache Tomcat 9** running on **CentOS Stream 9**. The deployment reconfigures Tomcat to serve traffic on a non-standard port (`3000`) and ensures the application is accessible at the root URL (`/`). The process follows production-grade procedures including artifact staging, ownership enforcement, and service validation.

---

## Problem Statement

The Nautilus project requires a Java web application to be deployed on **App Server 2 (stapp02)** within the Stratos Data Center. The application must:

- Be served by Apache Tomcat, installed fresh on the target host
- Listen on **port 3000** instead of the default port 8080
- Respond to requests at the **base URL** without any sub-path
- Persist across server reboots via systemd integration

This deployment was performed manually to validate the environment before pipeline automation.

---

## Environment and Prerequisites

| Parameter | Value |
|---|---|
| Data Center | Stratos DC |
| Target Server | App Server 2 (`stapp02`) |
| IP Address | `172.16.238.11` |
| Operating System | CentOS Stream 9 |
| Application User | `steve` |
| Application Artifact | `ROOT.war` (Java WAR file) |
| Target Port | `3000` |
| Tomcat Version | `9.0.87` |
| Jump Host User | `thor` |

**Prerequisites:**

- SSH access from the jump host (`thor@jumphost`) to `stapp02` using user `steve`
- `ROOT.war` artifact available at `/tmp/ROOT.war` on the jump host
- `sudo` privileges on `stapp02` for `steve`
- Outbound internet access on `stapp02` for package installation

---

## Tools and Technologies

- **Linux (CentOS Stream 9):** Target operating system
- **Apache Tomcat 9.0.87:** Java Servlet container and application server
- **OpenJDK 11:** Java runtime required by Tomcat
- **SCP (Secure Copy Protocol):** Artifact transfer from jump host to app server
- **Systemd / Systemctl:** Service lifecycle management and boot persistence
- **YUM / DNF:** Package management for dependency resolution

---

## Architecture Summary

```
[thor@jumphost]
      |
      | SSH (port 22)
      v
[steve@stapp02 - 172.16.238.11]
      |
      | Tomcat Service (port 3000)
      v
[/var/lib/tomcat/webapps/ROOT.war]
      |
      | Deployed and served at:
      v
http://stapp02:3000/  --> "Welcome to xFusionCorp Industries!"
```

---

## Deployment Steps

### Step 1: SSH Access to App Server

From the jump host, establish an SSH session to `stapp02` using the `steve` account. On first connection, accept the host key fingerprint to add `stapp02` to the list of known hosts.

```bash
ssh steve@stapp02
# Password: Am3ric@
```

**Screenshot: Successful SSH login to stapp02**

![SSH Login](https://github.com/user-attachments/assets/212d3909-4d0d-4e62-ae7d-76fdefa9f604)

*The terminal confirms successful authentication and presents the `[steve@stapp02 ~]$` prompt, indicating the session is active and ready.*

---

### Step 2: System Update and Java Installation

Before installing Tomcat, update all system packages to ensure a stable, patched baseline. Then install the Java 11 OpenJDK Development Kit, which is required by Tomcat 9.

```bash
sudo yum update -y
sudo yum install java-11-openjdk-devel -y
```

**Why Java 11:** Tomcat 9.x officially supports Java 8 and Java 11. Java 11 LTS is the recommended runtime for production environments as it provides long-term security and performance updates.

**Screenshot: System update and Java installation initiated**

![System Update Start](https://github.com/user-attachments/assets/1d0248ec-574f-4320-ab13-b89a811bb069)

*YUM resolves and downloads updates from the CentOS Stream 9 BaseOS and AppStream repositories, upgrading packages including core libraries, SSH, and system utilities.*

**Screenshot: System update completing**

![System Update Packages](https://github.com/user-attachments/assets/28a9e849-5e80-4478-8462-6d75c89550f5)

*The full list of upgraded packages is displayed. Packages such as `openssl-libs`, `python3`, `systemd`, and `vim` are among those updated.*

**Screenshot: Java 11 OpenJDK dependency resolution**

![Java Dependencies](https://github.com/user-attachments/assets/891e23f0-10bd-4467-8c8d-54567c3dc7a0)

*YUM resolves Java 11 dependencies including GUI and media libraries. This confirms the package manager is pulling `java-11-openjdk-devel 1:11.0.20.1.1-2.el9` from the AppStream repository.*

**Screenshot: Java 11 installation completing**

![Java Install Complete](https://github.com/user-attachments/assets/e7579392-139f-4abf-ab30-0a662d5a6698)

*Installation of all Java 11 dependencies concludes. The `Complete!` message confirms the successful installation.*

---

### Step 3: Install Apache Tomcat

With Java installed, install the Tomcat package and all required servlet API libraries using YUM.

```bash
sudo yum install tomcat -y
```

YUM resolves and installs the following components:

| Package | Version | Purpose |
|---|---|---|
| `tomcat` | `1:9.0.87-7.el9` | Core application server |
| `tomcat-lib` | `1:9.0.87-7.el9` | Shared Tomcat libraries |
| `tomcat-el-3.0-api` | `1:9.0.87-7.el9` | Expression Language API |
| `tomcat-jsp-2.3-api` | `1:9.0.87-7.el9` | JSP API |
| `tomcat-servlet-4.0-api` | `1:9.0.87-7.el9` | Servlet API |
| `tomcat-native` | `1:1.3.5-1.el9` | APR native connector (performance) |
| `ecj` | `1:4.20-17.el9` | Eclipse Compiler for Java |
| `apr` | `1.7.0-12.el9` | Apache Portable Runtime |
| `javapackages-tools` | `6.4.0-1.el9` | Java packaging utilities |

**Screenshot: Tomcat package and dependency resolution**

![Tomcat Install Resolution](https://github.com/user-attachments/assets/d8b4fd52-03d3-4c66-bbfe-a6b9845a0815)

*YUM identifies `tomcat-1:9.0.87-7.el9.noarch` and its full dependency tree. The transaction summary shows 9 packages totaling 8.7 MB to be installed.*

**Screenshot: Tomcat installation completing with verification**

![Tomcat Install Complete](https://github.com/user-attachments/assets/1b88b996-16a1-4a9a-ba88-a79dbed95cb8)

*All 9 packages pass transaction verification. The `Installed:` summary and `Complete!` message confirm Tomcat 9.0.87 is successfully installed on the system.*

---

### Step 4: Configure Tomcat Port

The default HTTP connector in Tomcat is set to port `8080`. To comply with the deployment requirement, the connector must be reconfigured to listen on port `3000` by editing the Tomcat server configuration file.

```bash
sudo vi /etc/tomcat/server.xml
```
**Screenshot:**

![Tomcat Service Enable](https://github.com/user-attachments/assets/9dc42913-2ea2-4692-b829-25fae9cb191a)


Locate the HTTP/1.1 Connector block and change the port attribute from `8080` to `3000`:

**Before:**
```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443"
           maxParameterCount="1000"
           />
```

**After:**
```xml
<Connector port="3000" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443"
           maxParameterCount="1000"
           />
```

**Screenshot: server.xml before modification (default port 8080)**

![server.xml Before](https://github.com/user-attachments/assets/58446ba4-3dbf-4713-981c-9df468747060)

*The default `server.xml` shows the Connector configured on port `8080`. The commented-out shared thread pool connector is also visible at the same default port.*

**Screenshot: server.xml after modification (port changed to 3000)**

![server.xml After](https://github.com/user-attachments/assets/34597781-359e-4062-a2dc-05f7df19cbc3)

*The active Connector entry now reflects `port="3000"`. Only the primary (uncommented) connector is modified; the commented thread-pool connector is left unchanged as it is not active.*

> **Operational Note:** Only the first active (non-commented) `<Connector>` block needs to be changed. The secondary connector in the commented `<Connector executor="tomcatThreadPool">` block does not affect runtime behavior and does not require modification.

---

### Step 5: Start and Enable Tomcat Service

Use `systemctl` to start the Tomcat service immediately and enable it to start automatically on every system boot.

```bash
sudo systemctl start tomcat
sudo systemctl enable tomcat
sudo systemctl status tomcat
```

**Screenshot: Tomcat service start and enable commands**



*The `systemctl enable` command creates the appropriate systemd symlink for boot persistence. The service is confirmed active after the start command.*

> **Best Practice:** Always run `systemctl enable` alongside `systemctl start` during initial deployments. Omitting `enable` results in the service not surviving reboots, which is a common cause of post-maintenance outages.

---

### Step 6: Transfer the WAR Artifact

The `ROOT.war` deployment artifact resides on the jump host at `/tmp/ROOT.war`. Use SCP to transfer it securely to the staging directory on `stapp02`.

This command is run **from the jump host (`thor@jumphost`)**, not from inside the app server session.

```bash
# Run from thor@jumphost
scp /tmp/ROOT.war steve@stapp02:/tmp/
```

**Screenshot: SCP transfer of ROOT.war from jump host to stapp02**

![SCP Transfer](https://github.com/user-attachments/assets/24dc75a2-370a-461f-af35-09bd715fb157)

*The SCP output confirms `ROOT.war` (4,529 bytes) was transferred at 6.7 MB/s with 100% completion. The artifact is now available at `/tmp/ROOT.war` on `stapp02`.*

> **Note:** Staging to `/tmp/` before moving to the webapps directory is a deliberate pattern. It allows ownership and integrity checks to be performed before deployment, and avoids partial writes to live directories.

---

### Step 7: Deploy the Application

After re-entering the SSH session on `stapp02`, deploy the application by:

1. Removing the default ROOT application to prevent conflicts
2. Moving the artifact into Tomcat's webapps directory
3. Correcting file ownership to allow Tomcat to read and unpack the WAR

```bash
# Remove the default ROOT application
sudo rm -rf /var/lib/tomcat/webapps/ROOT
sudo rm -rf /var/lib/tomcat/webapps/ROOT.war

# Move the staged artifact to the deployment directory
sudo mv /tmp/ROOT.war /var/lib/tomcat/webapps/

# Set correct ownership
sudo chown tomcat:tomcat /var/lib/tomcat/webapps/ROOT.war
```

**Screenshot: WAR deployment and ownership commands**

![WAR Deploy Commands](https://github.com/user-attachments/assets/f082a3a4-15c5-403b-bc43-a85530c24ade)

*The sequence of commands removes stale ROOT content, stages the new artifact, and transfers ownership to the `tomcat` service account. This ensures Tomcat has full read access to unpack and serve the application.*

> **Critical:** If the default `ROOT` directory is not removed before deploying `ROOT.war`, Tomcat may prioritize the existing directory and the new WAR will never be unpacked. Always clean the target before redeployment.

---

### Step 8: Restart Tomcat to Apply Changes

Restart the Tomcat service to force it to unpack the newly deployed `ROOT.war` and apply the port configuration change made to `server.xml`.

```bash
sudo systemctl restart tomcat
```

**Screenshot: Tomcat restart and curl verification**

![Tomcat Restart and Verify](https://github.com/user-attachments/assets/fa86c5e2-6824-4776-b579-7061d3c9de1d)

*The `sudo systemctl restart tomcat` command completes without error, confirming the service reloaded successfully with the updated configuration and new WAR artifact deployed.*

---

### Step 9: Verify Application Availability

Perform an end-to-end functional validation using `curl` to confirm the application is responding on port `3000` and returning the expected content.

```bash
curl http://stapp02:3000
```

**Screenshot: Application responding correctly on port 3000**

![Application Working](https://github.com/user-attachments/assets/fa86c5e2-6824-4776-b579-7061d3c9de1d)

*The `curl` response returns a valid HTML page with the title `SampleWebApp` and the body content `Welcome to xFusionCorp Industries!`. This confirms the ROOT application is deployed, unpacked, and serving requests correctly on port 3000.*

Expected response:

```html
<!DOCTYPE html>
<!--
To change this license header, choose License Headers in Project Properties.
To change this template file, choose Tools | Templates
and open the template in the editor.
-->
<html>
    <head>
        <title>SampleWebApp</title>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
    </head>
    <body>
        <h2>Welcome to xFusionCorp Industries!</h2>
        <br>
    </body>
</html>
```

---

## Outcome

| Objective | Status |
|---|---|
| Apache Tomcat 9.0.87 installed on CentOS Stream 9 | Completed |
| Tomcat HTTP connector reconfigured to port 3000 | Completed |
| ROOT.war artifact transferred and deployed to webapps | Completed |
| File ownership set to `tomcat:tomcat` | Completed |
| Application accessible at base URL on port 3000 | Completed |
| Service enabled for persistence across reboots | Completed |
| Functional validation via curl passed | Completed |

---

## Operational Considerations

**Firewall Rules:** In production environments, ensure that port `3000` is explicitly permitted in the host firewall (`firewalld`) and any network-level security groups. Tomcat binding to a port does not automatically open it through the OS firewall.

```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

**SELinux:** On CentOS Stream 9, SELinux is enforcing by default. If Tomcat fails to bind to a non-standard port, the SELinux port context must be updated:

```bash
sudo semanage port -a -t http_port_t -p tcp 3000
```

**Log Locations:** Monitor the following logs for deployment issues:

| Log File | Purpose |
|---|---|
| `/var/log/tomcat/catalina.out` | Tomcat startup and application logs |
| `/var/log/tomcat/localhost.log` | Web application errors |
| `journalctl -u tomcat` | Systemd service journal |

**WAR Deployment Timing:** After placing a WAR file in the `webapps` directory and restarting Tomcat, the unpacking process takes a few seconds. For large applications, allow additional time before running validation checks.

---

## Troubleshooting Reference

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `curl: (7) Failed to connect to stapp02 port 3000` | Tomcat not started or port not open | Check `systemctl status tomcat` and firewall rules |
| Tomcat starts but serves default page | Old `ROOT` directory not removed | Remove `/var/lib/tomcat/webapps/ROOT` and restart |
| `Permission denied` on WAR file | Incorrect file ownership | Run `sudo chown tomcat:tomcat /var/lib/tomcat/webapps/ROOT.war` |
| Port 3000 in use | Another process bound to the port | Run `ss -tlnp | grep 3000` to identify the conflict |
| Tomcat fails to start after `server.xml` edit | XML syntax error in configuration | Validate XML with `xmllint /etc/tomcat/server.xml` |
| Application not found after restart | WAR not unpacked yet | Wait 10-15 seconds and retry; check `catalina.out` for errors |

---

## Tags

`devops` `linux-administration` `tomcat` `java` `deployment` `centos` `systemd` `automation` `nautilus-project` `stratos-dc`




























# Tomcat Application Deployment on App Server 2



## Objective
- Install `Apache Tomcat` on `App Server 2`.

- Configure `Tomcat` to run on `port 3000`.

- Deploy Java `ROOT.war` application.

- Ensure application loads on base URL.

## Environment
- DATA CENTER: `Stratos DC.`

- SERVER: `App Server 2 (stapp02).`

- IP ADDRESS: `172.16.238.11.`

- APPLICATION: `Java-based ROOT.war.`

- PORT: `3000.`

## Tools Used
- Linux (CentOS Stream 9).

- Apache Tomcat 9.0.87.

- SCP (Secure Copy Protocol).

- Systemd (Systemctl).

## Deployment Steps

## Step 1: SSH to App Server
- Accessed stapp02 via SSH from the Jump Host using user steve.

- Authenticated with the password Am3ric@.

Screenshot:`ssh-login`
<img width="1034" height="585" alt="image" src="https://github.com/user-attachments/assets/212d3909-4d0d-4e62-ae7d-76fdefa9f604" />

## Step 2: Install Tomcat
- Ran `sudo yum install tomcat -y to install` the application server and its dependencies.

- Verified the installation of tomcat-1:9.0.87-7.el9.noarch.

Screenshots:`tomcat-installed`
<img width="1038" height="835" alt="image" src="https://github.com/user-attachments/assets/1d0248ec-574f-4320-ab13-b89a811bb069" />
<img width="1039" height="857" alt="image" src="https://github.com/user-attachments/assets/28a9e849-5e80-4478-8462-6d75c89550f5" />
<img width="1032" height="856" alt="image" src="https://github.com/user-attachments/assets/891e23f0-10bd-4467-8c8d-54567c3dc7a0" />
<img width="1031" height="853" alt="image" src="https://github.com/user-attachments/assets/e7579392-139f-4abf-ab30-0a662d5a6698" />
<img width="1033" height="855" alt="image" src="https://github.com/user-attachments/assets/d8b4fd52-03d3-4c66-bbfe-a6b9845a0815" />
<img width="1032" height="857" alt="image" src="https://github.com/user-attachments/assets/1b88b996-16a1-4a9a-ba88-a79dbed95cb8" />

## Step 3: Configure Tomcat Port
- Modified the configuration file located at /etc/tomcat/server.xml.

- Changed the default HTTP connector port from `8080` to `3000`.

Screenshots:`port-configuration`
<img width="1036" height="861" alt="image" src="https://github.com/user-attachments/assets/58446ba4-3dbf-4713-981c-9df468747060" />
<img width="1014" height="855" alt="image" src="https://github.com/user-attachments/assets/34597781-359e-4062-a2dc-05f7df19cbc3" />
<img width="1042" height="865" alt="image" src="https://github.com/user-attachments/assets/9dc42913-2ea2-4692-b829-25fae9cb191a" />

## Step 4: Start and Enable Tomcat
- Used `systemctl` to initialize and manage the Tomcat service.

- Ensured the service was configured to persist across system reboots.

Screenshot:`tomcat-running`
<img width="843" height="148" alt="image" src="https://github.com/user-attachments/assets/ce82d5cc-83f5-44f5-a58a-265dcc976fc9" />

## Step 5: Transfer Application File
- Successfully transferred `ROOT.war` from the `Jump Host` to `stapp02:/tmp/` using `SCP`.

- Confirmed the 100% transfer of the 4529-byte file.

Screenshot:`war-transfer`
<img width="1029" height="866" alt="image" src="https://github.com/user-attachments/assets/24dc75a2-370a-461f-af35-09bd715fb157" />

## Step 6: Deploy Application
- Cleaned the deployment directory by removing default ROOT files.

- Moved the artifact using `sudo mv /tmp/ROOT.war /var/lib/tomcat/webapps/`.

- Updated file ownership with `sudo chown tomcat:tomcat /var/lib/tomcat/webapps/ROOT.war`.

Screenshot:`war-deployed`
<img width="1032" height="657" alt="image" src="https://github.com/user-attachments/assets/f082a3a4-15c5-403b-bc43-a85530c24ade" />

## Step 7: Restart Tomcat
- Executed `sudo systemctl restart tomcat` to apply the new configuration and deploy the `.war` file.

Screenshot:`tomcat-restarted`
<img width="1033" height="770" alt="image" src="https://github.com/user-attachments/assets/fa86c5e2-6824-4776-b579-7061d3c9de1d" />

## Step 8: Verify Application
- Performed a final check using `curl http://stapp02:3000`.

- Confirmed the application returned the message: "Welcome to xFusionCorp Industries!".

Screenshot:`app-working`
<img width="1033" height="770" alt="image" src="https://github.com/user-attachments/assets/fa86c5e2-6824-4776-b579-7061d3c9de1d" />

## Outcome

- Tomcat successfully installed on `CentOS Stream 9`.

- Service successfully reconfigured to listen on `port 3000`.

- ROOT application successfully deployed and accessible via the base URL.

- Deployment completed successfully for the Nautilus project.

## Tags
`devops` `linux-administration` `tomcat` `deployment` `centos` `automation`
