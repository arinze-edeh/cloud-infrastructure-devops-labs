# Git Version Control

> Production-simulated Git operations across self-hosted Gitea, bare repository infrastructure, and multi-user Linux storage servers in the Stratos Datacenter environment.

---

## Overview

This directory documents hands-on Git version control work performed within the **Nautilus Application Development** infrastructure of the **Stratos Datacenter (Stratos DC)**. Each task reflects a real operational scenario: resolving push failures under diverged history, managing root-owned repositories in shared server environments, automating release tagging via native Git hooks, and enforcing peer-reviewed merge workflows through a self-hosted Gitea instance.

The work is structured around a jump-host topology where all operations originate from `thor@jumphost` and target `natasha@ststor01` or the Gitea web interface, mirroring common enterprise network segmentation patterns.

---

## Directory Structure

```
git-version-control/
├── distributed-scm-workflows/          # Rebase, conflict resolution, and push to Gitea remote
├── enterprise-git-provisioning/        # Bare repository init and Git server setup on storage node
├── git-branch-management/              # Feature branch creation with ownership and permission handling
├── git-commit-reversion-workflow/      # Non-destructive HEAD revert with custom commit message
├── git-release-automation/             # post-update hook for automated date-stamped release tagging
├── git-remote-origin-provisioning/     # Multi-remote configuration and privileged commit and push
├── git-repository-forking-workflow/    # Gitea UI fork provisioning for contributor onboarding
├── git-reset-commit-history/           # Hard reset and force push to clean polluted commit history
├── git-server-provisioning/            # Git installation and bare repo initialization via yum on CentOS
├── git-stash-recovery-and-origin-sync/ # Stash recovery, commit, and push on permission-restricted repo
├── git-workflow-automation/            # Branch creation, file commit, merge, and dual-branch push
├── gitea-pull-request-review-workflow/ # Full PR lifecycle: creation, reviewer assignment, approval, merge
├── linear-rebase-workflow/             # Feature branch rebase onto master with force push
└── selective-commit-propagation/       # Cross-branch cherry-pick targeting a single commit by hash
```

---

## Project Summaries

### [Distributed SCM Workflows](./distributed-scm-workflows)

**Quick Summary:** Resolved a failed push to a self-hosted Gitea remote caused by diverged history and an `add/add` merge conflict during rebase. Fixed a typo in a tracked file and completed the full push cycle.

- **Purpose:** Push a corrected file to `sarah/story-blog` on Gitea after remote history had advanced past the local branch tip.
- **Approach:** Escalated from `natasha` to `max` via `sudo su -`. Corrected content with `sed -i`, staged and committed, then pulled with `--rebase`. Resolved the resulting `add/add` conflict using a heredoc overwrite before completing the rebase and pushing.
- **Outcome:** Clean linear history pushed to `origin/master`. Verified in Gitea UI. Git automatically dropped a redundant typo-fix commit whose content already existed upstream, confirming correct rebase behavior.

---

### [Enterprise Git Provisioning](./enterprise-git-provisioning)

**Quick Summary:** Cloned a pre-existing bare repository (`/opt/apps.git`) into a designated workspace on a storage server, verified remote configuration, and validated clone integrity.

- **Purpose:** Provision a usable working directory for the development team from a bare repository with no prior working clone.
- **Approach:** Confirmed bare repo structure and target directory existence before executing `git clone /opt/apps.git` as `natasha`. Verified `origin` remote and branch state post-clone.
- **Outcome:** Working clone at `/usr/src/kodekloudrepos/apps` with `origin` correctly pointing to the bare repository. Zero directory modifications outside the clone target.

---

### [Git Branch Management](./git-branch-management)

**Quick Summary:** Created a feature branch (`xfusioncorp_apps`) from `master` on a root-owned repository, handling both a CVE-2022-24765 safe directory block and a write permission barrier.

- **Purpose:** Provision an isolated feature branch without modifying any existing files or code.
- **Approach:** Registered the safe directory exception as `natasha`, then escalated to `root` via `sudo su -` to gain write access to the `.git` directory. Re-registered the safe directory for the root context before branching.
- **Outcome:** `xfusioncorp_apps` created from `master` with a clean working tree. Key lesson: `safe.directory` is scoped per Unix user and must be set independently for each account that interacts with the repository.

---

### [Git Commit Reversion Workflow](./git-commit-reversion-workflow)

**Quick Summary:** Rolled back an erroneous HEAD commit on a shared repository using `git revert` with a custom commit message, preserving full history.

- **Purpose:** Revert the latest commit (`add data.txt file`) in `/usr/src/kodekloudrepos/games` with the exact message `revert games`, without destroying history.
- **Approach:** Used `git revert HEAD --no-commit` to stage the inverse diff, then committed with the required message. All write operations executed via `sudo` due to root ownership of `.git`. Safe directory registered for both `natasha` and root contexts.
- **Outcome:** New revert commit at HEAD. Full history preserved with the original bad commit retained. `git revert` chosen over `git reset` to avoid rewriting shared branch history.

