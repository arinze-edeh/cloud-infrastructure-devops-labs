# Git Repository Fork — Gitea (Nautilus Project)

> **Enterprise Git Workflow Documentation** | Nautilus Project Team | Gitea SCM Platform

---

## 📋 Table of Contents

- [Problem Statement](#-problem-statement)
- [Environment](#-environment)
- [Prerequisites](#-prerequisites)
- [Solution Overview](#-solution-overview)
- [Step-by-Step Resolution](#-step-by-step-resolution)
  - [Step 1: Access the Gitea UI](#step-1--access-the-gitea-ui)
  - [Step 2: Authenticate as Target User](#step-2--authenticate-as-target-user)
  - [Step 3: Locate the Source Repository](#step-3--locate-the-source-repository)
  - [Step 4: Fork the Repository](#step-4--fork-the-repository)
  - [Step 5: Verify the Fork](#step-5--verify-the-fork)
- [Outcome](#-outcome)
- [Troubleshooting](#-troubleshooting)
- [Security & Best Practices](#-security--best-practices)
- [References](#-references)

---

## ❗ Problem Statement

The **Nautilus project team** onboarded a new developer, **Jon**, who requires access to a shared Git repository in order to contribute to an ongoing project. Jon does not own the source repository and must work from a personal fork to maintain proper change isolation and pull request workflows.

**Business Impact:** Without a proper fork, Jon cannot submit isolated contributions, cannot open Pull Requests, and risks directly modifying a shared team repository — violating standard GitOps change-management practices.

**Resolution Required:** Fork the `sarah/story-blog` repository under Jon's Gitea account (`jon`) via the Gitea Web UI.

---

## 🌐 Environment

| Component | Detail |
|---|---|
| **Platform** | Gitea (Self-hosted Git Service) |
| **Gitea Version** | `84aefbb` (v1.14.7) |
| **Access Point** | Gitea Web UI (browser-based) |
| **Source Repository** | `sarah/story-blog` |
| **Target Owner** | `jon` |
| **Jump Host** | `thor@jumphost` (terminal access available) |
| **Infra Stack** | KodeKloud Labs / Stratos XFusion Corp |

---

## ✅ Prerequisites

Before executing this workflow, confirm the following:

- [ ] Gitea Web UI is accessible via browser (top bar link or direct URL)
- [ ] User account `jon` exists on the Gitea instance
- [ ] Credentials are available: **username** `jon` / **password** `Jon_pass123`
- [ ] Source repository `sarah/story-blog` exists and is visible to `jon`
- [ ] `jon` does **not** already have a fork of `sarah/story-blog`

---

## 🗺️ Solution Overview

```
[Gitea UI Access]
       │
       ▼
[Login as jon]
       │
       ▼
[Navigate to sarah/story-blog]
       │
       ▼
[Click "Fork" → Set Owner: jon]
       │
       ▼
[Confirm: jon/story-blog created]
       │
       ▼
[✅ Fork verified — Jon can now contribute via PRs]
```

---

## 🔧 Step-by-Step Resolution

---

### Step 1 — Access the Gitea UI

Click the **`Gitea UI`** button in the top navigation bar of the lab environment to open the Gitea web interface in a new browser tab.

> 📌 **Screenshot Placeholder**
>
> ![Step 1 — Lab task panel showing Gitea UI button in top bar](screenshots/step1-task-panel.png)
> *Figure 1: Task panel with the `Gitea UI` button highlighted in the top navigation bar.*

---

### Step 2 — Authenticate as Target User

On the Gitea login page, authenticate using Jon's credentials:

| Field | Value |
|---|---|
| **Username or Email** | `jon` |
| **Password** | `Jon_pass123` |

Click **Sign In** to proceed.

> 📌 **Screenshot Placeholder**
>
> ![Step 2 — Gitea Sign In page with username 'jon' entered](screenshots/step2-gitea-login.png)
> *Figure 2: Gitea Sign In form with `jon` entered as the username before submission.*

---

### Step 3 — Locate the Source Repository

After successful login, the Gitea **Dashboard** is displayed. Confirm you are authenticated as `jon`. In the right-side **Repositories** panel, click on **`sarah/story-blog`** to navigate to the source repository.

> 📌 **Screenshot Placeholder**
>
> ![Step 3 — Jon's Gitea dashboard showing sarah/story-blog in the Repositories sidebar](screenshots/step3-jon-dashboard.png)
> *Figure 3: Jon's authenticated Gitea Dashboard. The `sarah/story-blog` repository is listed in the sidebar under "All (1)".*

---

### Step 4 — Fork the Repository

Inside the `sarah/story-blog` repository page, locate the **`Fork`** button in the top-right action bar (alongside Watch and Star). Click **Fork**.

On the **New Repository Fork** dialog:

| Field | Value |
|---|---|
| **Owner** | `jon` *(select from dropdown)* |
| **Repository Name** | `story-blog` *(auto-populated)* |
| **Visibility** | Leave unchecked (public) |
| **Description** | `story blog` *(auto-populated)* |

Click **Fork Repository** to confirm.

> 📌 **Screenshot Placeholder**
>
> ![Step 4a — sarah/story-blog repository page with Fork button highlighted](screenshots/step4a-source-repo.png)
> *Figure 4a: `sarah/story-blog` repository page. Fork count is 0. The Fork button is visible in the top-right action bar.*

> 📌 **Screenshot Placeholder**
>
> ![Step 4b — New Repository Fork dialog with owner set to 'jon'](screenshots/step4b-fork-dialog.png)
> *Figure 4b: Fork configuration dialog. Owner is set to `jon`, forking from `sarah/story-blog`.*

---

### Step 5 — Verify the Fork

After clicking **Fork Repository**, Gitea redirects automatically to the newly created fork: **`jon/story-blog`**.

Verify the following indicators:

- [ ] Repository header shows **`jon / story-blog`** with the fork icon (🍴)
- [ ] Subtitle reads **"forked from sarah/story-blog"**
- [ ] Commit history mirrors source: `5 Commits`, `1 Branch`, `20 KiB`
- [ ] `New Pull Request` button is visible (confirms PR workflow is enabled)
- [ ] Repository contains expected files: `frogs-and-ox.txt`, `lion-and-mouse.txt`

> 📌 **Screenshot Placeholder**
>
> ![Step 5 — jon/story-blog fork page showing 'forked from sarah/story-blog'](screenshots/step5-fork-verified.png)
> *Figure 5: Successfully forked repository `jon/story-blog`. The fork origin `sarah/story-blog` is confirmed in the subtitle. All source commits and files are present.*

---

## 🎯 Outcome

| Metric | Result |
|---|---|
| **Fork Created** | ✅ `jon/story-blog` |
| **Source Repository** | `sarah/story-blog` |
| **Commits Mirrored** | 5 |
| **Branches** | 1 (master) |
| **Pull Request Workflow** | ✅ Enabled |
| **Jon's Contribution Path** | `jon/story-blog` → PR → `sarah/story-blog` |

Jon now has a fully isolated copy of the repository and can push branches, commit changes, and submit Pull Requests back to `sarah/story-blog` following the team's standard GitOps contribution workflow.

---

## 🛠️ Troubleshooting

### Fork button is greyed out / missing
- Confirm you are logged in as `jon` (not as `sarah` or an admin)
- Verify `sarah/story-blog` is **not** already forked under `jon`'s account
- Check that `jon` has at least **read access** to `sarah/story-blog`

### Login fails with correct credentials
- Confirm the Gitea instance is reachable at the correct port
- Verify the `jon` user account exists via **Admin Panel → Users**
- Reset password via the admin account if necessary: `Admin → Edit User → Reset Password`

### Fork exists but repository content is missing
- Navigate to `jon/story-blog` and confirm **"forked from sarah/story-blog"** is shown
- If the fork is empty, delete it and re-fork — Gitea may have hit a cloning timeout

### Cannot access Gitea UI from the lab
- From the jumphost terminal, verify the Gitea service is running:
  ```bash
  curl -I http://git.stratos.xfusioncorp.com
  ```
- If port is not accessible, check the lab's top bar for the correct Gitea UI port mapping

---

## 🔐 Security & Best Practices

- **Never commit directly to the upstream `sarah/story-blog`** — always work through your fork and submit a Pull Request
- **Rotate credentials** after initial setup; the default `Jon_pass123` is a lab password and should never be used in production
- **Enable branch protection** on `sarah/story-blog` master branch to require PR reviews before merging
- **Use SSH keys** for local Git operations instead of HTTP Basic Auth:
  ```bash
  # Generate SSH key for Jon
  ssh-keygen -t ed25519 -C "jon@nautilus-project"
  # Add public key to Gitea → User Settings → SSH/GPG Keys
  ```
- **Audit fork access** periodically — remove forks from users who are no longer active on the project

---

## 📚 References

| Resource | Link |
|---|---|
| Gitea Official Documentation | https://docs.gitea.io |
| Gitea Forking Guide | https://docs.gitea.io/en-us/forks |
| Git Branching & Fork Workflow | https://git-scm.com/book/en/v2/GitHub-Contributing-to-a-Project |
| KodeKloud Nautilus Project Labs | https://kodekloud.com |

---

> **Document Owner:** Nautilus Infrastructure Team
> **Last Updated:** 2026-03-01
> **Gitea Version Tested:** v1.14.7
> **Status:** ✅ Resolved



<img width="1919" height="1025" alt="image" src="https://github.com/user-attachments/assets/21202490-a5ff-46f0-baac-9d12bb3cd075" />
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/112455a5-f2b8-4f9e-9c94-c531ef0a0988" />
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/c431944a-7c15-45f6-8386-c394c90b836e" />
<img width="1895" height="952" alt="image" src="https://github.com/user-attachments/assets/9f776b42-7d25-48f5-a90e-09f34d558ff5" />
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/4efb9d53-03fa-4eaf-b4b5-39f7e1154146" />

