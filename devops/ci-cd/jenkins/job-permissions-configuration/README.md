# Jenkins Job-Level Permission Management Using Matrix Authorization Strategy

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1: Access the Jenkins UI and Authenticate](#phase-1-access-the-jenkins-ui-and-authenticate)
  - [Phase 2: Update Available Plugin Updates](#phase-2-update-available-plugin-updates)
  - [Phase 3: Install the Matrix Authorization Strategy Plugin](#phase-3-install-the-matrix-authorization-strategy-plugin)
  - [Phase 4: Restart Jenkins After Plugin Installation](#phase-4-restart-jenkins-after-plugin-installation)
  - [Phase 5: Verify Existing Users](#phase-5-verify-existing-users)
  - [Phase 6: Verify the Packages Job Exists](#phase-6-verify-the-packages-job-exists)
  - [Phase 7: Configure Project-Based Matrix Authorization Strategy](#phase-7-configure-project-based-matrix-authorization-strategy)
  - [Phase 8: Add Admin User to Global Security Matrix and Save](#phase-8-add-admin-user-to-global-security-matrix-and-save)
  - [Phase 9: Configure Job-Level Permissions on the Packages Job](#phase-9-configure-job-level-permissions-on-the-packages-job)
- [Permission Matrix Summary](#permission-matrix-summary)
- [Validation](#validation)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Technologies Used](#technologies-used)

---

## Overview

This project documents the end-to-end process of configuring granular, job-level access control in Jenkins for two newly onboarded developers at xFusionCorp Industries. The implementation uses the **Matrix Authorization Strategy** plugin to apply scoped, per-job permissions to the `sam` and `rohan` user accounts on the existing `Packages` Jenkins job, without modifying any other job configuration.

---

## Problem Statement

xFusionCorp Industries onboarded two new developers who require controlled access to an existing Jenkins job named `Packages`. The default Jenkins authorization model does not provide per-job, per-user permission granularity out of the box. The following constraints applied:

- Two existing Jenkins users (`sam` and `rohan`) needed distinct, non-overlapping permission sets on the `Packages` job.
- The **inheritance strategy** for the job had to be set to `Inherit permissions from parent ACL`.
- No other existing job configurations were to be modified.
- The `sam` user required: **Build**, **Configure**, and **Read** permissions.
- The `rohan` user required: **Build**, **Cancel**, **Configure**, **Read**, **Update**, and **Tag** permissions.

---

## Solution Architecture

The solution leverages the **Project-based Matrix Authorization Strategy** in Jenkins, which provides two layers of authorization control:

1. **Global security matrix** at the Jenkins system level, which governs overall system access per user.
2. **Project-level security matrix** within each individual job, which grants fine-grained permissions scoped only to that job.

By enabling project-based security on the `Packages` job and setting the inheritance strategy to inherit from the parent ACL, the configuration satisfies the principle of least privilege while remaining maintainable and auditable.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Jenkins version | 2.541.2 |
| Admin credentials | Username: `admin`, Password: `Admin321` |
| Existing users | `sam` (password: `sam@pass12345`), `rohan` (password: `rohan@pass12345`) |
| Existing job | `Packages` |
| Plugin required | Matrix Authorization Strategy |
| Access method | Jenkins web UI via browser |

---

## Implementation

### Phase 1: Access the Jenkins UI and Authenticate

Navigate to the Jenkins instance URL in your browser. On the login page, enter the administrator credentials:

- **Username:** `admin`
- **Password:** `Adm!n321`

Click **Sign in** to authenticate and access the Jenkins dashboard.

> Screenshot: Jenkins login page with admin username entered and password field populated

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/5f1046ce-29b6-44ce-919c-68882ab3c4b5" />

---

### Phase 2: Update Available Plugin Updates

Before installing the Matrix Authorization Strategy plugin, navigate to **Manage Jenkins** > **Plugins** > **Updates** tab. A pending update for the **Bouncy Castle API** plugin (version `2.30.1.84-291.v9f17b_21896e2`) was visible. This library plugin is a dependency for cryptographic operations and was selected for update to maintain a healthy plugin ecosystem.

Select the checkbox next to the available update and click **Update**.

> Screenshots: Jenkins Plugin Manager Updates tab showing Bouncy Castle API plugin available for update, with the plugin checked and the Update button visible

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/1bfa0755-6471-4540-b977-59bbceee9355" />
<img width="1914" height="1018" alt="image" src="https://github.com/user-attachments/assets/5b92d958-abb0-415a-b896-bef74e1ed6e3" />

---

### Phase 3: Install the Matrix Authorization Strategy Plugin

Navigate to **Manage Jenkins** > **Plugins** > **Available plugins**. In the search field, type:

```
Matrix Authorization Strategy
```

The plugin appears in the results with version `3.2.9`, tagged under **Security** and **Authentication and User Management**. It offers matrix-based security authorization strategies for both global and per-project scopes.

Select the checkbox next to **Matrix Authorization Strategy** and click **Install**.

> Screenshot: Jenkins Available Plugins tab with Matrix Authorization Strategy 3.2.9 selected and the Install button highlighted

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/f48394f5-ab63-400f-8548-ffe25b2110b8" />

---

### Phase 4: Restart Jenkins After Plugin Installation

After initiating the install, Jenkins displays the **Download progress** page. Confirm that all installation steps completed successfully:

| Component | Status |
|---|---|
| commons-lang3 v3.x Jenkins API | Success |
| Ionicons API | Success |
| Matrix Authorization Strategy | Success |
| Loading plugin extensions | Success |

Once all statuses show **Success**, Jenkins triggers a restart. The browser displays the **Jenkins is restarting** screen. Wait for the automatic page reload, which occurs once the service is fully back online.

> Screenshot: Jenkins Download progress page showing all four plugin installation steps completed with Success status

<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/386319c2-efc2-4fcf-8f72-9c87aea45157" />

> Screenshot: Jenkins restarting screen with the spinning indicator and Safe Restart option visible

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/1154f9f8-8381-43aa-a5dd-ef278d763145" />

---

### Phase 5: Verify Existing Users

After Jenkins restarts and you log back in as `admin`, navigate to **Manage Jenkins** > **Jenkins' own user database** (via the Security Realm section or direct URL path `/manage/securityRealm/`).

Confirm that the following three users exist in the Jenkins user database:

| User ID | Name |
|---|---|
| admin | admin |
| rohan | rohan |
| sam | sam |

This confirms the user accounts are present and no account creation steps are required.

> Screenshot: Jenkins user database listing showing admin, rohan, and sam accounts with their respective User IDs and Names

<img width="1916" height="1015" alt="image" src="https://github.com/user-attachments/assets/b384e1ea-5d38-4b44-ae47-b228d14c6f17" />

---

### Phase 6: Verify the Packages Job Exists

Return to the Jenkins dashboard. Confirm that the `Packages` job is listed in the job table. The job shows `N/A` for Last Success, Last Failure, and Last Duration, indicating it has not been run in this environment, but is correctly present and enabled.

> Screenshot: Jenkins dashboard showing the Packages job listed in the All view with N/A build history columns

---

### Phase 7: Configure Project-Based Matrix Authorization Strategy

Navigate to **Manage Jenkins** > **Security** (URL path `/manage/configureSecurity/`). Under the **Authorization** section, open the dropdown and select:

```
Project-based Matrix Authorization Strategy
```

This activates the global permission matrix. The matrix displays permission categories including Overall, Agent, Job, Run, View, and SCM, with rows for `Anonymous` and `Authenticated Users` pre-populated.

At this stage, leave the global matrix rows for Anonymous and Authenticated Users with no permissions checked. Proceed to add the `admin` user in the next phase before saving.

> Screenshot: Jenkins Security configuration page with Project-based Matrix Authorization Strategy selected in the Authorization dropdown, showing the global permission matrix with Anonymous and Authenticated Users rows

---

### Phase 8: Add Admin User to Global Security Matrix and Save

Still on the **Manage Jenkins** > **Security** page, click **Add user...** and type `admin`. This adds the admin user to the global matrix.

For the `admin` row, check the **Administer** permission checkbox under the **Overall** column. This grants the admin account full system-level control and prevents administrative lockout after saving.

Click **Save** to apply the global authorization strategy change.

> Screenshot: Jenkins Security configuration page showing the admin user added to the global matrix with the Overall Administer permission checked, alongside the Anonymous and Authenticated Users rows

---

### Phase 9: Configure Job-Level Permissions on the Packages Job

Navigate to the **Packages** job by clicking its name on the dashboard, then click **Configure** in the left sidebar (URL path `/job/Packages/configure`).

In the **General** section, locate and check the **Enable project-based security** checkbox. This reveals the job-level permission matrix and the **Inheritance Strategy** dropdown.

**Step 9a: Set the Inheritance Strategy**

In the **Inheritance Strategy** dropdown, select:

```
Inherit permissions from parent ACL
```

This configures the job to inherit permissions from the global Jenkins security settings in addition to any permissions explicitly granted at the job level.

**Step 9b: Add and Configure sam**

Click **Add user...** and enter `sam`. Configure the following permissions for sam:

| Permission Category | Permission | Granted |
|---|---|---|
| Job | Build | Yes |
| Job | Cancel | No |
| Job | Configure | Yes |
| Job | Read | Yes |
| Run | Delete | No |
| Run | Update | No |
| SCM | Tag | No |

**Step 9c: Add and Configure rohan**

Click **Add user...** and enter `rohan`. Configure the following permissions for rohan:

| Permission Category | Permission | Granted |
|---|---|---|
| Job | Build | Yes |
| Job | Cancel | Yes |
| Job | Configure | Yes |
| Job | Read | Yes |
| Run | Delete | No |
| Run | Update | Yes |
| SCM | Tag | Yes |

Click **Save** to apply all job-level permission changes.

> Screenshot: Packages job Configure page showing Enable project-based security checked, Inheritance Strategy set to Inherit permissions from parent ACL, and the permission matrix with sam granted Build, Configure, and Read, and rohan granted Build, Cancel, Configure, Read, Update, and Tag

---

## Permission Matrix Summary

### Packages Job - sam

| Build | Cancel | Configure | Read | Delete | Update | Tag |
|---|---|---|---|---|---|---|
| Yes | No | Yes | No (inherited) | No | No | No |

> Note: Read is effectively granted via the inherited global ACL. The job matrix reflects Build, Configure, and Read checked for sam.

### Packages Job - rohan

| Build | Cancel | Configure | Read | Delete | Update | Tag |
|---|---|---|---|---|---|---|
| Yes | Yes | Yes | No (inherited) | No | Yes | Yes |

---

## Validation

To validate that permissions are correctly applied, log into Jenkins as each user and verify access:

**Validate sam:**
1. Log out as `admin`.
2. Log in with username `sam` and password `sam@pass12345`.
3. Navigate to the `Packages` job.
4. Confirm that the **Build Now** and **Configure** options are visible.
5. Confirm that administrative options (such as Delete) are not present.

**Validate rohan:**
1. Log out as `sam`.
2. Log in with username `rohan` and password `rohan@pass12345`.
3. Navigate to the `Packages` job.
4. Confirm that **Build Now**, **Configure**, and build management controls are visible.
5. Confirm that SCM tagging capability is accessible.
6. Confirm that administrative controls beyond the granted set are absent.

---

## Best Practices Applied

* **Principle of least privilege:** Each user received only the permissions required for their role. No blanket access was granted at the global or job level.
* **Project-based security with inheritance:** Using `Inherit permissions from parent ACL` ensures that global-level permissions flow down correctly without duplication, reducing administrative overhead.
* **Admin lockout prevention:** The `admin` user was explicitly added to the global matrix with Overall Administer before saving the authorization strategy change, preventing loss of system access.
* **Plugin updates before configuration:** Existing plugin updates were applied before installing new plugins, ensuring a stable and compatible plugin environment.
* **Restart after plugin installation:** Jenkins was restarted after plugin installation (via the safe restart mechanism) to ensure the plugin was fully loaded before attempting configuration.
* **No modification to unrelated jobs:** Only the `Packages` job configuration was modified. All other jobs and their configurations were left intact.

---

## Lessons Learned

* **Authorization strategy lockout risk:** Switching from a permissive authorization model to Project-based Matrix Authorization Strategy without first adding the admin user to the matrix will lock all users out of Jenkins. Always add the admin row with Overall Administer before saving.

* **Plugin dependency chain:** Installing the Matrix Authorization Strategy plugin also pulled in `commons-lang3 v3.x Jenkins API` and `Ionicons API` as dependencies. Reviewing the download progress page confirmed all transitive dependencies installed successfully before proceeding.

* **Jenkins UI refresh requirement:** After a Jenkins restart triggered by plugin installation or the safe restart mechanism, the browser may not automatically redirect if it loses connectivity during the restart window. Manually refreshing the login page resolves this.

* **Project-based security checkbox:** The **Enable project-based security** checkbox on a job's Configure page is not visible until the global authorization strategy is set to `Project-based Matrix Authorization Strategy`. The global security configuration must be saved first before returning to job-level configuration.

* **Inheritance strategy behavior:** Setting the inheritance strategy to `Inherit permissions from parent ACL` means that any permission granted at the global matrix level (such as Read for Authenticated Users, if configured) will also apply to this job. Understanding the layered nature of Jenkins ACL inheritance is critical to avoiding unintended access grants.

---

## Technologies Used

| Technology | Purpose |
|---|---|
| Jenkins 2.541.2 | CI/CD automation server |
| Matrix Authorization Strategy Plugin 3.2.9 | Per-project, per-user permission management |
| Bouncy Castle API Plugin | Cryptographic library dependency |
| Jenkins' own user database | Local user authentication |
| Project-based Matrix Authorization Strategy | Fine-grained job-level access control |










<img width="1919" height="1014" alt="image" src="https://github.com/user-attachments/assets/1b9a4bf2-a836-479b-b68a-2974f75a1054" />

<img width="1901" height="1007" alt="image" src="https://github.com/user-attachments/assets/5c18562b-8b27-44a5-b0b7-c3fb2e2e466b" />
<img width="1918" height="1017" alt="image" src="https://github.com/user-attachments/assets/a54f7e9f-b1de-4519-89e4-d52bfcbd12f2" />
<img width="1914" height="1010" alt="image" src="https://github.com/user-attachments/assets/368b18fa-0a14-42c4-a4a4-0ddffa1b0858" />



