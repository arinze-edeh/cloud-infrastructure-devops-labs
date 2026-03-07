# Gitea Pull Request Review Workflow

> **Enterprise Git Collaboration:** Enforcing branch protection, peer code review, and controlled merges using Gitea as a self-hosted Git service in a production-simulated environment.

---

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Architecture](#architecture)
* [Infrastructure](#infrastructure)
* [Prerequisites](#prerequisites)
* [Solution Walkthrough](#solution-walkthrough)
  * [Phase 1: SSH Access to Storage Server](#phase-1-ssh-access-to-storage-server)
  * [Phase 2: Repository Inspection](#phase-2-repository-inspection)
  * [Phase 3: Gitea UI Access](#phase-3-gitea-ui-access)
  * [Phase 4: Pull Request Creation](#phase-4-pull-request-creation)
  * [Phase 5: Reviewer Assignment](#phase-5-reviewer-assignment)
  * [Phase 6: Code Review and Approval](#phase-6-code-review-and-approval)
  * [Phase 7: Merge and Closure](#phase-7-merge-and-closure)
* [Verification](#verification)
* [Key Concepts](#key-concepts)
* [Lessons Learned](#lessons-learned)
* [Repository Structure](#repository-structure)

---

## Overview

| Attribute | Detail |
|-----------|--------|
| **Category** | Git Version Control |
| **Platform** | Gitea v1.25.3 (Self-hosted) |
| **Difficulty** | Intermediate |
| **Environment** | Stratos DC (KodeKloud Lab) |
| **Skills Demonstrated** | SSH, Git branching, Pull Requests, Code Review, Protected Branch Merge |

---

## Problem Statement

The engineering team requires that **no developer pushes code directly to the `master` branch**. All changes must go through a formal Pull Request (PR) process, be reviewed and approved by a designated peer, and only then be merged. This enforces code quality gates and prevents untested or unreviewed code from reaching the production baseline.

**Specific requirements:**

* Developer `max` has written a new story and pushed it to a feature branch `story/fox-and-grapes` on the remote Gitea repository
* The `master` branch must only receive code that has been reviewed and approved
* User `tom` must be assigned as the reviewer for this PR
* `tom` must approve the PR before the merge can proceed
* The merge must be performed through the Gitea web UI, not via direct CLI push

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Stratos DC Network                   │
│                                                         │
│   ┌──────────────┐        ┌──────────────────────────┐  │
│   │  jump_host   │  SSH   │  ststor01 (Storage)      │  │
│   │  (thor)      │───────>│  user: max               │  │
│   └──────────────┘        │  /home/max/story-blog/   │  │
│                           └─────────────┬────────────┘  │
│                                         │               │
│                                    git remote           │
│                                         │               │
│                           ┌─────────────▼────────────┐  │
│                           │     Gitea Server         │  │
│                           │  sarah/story-blog repo   │  │
│                           │                          │  │
│                           │  master  <── PR #1 ───── │  │
│                           │  story/fox-and-grapes    │  │
│                           └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Branch Flow:**

```
story/fox-and-grapes  ──── PR #1 (Reviewed by tom) ────>  master
        (source)                                          (destination)
```

---

## Infrastructure

| Server | IP | User | Purpose |
|--------|----|------|---------|
| jump_host | Dynamic | thor | Entry point to Stratos DC |
| ststor01 | 172.16.238.15 | max | Storage server hosting cloned repo |
| Gitea UI | Internal | max / tom | Web-based Git portal |

---

## Prerequisites

* SSH access to `jump_host` as user `thor`
* Gitea account credentials for `max` and `tom`
* Repository `sarah/story-blog` already cloned on `ststor01` under `/home/max/`
* Feature branch `story/fox-and-grapes` already pushed to remote

---

## Solution Walkthrough

### Phase 1: SSH Access to Storage Server

**From the jump host**, establish an SSH session to the storage server as user `max`:

```bash
ssh max@ststor01.stratos.xfusioncorp.com
```

When prompted about host authenticity, type `yes` to add the host to known hosts. Enter password `Max_pass123` when prompted.

**Verify the session:**

```bash
whoami
# Expected: max

hostname
# Expected: ststor01
```

***Screenshot: Terminal showing successful SSH login as max on ststor01***
<img width="1032" height="482" alt="image" src="https://github.com/user-attachments/assets/0615d136-7ad5-4ae0-84cc-ca9810291e70" />

---

### Phase 2: Repository Inspection

**Locate the cloned repository:**

```bash
ls -la ~
```

```bash
find ~ -name ".git" -type d 2>/dev/null
# Expected: /home/max/story-blog/.git
```

**Navigate into the repository:**

```bash
cd ~/story-blog
```

**Confirm the current branch and remote state:**

```bash
git status
# Expected: On branch story/fox-and-grapes
# Expected: Your branch is up to date with 'origin/story/fox-and-grapes'
```

**List all local and remote branches:**

```bash
git branch -a
```

Expected output confirms the presence of:
* `master` (local)
* `story/fox-and-grapes` (local, active)
* `remotes/origin/master`
* `remotes/origin/story/fox-and-grapes`

**Inspect the full commit graph:**

```bash
git log --oneline --all --graph --decorate
```

```
* 7039186 (HEAD -> story/fox-and-grapes, origin/story/fox-and-grapes) Added fox-and-grapes story
*   07cc13e (origin/master, origin/HEAD, master) Merge branch 'story/frogs-and-ox'
|\
| * fd647ae Completed frogs-and-ox story
| * f9045b5 Add incomplete frogs-and-ox story
* | 25d2d8f Fix typo in story title
|/
* de26b9b Added the lion and mouse story
```

This confirms that commit `7039186` on `story/fox-and-grapes` is one commit ahead of `master`, containing the fox-and-grapes story content.

***Screenshot: Terminal output showing git branch -a and git log graph***
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/3bf2c141-1339-4fee-980a-9f835ed10b0c" />

---

### Phase 3: Gitea UI Access

1. Click the **Gitea UI** button in the lab top navigation bar
2. Log in with the following credentials:
   * **Username:** `max`
   * **Password:** `Max_pass123`
3. Navigate to the `sarah/story-blog` repository from the dashboard

***Screenshots: Gitea login page and repository landing page as user max***
<img width="1919" height="955" alt="image" src="https://github.com/user-attachments/assets/101ddc78-6eed-4ee5-afff-6b937b4663ed" />
<img width="1913" height="951" alt="image" src="https://github.com/user-attachments/assets/e6b9acc1-c611-45c9-93aa-164ae7da71f7" />

---

### Phase 4: Pull Request Creation

Upon navigating to the repository, Gitea displays a prompt banner:

> *"You pushed on branch story/fox-and-grapes 19 minutes ago"*

1. Click the green **"New Pull Request"** button on the right of that banner
2. Confirm the branch configuration on the PR form:
   * **Merge into (destination):** `sarah:master`
   * **Pull from (source):** `sarah:story/fox-and-grapes`
3. Enter the PR title exactly as:

```
Added fox-and-grapes story
```

***Screenshot: New Pull Request form showing correct source and destination branches with title filled in***
<img width="1919" height="947" alt="image" src="https://github.com/user-attachments/assets/fb5d96a0-404d-4702-904c-ffe506b38b4f" />

---

### Phase 5: Reviewer Assignment

**Before submitting the PR**, assign `tom` as the reviewer:

1. On the right sidebar, click the **gear icon** next to **"Reviewers"**
2. In the search box, type `tom`
3. Select `tom` from the dropdown list
4. Click outside the dropdown to confirm the selection
5. Verify `tom` appears under the Reviewers section
6. Click **"Create Pull Request"** to submit

***Screenshot: PR creation form with tom listed as reviewer in the sidebar***
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/8f138be1-538e-424e-b576-bcb895b7652d" />

***Screenshot Placeholder: Newly created PR #1 showing Open status, correct branch info, and tom assigned as reviewer***

---

### Phase 6: Code Review and Approval

**Log out as `max`:**

1. Click the profile avatar in the top-right corner
2. Click **"Sign Out"**

**Log in as `tom`:**

* **Username:** `tom`
* **Password:** `Tom_pass123`

**Navigate to the PR:**

1. Go to `sarah/story-blog` repository
2. Click the **"Pull Requests"** tab
3. Open **"Added fox-and-grapes story #1"**

**Review the changes:**

1. Click the **"Files Changed"** tab to inspect the diff
2. Scroll to the bottom of the page
3. Select **"Approve"** from the review options
4. Click **"Submit Review"**

The PR timeline will now display:

```
tom approved these changes
```

***Screenshot Placeholder: Files Changed tab showing the fox-and-grapes story diff***

***Screenshot Placeholder: PR Conversation tab showing "tom approved these changes" with green checkmark***

---

### Phase 7: Merge and Closure

With the review approved, proceed with the merge:

1. On the Conversation tab, click the blue **"Create merge commit"** button
   * Click the **main button text**, not the dropdown arrow on the right
2. Confirm the merge

The PR timeline will update to show:

```
tom merged commit 3db3110416 into master
Pull request successfully merged and closed
```

The PR status changes from **Open** (green) to **Merged** (purple).

***Screenshot Placeholder: PR showing Merged status badge and "Pull request successfully merged and closed" confirmation***

---

## Verification

**Full audit trail confirmed on PR #1:**

| Event | Actor | Status |
|-------|-------|--------|
| Commit pushed to `story/fox-and-grapes` | max | Confirmed |
| PR created with title "Added fox-and-grapes story" | max | Confirmed |
| Review requested from tom | max | Confirmed |
| PR approved | tom | Confirmed |
| Merge commit `3db3110416` into master | tom | Confirmed |
| PR closed | System | Confirmed |

**Optional terminal verification (run on ststor01 after merge):**

```bash
git fetch origin

git log origin/master --oneline --graph --decorate
```

The fox-and-grapes commit should now appear in the `master` branch history.

***Screenshot Placeholder: Terminal showing git log confirming fox-and-grapes commit now present in origin/master***

---

## Key Concepts

### Why Not Push Directly to Master?

Direct pushes to `master` bypass code review entirely. In production environments, this creates risk of:

* Unreviewed bugs reaching production
* No audit trail of who approved what change
* No opportunity for knowledge sharing through review comments

Branch protection rules enforce that all changes go through a reviewable PR before landing in the protected branch.

### The Role of Gitea in This Workflow

Gitea acts as the self-hosted equivalent of GitHub in this environment. It provides:

* **Repository hosting** on the internal network
* **Pull Request management** with reviewer assignment
* **Audit logging** of all review and merge events
* **Web UI** for non-CLI review interactions

### Feature Branch Strategy

```
master          A ─────────────────────────── D (merge commit)
                                              ^
story/fox-...        B ── C (feature commit) ─┘
```

* Developers never commit directly to `master`
* Each feature or story lives in its own branch
* The PR is the formal gate between feature work and the stable baseline

---

## Lessons Learned

* **Assign reviewers before submitting the PR** when possible. It creates a cleaner audit trail showing the reviewer was part of the intent from creation, not an afterthought.

* **The green banner prompt in Gitea** ("You pushed on branch X N minutes ago") is a productivity shortcut. It pre-populates the source branch on the PR form, reducing the chance of selecting the wrong branch.

* **Always verify branch direction on the PR form** before submitting. The source (pull from) and destination (merge into) must be confirmed explicitly. Reversing them would attempt to overwrite the feature branch with master.

* **The `git log --oneline --all --graph --decorate` command** is the fastest way to visualize branch divergence and confirm the feature commit is not yet in master before raising the PR.

* **Tom's approval creates a permanent, immutable audit record** in Gitea's activity log. Even if the PR is later referenced in a compliance review, the complete timeline of who reviewed, who approved, and who merged is preserved.

---


<img width="1028" height="482" alt="image" src="https://github.com/user-attachments/assets/754f42b6-171e-4420-b2c2-395bca45d1bc" />

<img width="1032" height="546" alt="image" src="https://github.com/user-attachments/assets/30cb2eb7-6fd3-47c8-96e7-d37154d8b246" />
<img width="1030" height="559" alt="image" src="https://github.com/user-attachments/assets/1a289f2e-e606-45cd-9793-1261699efa9a" />
<img width="1038" height="594" alt="image" src="https://github.com/user-attachments/assets/8874ce00-6f06-49ee-b153-aa4d028f2e0b" />
<img width="1029" height="649" alt="image" src="https://github.com/user-attachments/assets/70c25c24-5358-4500-bdd7-e26c80c74e8f" />
<img width="1031" height="724" alt="image" src="https://github.com/user-attachments/assets/766ac1b1-3037-49ab-87d2-cf49de0afdd8" />


<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/edc868cb-7b35-422b-a4d0-c1892ad3ed18" />


<img width="1919" height="953" alt="image" src="https://github.com/user-attachments/assets/15a0b08b-2399-45b4-82bc-4cdcb420aa14" />
<img width="1919" height="946" alt="image" src="https://github.com/user-attachments/assets/a85dcd66-4e2b-46df-ac38-091de856454c" />
<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/453042ce-1a12-499a-86b9-055efed846ab" />
<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/1520f998-2764-479c-af3e-73adb5a04a78" />
<img width="1050" height="912" alt="image" src="https://github.com/user-attachments/assets/24dc47a7-4a52-47a9-bfba-c5b11d791bbb" />
<img width="1919" height="947" alt="image" src="https://github.com/user-attachments/assets/147dda91-965d-48df-bb26-884af4159ae7" />
<img width="1919" height="952" alt="image" src="https://github.com/user-attachments/assets/9db44b9f-88d3-415d-a5dd-e0ea996e1e09" />
<img width="1203" height="941" alt="image" src="https://github.com/user-attachments/assets/8e2891bf-a302-4967-a58a-2a1ce90b7d2a" />
<img width="1896" height="953" alt="image" src="https://github.com/user-attachments/assets/33071ae4-d0e2-4b50-843b-268ee4116b5c" />
