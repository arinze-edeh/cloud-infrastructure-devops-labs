# Jenkins CI/CD User Access Configuration with Project-Based Matrix Authorization Strategy

> **Enterprise-style Jenkins user access management using the Matrix Authorization Strategy Plugin for granular, project-level permission control in CI/CD pipelines.**

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Access the Jenkins UI and Authenticate as Admin](#step-1-access-the-jenkins-ui-and-authenticate-as-admin)
  - [Step 2: Navigate to Plugin Manager and Apply Pending Updates](#step-2-navigate-to-plugin-manager-and-apply-pending-updates)
  - [Step 3: Install the Matrix Authorization Strategy Plugin](#step-3-install-the-matrix-authorization-strategy-plugin)
  - [Step 4: Restart Jenkins and Verify Plugin Installation](#step-4-restart-jenkins-and-verify-plugin-installation)
  - [Step 5: Create the Jenkins User `javed`](#step-5-create-the-jenkins-user-javed)
  - [Step 6: Configure Project-Based Matrix Authorization at the Global Level](#step-6-configure-project-based-matrix-authorization-at-the-global-level)
  - [Step 7: Grant Job-Level Read Permission to `javed` on the Existing Job](#step-7-grant-job-level-read-permission-to-javed-on-the-existing-job)
- [Verification](#verification)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)

---

## Overview

This document details the end-to-end process for integrating Jenkins into a CI/CD pipeline with properly scoped user access controls. The Nautilus engineering team required a new Jenkins server to be configured with developer-specific permissions, ensuring least-privilege access is enforced at both the global and per-project levels using the **Project-based Matrix Authorization Strategy**.

---

## Problem Statement

Following the setup of a new Jenkins server, the development team needed user access configured according to the principle of least privilege. Specifically:

- A developer user (`javed`) needed to be created with a defined full name and password.
- The `javed` user required **Overall: Read** access at the global level.
- The `javed` user required **Job: Read** access scoped only to the existing job.
- The `Anonymous` group needed all permissions removed.
- The `admin` user needed to retain full **Overall: Administer** access.
- All configuration had to use the **Project-based Matrix Authorization Strategy**, which required plugin installation and a Jenkins service restart.

---

## Solution Architecture

```
Jenkins Server (Port 8080)
+--------------------------------------------------+
|  Authorization: Project-based Matrix Strategy    |
|                                                  |
|  Global Permissions Matrix:                      |
|    admin   --> Overall: Administer               |
|    javed   --> Overall: Read                     |
|    Anonymous --> (none)                          |
|                                                  |
|  Job-Level Permissions (Helloworld):             |
|    javed   --> Job: Read                         |
+--------------------------------------------------+
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Jenkins Version | 2.541.2 |
| Admin Credentials | Username: `admin` / Password: `Admin321` |
| Access URL | Jenkins UI on port `8080` |
| Plugin Required | Matrix Authorization Strategy 3.2.9 |
| New User | Username: `javed` / Full Name: `Javed` / Password: `BruCStrnMT5` |

---

## Implementation Guide

### Step 1: Access the Jenkins UI and Authenticate as Admin

1. Open a browser and navigate to the Jenkins server URL on port `8080`.
2. On the **Sign in to Jenkins** page, enter the following credentials:
   - **Username:** `admin`
   - **Password:** `Admin321`
3. Click **Sign in**.

*Screenshot: Jenkins login page with admin credentials entered*

<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/10effea9-4969-4324-a3a8-ea4d58dff6f8" />

Upon successful authentication, the Jenkins dashboard will load, showing the existing job **Helloworld** in the jobs list with no previous build history (Last Success: N/A, Last Failure: N/A, Last Duration: N/A).

*Screenshot: Jenkins dashboard showing the Helloworld job*

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/dc1eea89-db71-4abb-a46a-27493b280969" />

---

### Step 2: Navigate to Plugin Manager and Apply Pending Updates

1. From the Jenkins dashboard, click the **gear icon** (Manage Jenkins) in the top navigation bar, or navigate to `Manage Jenkins` from the left sidebar.
2. On the **Manage Jenkins** page, locate the **System Configuration** section.
3. Click **Plugins**.

*Screenshot: Manage Jenkins page showing Plugins option under System Configuration*

<img width="1919" height="1026" alt="image" src="https://github.com/user-attachments/assets/68070ae9-a4f8-43a3-bae9-f68f64583148" />

4. On the **Plugins** page, click the **Updates** tab in the left sidebar.
5. A pending update for **bouncycastle API** (version `2.30.1.84-291.v9f17b_21896e2`) is listed.
6. Select the checkbox next to the bouncycastle API plugin.
7. Click the **Update** button to download and apply the update.

*Screenshot: Plugin Manager Updates tab showing bouncycastle API pending update selected*

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/be8d3488-efb3-4a25-b5a3-dfba8f77860e" />

8. On the **Download progress** page, confirm that bouncycastle API shows the status: **Downloaded Successfully. Will be activated during the next boot**.
9. Check the box labeled **Restart Jenkins when installation is complete and no jobs are running**.

*Screenshot: Download progress page with bouncycastle API download success status and restart checkbox selected*

<img width="1919" height="1024" alt="image" src="https://github.com/user-attachments/assets/de429754-be00-41ca-ae32-29ffcbdd9519" />

10. Wait for Jenkins to complete the restart cycle. The browser will display the Jenkins restarting screen.

*Screenshot: Jenkins restarting screen*

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/1324d3bb-670a-48ec-b846-ff563ddc595d" />

---

### Step 3: Install the Matrix Authorization Strategy Plugin

After Jenkins restarts and the login page reappears, log back in as `admin`.

1. Navigate to **Manage Jenkins** > **Plugins** > **Available plugins**.
2. In the search bar, type `matrix`.
3. From the search results, locate **Matrix Authorization Strategy** (version 3.2.9, tagged under **Security** and **Authentication and User Management**).
4. Check the checkbox next to **Matrix Authorization Strategy**.
5. Click **Install**.

*Screenshot: Available plugins page showing Matrix Authorization Strategy plugin selected for installation*

<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/0d58d15a-511d-4697-a052-8ce01bf67ca1" />

6. On the **Download progress** page, confirm all components install successfully:
   - **commons-lang3 v3.x Jenkins API:** Success
   - **Ionicons API:** Success
   - **Matrix Authorization Strategy:** Success
   - **Loading plugin extensions:** Success

*Screenshot: Download progress page showing all Matrix Authorization Strategy components installed successfully*

7. Check the box labeled **Restart Jenkins when installation is complete and no jobs are running**.

*Screenshot: Download progress page with restart checkbox selected after successful installation*

8. Wait for Jenkins to restart again.

*Screenshot: Jenkins restarting screen after Matrix Authorization Strategy installation*

---

### Step 4: Restart Jenkins and Verify Plugin Installation

After Jenkins restarts, log back in as `admin`.

1. Navigate to **Manage Jenkins** > **Plugins** > **Installed plugins**.
2. In the search bar, type `mat`.
3. Confirm that **Matrix Authorization Strategy Plugin** version **3.2.9** appears in the installed plugins list with the **Enabled** toggle active (blue).

*Screenshot: Installed plugins page confirming Matrix Authorization Strategy Plugin 3.2.9 is enabled*

---

### Step 5: Create the Jenkins User `javed`

1. Navigate to **Manage Jenkins**.
2. Under the **Security** section, click **Users**.

*Screenshot: Manage Jenkins page with Users option highlighted under Security section*

3. On the **Users** page, only the `admin` user currently exists (Users: 1).
4. Click the **+ Create User** button.

*Screenshot: Jenkins own user database showing only the admin user*

5. Fill in the **Create User** form with the following details:
   - **Username:** `javed`
   - **Password:** `BruCStrnMT5`
   - **Confirm password:** `BruCStrnMT5`
   - **Full name:** `Javed`
6. Click **Create User**.

*Screenshot: Create User form filled with username javed, password, and full name Javed*

7. The Users list now shows **Users: 2**, with both `admin` and `javed` (Full Name: `Javed`) listed.

*Screenshot: Jenkins own user database showing admin and javed users with count 2*

---

### Step 6: Configure Project-Based Matrix Authorization at the Global Level

1. Navigate to **Manage Jenkins** > **Security**.
2. On the **Security** configuration page, locate the **Authorization** section.
3. From the **Authorization** dropdown, select **Project-based Matrix Authorization Strategy**.
4. The global permissions matrix will appear with rows for **Anonymous**, **Authenticated Users**, **admin**, and **Javed**.
5. Configure permissions as follows:

   | User/Group | Permission Granted |
   |---|---|
   | `admin` | Overall: Administer (checked) |
   | `Javed` | Overall: Read (checked) |
   | `Anonymous` | All permissions unchecked (none) |
   | `Authenticated Users` | All permissions unchecked (none) |

6. Ensure the `admin` row has **Overall: Administer** checked, and the `Javed` row has **Overall: Read** checked.
7. Verify that the `Anonymous` row has no checkboxes selected.
8. Click **Save**.

*Screenshot: Security configuration page showing Project-based Matrix Authorization Strategy with admin having Administer, Javed having Overall Read, and Anonymous having no permissions*

---

### Step 7: Grant Job-Level Read Permission to `javed` on the Existing Job

1. From the Jenkins dashboard, click on the **Helloworld** job.

*Screenshot: Jenkins dashboard after saving security config, showing Helloworld job*

2. In the left sidebar of the job, click **Configure**.
3. On the **Configure** page under the **General** section, check the box labeled **Enable project-based security**.
4. The **Inheritance Strategy** dropdown will appear, set to **Inherit permissions from parent ACL**.
5. A permissions matrix will be displayed for the job scope, showing **Anonymous**, **Authenticated Users**, and **Javed** rows.
6. In the **Javed** row, check only the **Job: Read** checkbox.
7. Ensure no other checkboxes are selected for `Javed` (no Build, Cancel, Configure, Delete, Discover, Workspace, or SCM permissions).
8. Click **Save**.

*Screenshot: Helloworld job Configure page showing Enable project-based security checked, with Javed having only Job Read permission selected*

---

## Verification

After completing all steps, the configuration should satisfy the following verification criteria:

| Requirement | Status |
|---|---|
| User `javed` created with full name `Javed` | Confirmed in Users list |
| Matrix Authorization Strategy Plugin installed and enabled | Confirmed in Installed Plugins |
| `admin` has Overall: Administer at global level | Confirmed in Security config matrix |
| `javed` has Overall: Read at global level | Confirmed in Security config matrix |
| `Anonymous` has no permissions | Confirmed in Security config matrix |
| `javed` has Job: Read on Helloworld job only | Confirmed in Helloworld job Configure page |

---

## Best Practices Applied

***Least Privilege Access:*** The `javed` user was granted only the minimum permissions required, Overall: Read at the global level and Job: Read at the project level. No administrative, build, or configuration access was provisioned.

***Anonymous Access Removal:*** All permissions for the `Anonymous` group were explicitly removed, preventing unauthenticated access to any Jenkins resource. This is a critical hardening step for any Jenkins instance exposed on a network.

***Project-Based Authorization Scoping:*** Rather than using the legacy Matrix Authorization Strategy (which only supports global permissions), the Project-based Matrix Authorization Strategy was used. This enables per-job access control, allowing `javed` to view the Helloworld job without access to agent, SCM, run, or view-level operations.

***Controlled Restart Cycle:*** The option **Restart Jenkins when installation is complete and no jobs are running** was used after both plugin installation events. This ensures no running builds are interrupted during the restart, maintaining pipeline integrity.

***Explicit Admin Retention:*** Before modifying the authorization strategy, the `admin` user was pre-added to the global matrix with **Overall: Administer** to prevent accidental lockout. This is essential when switching authorization strategies.

***Plugin Hygiene:*** Pending plugin updates (bouncycastle API) were applied before installing new plugins to ensure a clean dependency chain and reduce the risk of version conflicts.

---

## Lessons Learned

**1. Never switch authorization strategies without pre-configuring admin access.**
When switching from the default authorization model to Project-based Matrix Authorization Strategy, the admin user must already be present in the matrix with Administer permission before saving. Failing to do so results in a locked-out Jenkins instance that requires filesystem-level recovery.

**2. Wait for the login page to fully reappear before interacting with Jenkins post-restart.**
After checking the restart option in the Plugin Manager, Jenkins enters a restart cycle. Attempting to interact with Jenkins before the login page fully reloads can result in incomplete page loads or session errors. Always wait for the login prompt to appear before proceeding.

**3. Do not click Finish immediately after a restart-triggered plugin installation.**
The task notes explicitly caution against clicking any Finish button immediately after restarting the service. The restart must be allowed to complete naturally via the browser's auto-reload mechanism.

**4. The Matrix Authorization Strategy Plugin must be installed before the authorization dropdown option becomes available.**
The **Project-based Matrix Authorization Strategy** option in the Authorization dropdown under Manage Jenkins > Security will not appear until the plugin is installed and Jenkins has been restarted. Attempting to configure this before installation will result in the option being absent.

**5. Job-level permissions require the Enable project-based security checkbox on the individual job.**
Global matrix permissions alone do not apply project-level overrides. Each job must have the **Enable project-based security** option checked within its own configuration to activate the per-job ACL matrix. Without this, the job inherits global-only permissions.

---

## Errors Encountered and Resolutions

### Error 1: bouncycastle API Pending Update Showed "Will be activated during the next boot"

**Symptom:** After updating the bouncycastle API plugin, the Download progress page showed a warning icon with the message "Downloaded Successfully. Will be activated during the next boot" rather than a green success checkmark.

**Root Cause:** The bouncycastle API is a library plugin that requires a full Jenkins restart to activate. Unlike some plugins that can be hot-loaded, core library plugins must be initialized during the boot sequence.

**Resolution:** The restart checkbox was selected on the download progress page, triggering a safe Jenkins restart. After the restart, the plugin was fully activated.

---

### Error 2: Matrix Authorization Strategy Plugin Not Available Before First Restart

**Symptom:** After the bouncycastle API restart, the Matrix Authorization Strategy plugin was searched under Available plugins and was not pre-installed.

**Root Cause:** This was expected behavior. The Matrix Authorization Strategy is not bundled with Jenkins by default and must be explicitly installed from the plugin marketplace.

**Resolution:** The plugin was located by searching for "matrix" in the Available plugins tab, selected, and installed. A subsequent restart was triggered to complete activation.

---

*This implementation was executed on Jenkins 2.541.2 running on a KodeKloud lab environment accessible via port 8080.*













<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/60421e48-52e6-40e8-8a48-d81a0b59ded8" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/ddbb1ffc-2aac-4340-a6fd-7ff7949b274c" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/ef966c29-0e6c-4115-a863-9e10f79f82d4" />
<img width="1919" height="1025" alt="image" src="https://github.com/user-attachments/assets/799e69cf-43e7-4cff-93fc-4ca29e4d4bd6" />
<img width="1916" height="1018" alt="image" src="https://github.com/user-attachments/assets/cf2809d5-0b8a-41df-86ac-3a29e30effe6" />
<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/c5f10db7-03cd-4fe3-8f41-20d485c2599a" />
<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/bba33709-eed3-487c-9555-abcac03afc95" />
<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/5a801dc8-2fcd-4223-b1ff-071e05084004" />
<img width="1918" height="1020" alt="image" src="https://github.com/user-attachments/assets/e0346419-4898-4b98-bec7-e3df95c1bcdb" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/4ff8b40e-be45-4e55-9850-30fe2153af41" />
<img width="1918" height="1022" alt="image" src="https://github.com/user-attachments/assets/47bc73df-f5f0-46ce-a261-3d2c960d5979" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/dcf628c6-697e-4820-9287-b8aebaf30d15" />
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/47fc3d70-92cd-4700-a06c-5705cc25984a" />
<img width="1919" height="1025" alt="image" src="https://github.com/user-attachments/assets/9631bed4-c723-4784-8822-9df13cfcef91" />