---

### [Git Release Automation](./git-release-automation)

**Quick Summary:** Implemented a `post-update` Git hook on a working clone to automatically create a `release-YYYY-MM-DD` tag on every push to `master`, eliminating manual tagging overhead.

- **Purpose:** Enforce consistent release naming on the `ecommerce` repository without client-side configuration changes.
- **Approach:** Merged `feature` into `master`, then wrote a `post-update` hook using a heredoc and marked it executable. Manually triggered the hook to validate output before pushing. Tags pushed to the bare remote separately via `git push origin --tags`.
- **Outcome:** Hook fires on each push to `master`, generating a date-stamped tag. Verified via `git tag` and `git log --oneline`. No privilege escalation required; `natasha` owned the repository.

---

### [Git Remote Origin Provisioning](./git-remote-origin-provisioning)

**Quick Summary:** Added a second named remote (`dev_demo`) to an existing clone, committed a file from `/tmp`, and pushed `master` to the new remote, overcoming two sequential permission barriers.

- **Purpose:** Register `/opt/xfusioncorp_demo.git` as `dev_demo` inside `/usr/src/kodekloudrepos/demo` and deliver `index.html` to it.
- **Approach:** Resolved the CVE-2022-24765 ownership block for `natasha`, then confirmed that `git remote add` required `sudo` due to root-owned `.git/config`. All subsequent operations (copy, stage, commit, push) prefixed with `sudo`.
- **Outcome:** Both `origin` and `dev_demo` remotes confirmed via `git remote -v`. `master` pushed successfully to the new remote with all objects transferred.

---

### [Git Repository Forking Workflow](./git-repository-forking-workflow)

**Quick Summary:** Forked `sarah/story-blog` under the `jon` account via Gitea Web UI to onboard a new contributor with an isolated development environment.

- **Purpose:** Provision a personal fork for `jon` to enable PR-based contributions without direct repository access.
- **Approach:** Authenticated as `jon` in Gitea, navigated to the source repository, and completed the fork via the UI fork dialog. Verified fork indicators: correct owner, upstream reference, and commit history parity.
- **Outcome:** `jon/story-blog` created with all 5 upstream commits and PR workflow enabled. `jon` can now submit changes via Pull Request to `sarah/story-blog`.

---

### [Git Reset Commit History](./git-reset-commit-history)

**Quick Summary:** Removed 10 test commits from a shared repository using `git reset --hard` and synchronized the rewritten history to the remote via force push.

- **Purpose:** Clean a commit history polluted with test commits, leaving only `initial commit` and `add data.txt file`.
- **Approach:** Identified target commit hash via `git log --oneline`. Escalated to root to overcome `.git` write permissions. Registered safe directory for root context, executed hard reset, then force pushed with `git push -f origin master`.
- **Outcome:** Remote `origin/master` reduced to exactly 2 commits. Force push confirmed via `(forced update)` label in push output. All 10 test commits permanently removed from both local and remote history.

---

### [Git Server Provisioning](./git-server-provisioning)

**Quick Summary:** Installed Git on a CentOS Stream 9 storage node via `yum` and initialized a bare repository at `/opt/demo.git` to serve as a centralized team remote.

- **Purpose:** Stand up a Git hosting node on `ststor01` from scratch, providing a push/pull endpoint for distributed teams.
- **Approach:** Installed Git 2.52.0 via `sudo yum install -y git`. Initialized the bare repository with `sudo git init --bare /opt/demo.git`. Validated internal structure (`HEAD`, `objects/`, `refs/`, `config`, `hooks/`).
- **Outcome:** Fully operational bare repository ready for `git clone`, `push`, and `fetch` over SSH. Hooks directory pre-provisioned for future server-side automation.

---

### [Git Stash Recovery and Origin Sync](./git-stash-recovery-and-origin-sync)

**Quick Summary:** Recovered a specific stash entry (`stash@{1}`) from a restricted repository, committed the restored content, and pushed to `origin/master`.

- **Purpose:** Restore `welcome.txt` from a stash on `ststor01` and deliver it to the remote without data loss.
- **Approach:** Applied `safe.directory` exception to unblock `git stash list` as `natasha`. Used `sudo git stash apply stash@{1}` to overcome `.git/index` write restriction. Staged, committed, and pushed with `sudo`.
- **Outcome:** `welcome.txt` committed and pushed. Remote `origin/master` advanced by one commit. Stash index confirmed via `git stash list` before application.

---

### [Git Workflow Automation](./git-workflow-automation)

