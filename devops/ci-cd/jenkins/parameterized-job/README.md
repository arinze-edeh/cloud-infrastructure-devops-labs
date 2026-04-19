# Jenkins Parameterized Job: Dynamic Build Configuration with String and Choice Parameters

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Solution Architecture](#solution-architecture)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Access the Jenkins UI](#step-1-access-the-jenkins-ui)
  * [Step 2: Apply Pending Plugin Updates](#step-2-apply-pending-plugin-updates)
  * [Step 3: Restart Jenkins and Verify Plugin State](#step-3-restart-jenkins-and-verify-plugin-state)
  * [Step 4: Create the Parameterized Freestyle Job](#step-4-create-the-parameterized-freestyle-job)
  * [Step 5: Enable Parameterized Builds and Add the String Parameter](#step-5-enable-parameterized-builds-and-add-the-string-parameter)
  * [Step 6: Add the Choice Parameter](#step-6-add-the-choice-parameter)
  * [Step 7: Configure the Shell Build Step](#step-7-configure-the-shell-build-step)
  * [Step 8: Save the Job Configuration](#step-8-save-the-job-configuration)
  * [Step 9: Trigger the First Parameterized Build](#step-9-trigger-the-first-parameterized-build)
  * [Step 10: Verify Build Success and Console Output](#step-10-verify-build-success-and-console-output)
* [Console Output Verification](#console-output-verification)
* [Errors and Resolutions](#errors-and-resolutions)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)

---

## Overview

This document covers the end-to-end implementation of a parameterized Jenkins Freestyle job named `parameterized-job`. The job accepts runtime inputs via a **String parameter** (`Stage`) and a **Choice parameter** (`env`), and echoes both values during build execution. This pattern is fundamental to building flexible, reusable CI/CD pipelines that can target different build stages and deployment environments without modifying job configuration for each run.

---

## Problem Statement

A newly onboarded DevOps Engineer requires a working demonstration of Jenkins parameterized build functionality. The objective is to create a job that:

* Accepts runtime parameters to control build behavior
* Supports a free-form string input for the build stage
* Supports a predefined dropdown for the target environment
* Executes a shell step that surfaces both parameter values in the build log
* Completes at least one successful build using the `Development` environment selection

This exercise validates the engineer's understanding of parameterized pipelines and confirms that the Jenkins instance is correctly configured to support them.

---

## Solution Architecture

```
Jenkins UI (Port 8080)
       |
       v
+----------------------------+
|   parameterized-job        |
|   (Freestyle Project)      |
|                            |
|  Parameters:               |
|    Stage  [String]         |
|      default: Build        |
|    env    [Choice]         |
|      - Development         |
|      - Staging             |
|      - Production          |
|                            |
|  Build Step:               |
|    Execute Shell           |
|      echo "Stage: $Stage"  |
|      echo "env: $env"      |
+----------------------------+
       |
       v
  Console Output:
    Stage: Build
    env: Development
    Finished: SUCCESS
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Jenkins Version | 2.541.2 |
| Access Method | Web UI on port 8080 |
| Credentials | Username: `admin` / Password: `Adm!n321` |
| Job Type | Freestyle Project |
| Parameters Required | String (`Stage`), Choice (`env`) |
| Plugin Dependency | Parameterized build support (bundled in Jenkins core) |

---

## Implementation Guide

### Step 1: Access the Jenkins UI

Navigate to the Jenkins instance via the provided URL on port 8080. At the login screen, enter the following credentials:

* **Username:** `admin`
* **Password:** `Adm!n321`

Click **Sign in** to proceed to the Jenkins dashboard.

> Screenshot: Jenkins login screen with admin credentials entered

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/50ab1fc1-59e7-4a4e-af47-520435b50886" />

---

### Step 2: Apply Pending Plugin Updates

Before creating the job, apply any available plugin updates to ensure a stable and current Jenkins environment.

1. From the Jenkins dashboard, navigate to **Manage Jenkins** > **Plugins** > **Updates**
2. The **bouncycastle API** plugin update was available (upgrading from `2.30.1.82` to `2.30.1.84`)
3. Check the checkbox next to the plugin entry to select it
4. Click the **Update** button in the top-right corner

> Screenshot: Plugin Updates page showing bouncycastle API plugin selected for update

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/0f368688-125f-42e9-a2ce-57d89f41cd08" />

The plugin download progress page confirms the update status. The **bouncycastle API** plugin shows:

```
Downloaded Successfully. Will be activated during the next boot.
```

> Screenshot: Plugin Download Progress page confirming bouncycastle API downloaded successfully

<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/034f95b3-73b4-485f-befe-1d63147f6b31" />

5. Check the **"Restart Jenkins when installation is complete and no jobs are running"** checkbox at the bottom of the download progress page to trigger an automatic safe restart

---

### Step 3: Restart Jenkins and Verify Plugin State

After triggering the restart, the Jenkins UI transitions to the restart splash screen:

```
Jenkins is restarting
Your browser will reload automatically when Jenkins is ready.
```

> Screenshot: Jenkins restarting screen with spinning loader

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/9d108445-a1e0-478d-80c7-9727f3001883" />

Wait for the browser to reload automatically. Once Jenkins is back online, navigate to **Manage Jenkins** > **Plugins** > **Installed plugins** and search for `parameterized` to confirm that parameterized build support is available in the current Jenkins core installation.

> Screenshot: Installed plugins search for "parameterized" showing the search interface

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/244dd79e-ed84-4b81-84fd-713606e3e81a" />

**Note:** In Jenkins 2.x, parameterized builds are supported natively within the Jenkins core. No additional plugin installation is required. The search returns no separate plugin entry, which is the expected result confirming that the capability is bundled into core.

---

### Step 4: Create the Parameterized Freestyle Job

1. From the Jenkins dashboard, click **New Item** in the left sidebar (or select **+ Create a job** from the welcome panel)
2. In the **"Enter an item name"** field, type:

```
parameterized-job
```

3. Select **Freestyle project** as the item type
4. Click **OK** to proceed to the job configuration page

> Screenshot: New Item creation page with "parameterized-job" entered and Freestyle project selected

<img width="1914" height="1018" alt="image" src="https://github.com/user-attachments/assets/35ed3523-95fa-46d3-88f7-93c7a56fad3f" />

---

### Step 5: Enable Parameterized Builds and Add the String Parameter

On the job configuration page, scroll to the **General** section.

1. Check the **"This project is parameterized"** checkbox. This reveals the parameter definition panel
2. Click **Add Parameter** and select **String Parameter**
3. Fill in the following fields:

| Field | Value |
|---|---|
| Name | `Stage` |
| Default Value | `Build` |
| Description | (left blank) |

Leave **Trim the string** unchecked.

> Screenshot: Job Configure page showing String Parameter named "Stage" with default value "Build"

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/85f4e199-1320-4d6b-96a7-c689d1383380" />

---

### Step 6: Add the Choice Parameter

Below the String Parameter block, click **Add Parameter** again and select **Choice Parameter**.

Fill in the following fields:

| Field | Value |
|---|---|
| Name | `env` |
| Choices | `Development` (line 1) / `Staging` (line 2) / `Production` (line 3) |
| Description | (left blank) |

Each choice must be entered on a separate line in the Choices textarea. The first entry in the list (`Development`) becomes the default selected value at build time.

> Screenshot: Job Configure page showing Choice Parameter named "env" with Development, Staging, and Production listed

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/acac4f52-f3a6-40c3-b5d1-3781b0446bc2" />

---

### Step 7: Configure the Shell Build Step

Scroll down to the **Build Steps** section.

1. Click **Add build step** and select **Execute shell**
2. In the **Command** textarea, enter the following:

```bash
echo "Stage: $Stage"
echo "env: $env"
```

These shell commands reference the two parameters by their exact declared names. Jenkins injects parameters as environment variables at build time, making them accessible via standard shell `$VARIABLE` syntax.

> Screenshot: Build Steps section showing Execute shell step with echo commands for both parameters

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/dad45a92-d57b-40b8-a095-4e50c20da889" />

---

### Step 8: Save the Job Configuration

Scroll to the bottom of the configuration page and click **Save**.

Jenkins redirects to the job status page for `parameterized-job`. The **Builds** panel shows **"No builds"**, confirming the job was just created and has not yet been executed.

> Screenshot: parameterized-job status page after configuration save showing no builds

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/86494bf3-28c5-46ad-a0b3-7a6d4b15e832" />

---

### Step 9: Trigger the First Parameterized Build

From the job status page, click **Build with Parameters** in the left sidebar.

The build input form displays:

* **Stage** field pre-filled with the default value: `Build`
* **env** dropdown defaulting to: `Development`

Verify the following selections are in place before triggering:

| Parameter | Value |
|---|---|
| Stage | `Build` |
| env | `Development` |

Click the **Build** button to trigger the build.

> Screenshot: "Build with Parameters" form showing Stage=Build and env=Development with Build button visible

<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/8b8c78b3-c2ff-40c3-8557-fdfe0f0c98dd" />

---

### Step 10: Verify Build Success and Console Output

After the build completes, the job status page updates to show:

* A green checkmark next to the job title confirming a successful build
* **Build #1** listed under the Builds panel with a timestamp

Permalink entries are visible:

```
Last build (#1), 0.6 sec ago
Last stable build (#1), 0.6 sec ago
Last successful build (#1), 0.6 sec ago
Last completed build (#1), 0.6 sec ago
```

> Screenshot: parameterized-job status page showing green success indicator and Build #1 in the Builds panel

<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/d89575bb-950a-4273-b1fa-4a0ae756f259" />

Click **#1** in the Builds panel, then select **Console Output** from the left sidebar to review the full build log.

> Screenshot: Console Output for Build #1 showing full execution log

<img width="1918" height="1016" alt="image" src="https://github.com/user-attachments/assets/fb65e53a-7e8f-4de0-8506-319bd176b29c" />


---

## Console Output Verification

The console output for Build #1 confirms end-to-end execution success:

```
Started by user admin
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/parameterized-job
[parameterized-job] $ /bin/sh -xe /tmp/jenkins12639362766799860140.sh
+ echo Stage: Build
Stage: Build
+ echo env: Development
env: Development
Finished: SUCCESS
```

**Verification checklist:**

* `Stage: Build` confirms the String parameter default value was injected correctly
* `env: Development` confirms the Choice parameter selection was passed into the shell environment
* `Finished: SUCCESS` confirms zero errors during execution

---

## Errors and Resolutions

### Plugin Update Required Before Job Creation

**Observation:** Upon initial login, the Plugins section showed a pending update for the **bouncycastle API** plugin. While this plugin does not directly affect parameterized build functionality, applying outstanding updates before beginning configuration is critical in production environments to avoid unexpected behavior.

**Resolution:** The update was applied via **Manage Jenkins > Plugins > Updates**, the restart checkbox was enabled, and Jenkins was allowed to restart cleanly. All subsequent steps were performed on a fully updated instance.

---

### Jenkins UI Temporarily Unresponsive After Restart

**Observation:** After triggering the safe restart from the plugin update progress page, the Jenkins UI showed a spinning restart screen. Attempting to navigate during this window returns a blank or error page.

**Resolution:** Waited for the automatic browser reload as indicated on the restart screen. No manual intervention was required. In environments where the automatic reload does not trigger, refreshing the browser after 30 to 60 seconds resolves the issue.

---

### Parameterized Plugin Not Visible in Installed Plugins Search

**Observation:** A search for `parameterized` under **Installed plugins** returned no results, which could be interpreted as the feature being unavailable.

**Root Cause:** Jenkins 2.x bundles parameterized build support directly into the Jenkins core. There is no separately installable "Parameterized Build" plugin in modern Jenkins versions.

**Resolution:** No plugin installation was necessary. The **"This project is parameterized"** checkbox appears natively in the General section of any Freestyle or Pipeline job configuration.

---

## Best Practices

* **Apply plugin updates before beginning configuration work.** An outdated plugin state can introduce subtle rendering or functionality issues in the Jenkins UI, particularly around parameter types and build triggers.

* **Use the "Restart Jenkins when installation is complete and no jobs are running" option.** This performs a safe restart that waits for in-flight builds to complete rather than terminating them abruptly. This is the preferred restart method in shared or production Jenkins environments.

* **Name parameters using PascalCase or camelCase consistently.** Parameters are injected as shell environment variables. Inconsistent casing between the parameter declaration and the shell command reference will cause silent empty variable substitution. The implementation uses `Stage` (PascalCase) and `env` (lowercase) matching exactly in both the parameter definition and the `echo` commands.

* **Set a meaningful default value for String parameters.** The default `Build` for the `Stage` parameter ensures that accidental or automated builds without explicit parameter input still produce a valid, identifiable output rather than an empty string.

* **List the most common or safest choice first in a Choice parameter.** Jenkins selects the first entry as the default. Listing `Development` first ensures that any build triggered without explicit selection defaults to the least risky environment, not `Production`.

* **Use `$VARIABLE` syntax in shell build steps, not `${VARIABLE}`.** While both are valid in bash, the simpler `$Stage` and `$env` form is more readable and universally portable across shell implementations available on Jenkins agents.

* **Verify each build via Console Output, not just the status indicator.** A green build confirms exit code 0, but reviewing the console log confirms that parameters were injected correctly and that the expected output was produced. This is especially important when validating a parameterized job for the first time.

---

## Lessons Learned

* **Jenkins core bundles parameterized build support natively in version 2.x.** Searching for a "Parameterized Build" plugin and finding no results is not an error condition. Engineers transitioning from Jenkins 1.x should be aware that this plugin was merged into core. Attempting to install a legacy plugin of the same name may cause conflicts.

* **Parameter names are case-sensitive environment variables.** A parameter declared as `Stage` cannot be referenced as `$stage` or `$STAGE` in the shell step. This is a common source of silent failures where the build succeeds but the echo output is blank. Always match declaration and reference casing exactly.

* **The "Build with Parameters" sidebar option only appears after the job is saved with at least one parameter defined.** Prior to enabling "This project is parameterized" and saving, the standard "Build Now" option is shown. Saving the configuration replaces it with "Build with Parameters", which surfaces the parameter input form before each run.

* **Plugin restarts should be treated as maintenance windows.** Even a safe restart interrupts the Jenkins UI for all users. In team environments, coordinate plugin updates during low-traffic periods and communicate the restart window in advance.

* **Choice parameters enforce valid input at the UI level.** Unlike a String parameter where any value can be typed, a Choice parameter constrains the user to a predefined set of options. This is preferable for environment selection (`Development`, `Staging`, `Production`) where arbitrary input could cause downstream pipeline failures.

* **The workspace path reveals the job name verbatim.** The console output line `Building in workspace /var/lib/jenkins/workspace/parameterized-job` confirms that Jenkins uses the exact job name as the workspace directory. Job renaming changes this path, which can break scripts or integrations that reference the workspace path directly.
