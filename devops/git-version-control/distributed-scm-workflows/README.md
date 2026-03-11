# Resolving Remote Push Failures via Rebase and Conflict Resolution on a Self-Hosted Gitea Instance

![Git](https://img.shields.io/badge/Git-2.x-F05032?style=flat-square&logo=git&logoColor=white)
![Gitea](https://img.shields.io/badge/Gitea-1.25.3-609926?style=flat-square&logo=gitea&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-SSH-FCC624?style=flat-square&logo=linux&logoColor=black)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=flat-square)

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Problem Statement](#problem-statement)
- [Root Cause Analysis](#root-cause-analysis)
- [Resolution Workflow](#resolution-workflow)
  - [Phase 1: Remote Server Access](#phase-1-remote-server-access)
  - [Phase 2: Repository Inspection](#phase-2-repository-inspection)
  - [Phase 3: File Remediation](#phase-3-file-remediation)
  - [Phase 4: Commit and Push Attempt](#phase-4-commit-and-push-attempt)
  - [Phase 5: Remote URL Correction and Rebase](#phase-5-remote-url-correction-and-rebase)
  - [Phase 6: Conflict Resolution](#phase-6-conflict-resolution)
  - [Phase 7: Rebase Completion and Successful Push](#phase-7-rebase-completion-and-successful-push)
  - [Phase 8: Gitea UI Verification](#phase-8-gitea-ui-verification)
- [Screenshots](#screenshots)
- [Key Concepts](#key-concepts)
- [Lessons Learned](#lessons-learned)
- [Commands Quick Reference](#commands-quick-reference)

---

## Overview

This document details the end-to-end resolution of a failed Git push operation on a self-hosted Gitea instance within a multi-server infrastructure. The task required accessing a remote storage server via SSH, correcting corrupted content in a tracked file, and successfully pushing the fix to a shared upstream repository owned by another user. The process encountered two compounding failures: an initial push rejection due to diverged history, and a subsequent merge conflict during rebase. Both were resolved systematically.

---

## Environment

| Component | Value |
|---|---|
| Jump Host | `thor@jumphost` |
| Storage Server | `ststor01.stratos.xfusioncorp.com` |
| Storage Server User | `natasha` / `max` |
| Git Remote | `http://gitea:3000/sarah/story-blog.git` |
| Gitea Version | 1.25.3 |
| Target Branch | `master` |
| Target File | `story-index.txt` |
| Repository Path | `/home/max/story-blog` |

---

## Problem Statement

Two collaborators, Sarah and Max, share a `story-blog` repository hosted on an internal Gitea server. Max had committed local changes but was unable to push them to the remote origin. The issues identified were:

1. **Content defect** -- `story-index.txt` contained a typographical error: `The Lion and the Mooose` instead of `The Lion and the Mouse`
2. **Push rejection** -- the remote `origin/master` had advanced beyond Max's local branch tip, causing a non-fast-forward rejection
3. **Merge conflict** -- rebasing against the remote introduced an `add/add` conflict in `story-index.txt` that required manual resolution before the push could complete

---

## Root Cause Analysis

| Failure | Cause | Classification |
|---|---|---|
| `Permission denied` on `cd` | `natasha` lacked read access to `/home/max` | Access control |
| `[rejected] master -> master (fetch first)` | Remote had 1 new commit not present locally | Diverged history |
| `CONFLICT (add/add): Merge conflict in story-index.txt` | Both remote and local independently modified the same file | Concurrent edit collision |
| Second push rejection `(non-fast-forward)` | Push was attempted while rebase was still in a paused/conflicted state | Incomplete rebase |

---

## Resolution Workflow

### Phase 1: Remote Server Access

SSH into the storage server from the jump host using `natasha`'s credentials.

```bash
ssh natasha@ststor01.stratos.xfusioncorp.com
```

**Problem encountered:** `natasha` does not have access to `/home/max/story-blog`.

```
-bash: cd: /home/max/story-blog: Permission denied
```

**Resolution:** Escalate to the `max` user via `sudo`.

```bash
sudo su - max
whoami
# Output: max
```

***Screenshot: Terminal showing successful SSH login as natasha and sudo escalation to max***
<img width="1034" height="467" alt="image" src="https://github.com/user-attachments/assets/e739e699-874d-454a-aafc-fc594f5203ad" />

---

### Phase 2: Repository Inspection

Navigate to the repository and assess its current state before making any changes.

```bash
cd /home/max/story-blog
git status
```

**Output observed:**

```
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)

nothing to commit, working tree clean
```

**Interpretation:** Max had an existing unpushed commit. No uncommitted changes were present. The file required inspection before any action.

```bash
cat story-index.txt
```

***Screenshot: Terminal output of git status and cat story-index.txt showing the Mooose typo***
<img width="1029" height="672" alt="image" src="https://github.com/user-attachments/assets/05b0d412-6713-4751-9a51-3042088c1abf" />

---

### Phase 3: File Remediation

The file contained `The Lion and the Mooose` on line 1. The correct value is `The Lion and the Mouse`. Use `sed` for a precise, non-interactive in-place substitution.

```bash
sed -i 's/The Lion and the Mooose/The Lion and the Mouse/g' story-index.txt
```

Verify the correction immediately after:

```bash
cat story-index.txt
```

**Expected output:**

```
1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
```

***Screenshot: Terminal output confirming the corrected story-index.txt with all 4 titles***
<img width="1034" height="685" alt="image" src="https://github.com/user-attachments/assets/65dda5ea-c09d-45df-8253-e7597e9f7532" />

---

### Phase 4: Commit and Push Attempt

Stage the corrected file and commit with a descriptive message.

```bash
git add story-index.txt
git commit -m "Fix typo in story-index.txt: Mooose to Mouse"
```

Attempt to push to the remote origin:

```bash
git push origin master
```

**Failure encountered:**

```
! [rejected]        master -> master (fetch first)
error: failed to push some refs to 'http://gitea:3000/sarah/story-blog.git'
hint: Updates were rejected because the remote contains work that you do not have locally.
```

**Cause:** The remote `sarah/story-blog` had received at least one new commit after Max's last fetch. The local branch tip was behind the remote, making a direct push impossible.

***Screenshot Placeholder -- 04: Terminal showing the rejected push error with fetch first hint***

---

### Phase 5: Remote URL Correction and Rebase

The original remote URL did not include credentials, causing an interactive password prompt. Update the remote URL to embed authentication, then pull with rebase to replay local commits on top of the updated remote.

```bash
git remote set-url origin http://max:Max_pass123@gitea:3000/sarah/story-blog.git
git pull --rebase origin master
```

**Failure encountered during rebase:**

```
CONFLICT (add/add): Merge conflict in story-index.txt
error: could not apply f173ef4... Added the fox and grapes story
```

**Cause:** Both the remote commit and Max's local commit had independently modified `story-index.txt`, resulting in an `add/add` conflict that Git could not resolve automatically.

***Screenshot Placeholder -- 05: Terminal showing the rebase conflict error output***

---

### Phase 6: Conflict Resolution

Inspect the conflicted file to understand both versions before resolving.

```bash
cat story-index.txt
```

**Conflict structure observed:**

```
<<<<<<< HEAD
1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
=======
1. The Lion and the Mooose
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
>>>>>>> f173ef4 (Added the fox and grapes story)
```

**Analysis:**

| Version | Contains | Missing |
|---|---|---|
| HEAD (remote) | Correct spelling "Mouse" | Title 4 ("The Donkey and the Dog") |
| f173ef4 (local) | All 4 titles | Correct spelling (has "Mooose") |

**Resolution:** Write the definitive merged version directly using a heredoc, combining correct spelling from HEAD with the complete title list from the local commit.

```bash
cat > story-index.txt << 'EOF'
1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
EOF
```

Verify the resolved file:

```bash
cat story-index.txt
```

***Screenshot Placeholder -- 06: Terminal showing the clean resolved story-index.txt with 4 correct titles***

Mark the conflict as resolved and continue the rebase:

```bash
git add story-index.txt
git rebase --continue
```

***Screenshot Placeholder -- 07: Terminal showing git rebase continue output and "Successfully rebased and updated refs/heads/master"***

---

### Phase 7: Rebase Completion and Successful Push

With the rebase completed, push to the remote origin.

```bash
git push origin master
```

**Successful output:**

```
Writing objects: 100% (4/4), 871 bytes | 871.00 KiB/s, done.
To http://gitea:3000/sarah/story-blog.git
   51dd941..9f6819e  master -> master
```

**Note:** Git automatically dropped the redundant typo-fix commit (`7ec370b`) during rebase because its content was already present upstream, reporting:

```
dropping 7ec370b...Fix typo in story-index.txt: Mooose to Mouse -- patch contents already upstream
```

This is expected and correct behavior.

***Screenshot Placeholder -- 08: Terminal showing the successful git push output with master to master confirmation***

---

### Phase 8: Gitea UI Verification

Log into the Gitea web interface to visually confirm the pushed state of the repository.

**Access:** Click the Gitea UI button in the lab interface top bar.

**Credentials:**
```
Username: sarah
Password: Sarah_pass123
```

**Verification steps:**

1. Navigate to `sarah / story-blog`
2. Open `story-index.txt`
3. Confirm all 4 story titles are present
4. Confirm line 1 reads `The Lion and the Mouse`
5. Confirm the latest commit hash matches `9f6819e480`

***Screenshot Placeholder -- 09: Gitea UI showing sarah/story-blog repository file tree***

***Screenshot Placeholder -- 10: Gitea UI showing story-index.txt with all 4 correct titles and commit 9f6819e480 by Max***

---

## Screenshots

| # | Description |
|---|---|
| 01 | SSH login and sudo escalation to max |
| 02 | git status and cat story-index.txt showing original typo |
| 03 | Corrected story-index.txt after sed substitution |
| 04 | Rejected push error (fetch first) |
| 05 | Rebase conflict error output |
| 06 | Resolved story-index.txt via heredoc |
| 07 | git rebase continue and success message |
| 08 | Successful git push output |
| 09 | Gitea UI repository file tree |
| 10 | Gitea UI story-index.txt final verified state |

---

## Key Concepts

**Non-fast-forward rejection**
Occurs when the remote branch has commits that do not exist in the local branch. Git refuses to overwrite remote history. Resolution requires fetching and integrating remote changes before pushing.

**`git pull --rebase` vs `git pull --merge`**
Rebase replays local commits on top of the fetched remote tip, producing a linear history. This is preferred over merge in shared repositories to avoid unnecessary merge commits.

**`add/add` conflict**
A specific conflict type that arises when two branches independently add content to the same file. Git cannot determine which version takes precedence and requires manual resolution.

**Heredoc for conflict resolution**
Using `cat > file << 'EOF'` overwrites a conflicted file with a known-good value in a single atomic operation, eliminating the risk of accidentally leaving conflict markers in the file.

**Automatic commit dropping during rebase**
When a local commit's diff is already present in the upstream history (because the same change was pushed by another path), Git silently drops the duplicate commit during rebase. This is not an error.

---

## Lessons Learned

* Always run `git status` and `cat` the target file before making any changes to establish a verified baseline
* Embed credentials in the remote URL or configure a credential helper before attempting push operations to avoid interactive prompts in automated or constrained environments
* Never attempt `git push` while a rebase is in a paused/conflicted state -- resolve all conflicts and run `git rebase --continue` first
* When resolving conflicts, analyze both sides of the conflict marker before writing the final version to ensure no content is lost from either branch
* Verify the final state in the remote UI after every push, not just in the terminal

---

## Commands Quick Reference

```bash
# Access remote server
ssh natasha@ststor01.stratos.xfusioncorp.com
sudo su - max

# Inspect repository state
cd /home/max/story-blog
git status
cat story-index.txt

# Fix file content
sed -i 's/The Lion and the Mooose/The Lion and the Mouse/g' story-index.txt

# Stage and commit
git add story-index.txt
git commit -m "Fix typo in story-index.txt: Mooose to Mouse"

# Update remote URL with credentials
git remote set-url origin http://max:Max_pass123@gitea:3000/sarah/story-blog.git

# Pull with rebase
git pull --rebase origin master

# Resolve conflict via heredoc
cat > story-index.txt << 'EOF'
1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
EOF

# Complete rebase
git add story-index.txt
git rebase --continue

# Push to remote
git push origin master
```

---


<img width="1031" height="438" alt="image" src="https://github.com/user-attachments/assets/3f61d3eb-3e35-4b9b-ad09-75339f2ee2de" />
<img width="1034" height="359" alt="image" src="https://github.com/user-attachments/assets/033ad467-9a08-4d7c-b340-f3fb7cf1ec3d" />

<img width="1031" height="591" alt="image" src="https://github.com/user-attachments/assets/e5e412fa-34ea-4497-bd39-13125fad293e" />


<img width="1030" height="742" alt="image" src="https://github.com/user-attachments/assets/b42148d7-ffb5-499e-be00-188334f1e793" />
<img width="1034" height="763" alt="image" src="https://github.com/user-attachments/assets/18629bdc-cb11-44d7-ad5d-f4ed940a5afa" />
<img width="1033" height="859" alt="image" src="https://github.com/user-attachments/assets/673546da-45b5-4d87-aed8-4a72345ea93b" />
<img width="1043" height="364" alt="image" src="https://github.com/user-attachments/assets/7cdb6132-7907-4065-895d-0cf8aef739e8" />
<img width="1033" height="276" alt="image" src="https://github.com/user-attachments/assets/ff5ed140-9058-4d16-b134-acc45a2269ab" />
<img width="1028" height="461" alt="image" src="https://github.com/user-attachments/assets/cebe26f8-97fa-4069-a70e-b6f874478bcd" />
<img width="1033" height="818" alt="image" src="https://github.com/user-attachments/assets/6948ed0b-f2ec-41ec-9641-ef02c57703e5" />
<img width="1033" height="818" alt="image" src="https://github.com/user-attachments/assets/1f830026-7f71-4f57-8ee8-5c808323d55e" />
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/296074e2-74a0-4e4c-a6a4-6ed1e4850950" />
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/6992ba84-8540-4a5a-bbe3-0f046512c9d7" />