**Quick Summary:** Created a feature branch (`nautilus`) from `master`, committed a file, merged back, and pushed both branches to the remote, all within a root-owned repository.

- **Purpose:** Execute a standard feature branch lifecycle on `/usr/src/kodekloudrepos/blog` with correct branch hygiene.
- **Approach:** Resolved ownership block for `natasha` then used `sudo` for all write operations. Created `nautilus` branch, copied `/tmp/index.html`, committed, merged via fast-forward into `master`, and pushed both branches independently.
- **Outcome:** `git log --oneline --all --graph` confirmed all four refs (`master`, `nautilus`, `origin/master`, `origin/nautilus`) aligned on the same commit, validating a clean fast-forward merge with no divergence.

---

### [Gitea Pull Request Review Workflow](./gitea-pull-request-review-workflow)

**Quick Summary:** Managed a full PR lifecycle on a self-hosted Gitea instance: created a PR from a feature branch, assigned a reviewer, approved as the reviewer, and merged into `master`.

- **Purpose:** Enforce a code review gate on `sarah/story-blog` using Gitea's PR workflow before merging `story/fox-and-grapes` into `master`.
- **Approach:** Authenticated as `max` to create and configure the PR with `tom` as reviewer. Switched sessions to `tom` to review the diff, approve, and execute the merge commit via the Gitea UI.
- **Outcome:** PR merged with a full, immutable audit trail: creation, reviewer assignment, approval, and merge events all recorded in Gitea. Demonstrates separation of duties between author and approver.

---

### [Linear Rebase Workflow](./linear-rebase-workflow)

**Quick Summary:** Rebased a diverged `feature` branch onto `master` to produce a linear commit history, then force pushed the result to the bare remote.

- **Purpose:** Synchronize `feature` with new `master` commits without introducing a merge commit, on `/usr/src/kodekloudrepos/news`.
- **Approach:** Fixed directory ownership with `sudo chown -R`, configured safe directories and Git identity, then ran `git rebase master` on the feature branch. Used `--force-with-lease` on the push to guard against concurrent remote updates.
- **Outcome:** `feature` replayed cleanly on top of `master` with a new commit SHA. Remote updated via forced push. `--force-with-lease` preferred over `--force` to prevent accidental data loss from concurrent pushes.

---

### [Selective Commit Propagation](./selective-commit-propagation)

**Quick Summary:** Cherry-picked a single specific commit (`Update info.txt`) from a `feature` branch onto `master`, leaving unrelated feature work isolated, then pushed to the bare remote.

- **Purpose:** Propagate only the `info.txt` change to `master` without pulling in the adjacent `welcome.txt` commit from the same branch.
- **Approach:** Identified the target hash (`d1a149c`) via `git log --oneline` on the feature branch. Switched to `master` with `sudo`, executed `git cherry-pick d1a149c`, and pushed. Verified that only the intended commit appeared in `master` history.
- **Outcome:** New commit `b8d9e80` on `master` with the correct diff. Remote fast-forwarded by one commit. Feature branch untouched.

---

## Technologies and Tools

| Category | Stack |
|---|---|
| Version Control | Git 2.x, Gitea 1.25.3 |
| Operating System | CentOS Stream 9, Linux (Ubuntu 24) |
| Access Pattern | SSH jump host topology |
| Package Management | yum / dnf |
| Scripting | Bash (Git hooks, heredocs, sed) |
| Security | CVE-2022-24765 safe directory handling, SSH host key verification |
| Collaboration | Gitea Pull Requests, fork workflows, reviewer assignment |

---

## Key Skills Demonstrated

- **Rebase and conflict resolution** under diverged history, including `add/add` conflict handling and automatic commit deduplication
- **Root-owned repository operations** using layered privilege escalation (`sudo`, `sudo su -`) and per-user `safe.directory` configuration
- **Bare repository architecture** including provisioning, cloning, hook deployment, and multi-remote configuration
- **History rewriting** via `git reset --hard` with force push, and `git revert` for non-destructive rollback on shared branches
- **Selective commit propagation** via cherry-pick targeting specific hashes across branches
- **Git hook automation** using `post-update` scripts for release tagging without client-side dependencies
- **Gitea administration** covering fork provisioning, PR creation, reviewer assignment, and merge governance
- **Linear history maintenance** via feature branch rebase with `--force-with-lease` push safety

---

## Navigation Guide

Each subdirectory contains a `README.md` with the full task walkthrough, including:

- Environment and infrastructure tables
- Annotated command sequences
- Error index with root cause and resolution for each failure encountered
- Terminal screenshots with captions
- Verification checklists confirming end state

Start with [`distributed-scm-workflows`](./distributed-scm-workflows) for a representative example of multi-phase conflict resolution, or [`git-release-automation`](./git-release-automation) for a practical hook implementation walkthrough.

---
