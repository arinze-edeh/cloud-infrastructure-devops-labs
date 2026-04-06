# Passwordless SSH Authentication: Jump Host to Application Servers

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Objectives](#objectives)
- [Prerequisites](#prerequisites)
- [High-Level Logic](#high-level-logic)
- [Implementation](#implementation)
  - [Step 1: Authenticate to Jump Host](#step-1-authenticate-to-jump-host)
  - [Step 2: Generate RSA SSH Key Pair](#step-2-generate-rsa-ssh-key-pair)
  - [Step 3: Verify SSH Key Files](#step-3-verify-ssh-key-files)
  - [Step 4: Distribute Public Key to App Server 1 (stapp01)](#step-4-distribute-public-key-to-app-server-1-stapp01)
  - [Step 5: Distribute Public Key to App Server 2 (stapp02)](#step-5-distribute-public-key-to-app-server-2-stapp02)
  - [Step 6: Distribute Public Key to App Server 3 (stapp03)](#step-6-distribute-public-key-to-app-server-3-stapp03)
  - [Step 7: Validate Passwordless SSH Access](#step-7-validate-passwordless-ssh-access)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)
- [Tags](#tags)

---

## Overview

This document details the configuration of passwordless SSH key-based authentication from a centralized jump host to all Nautilus application servers in the xFusionCorp infrastructure. The setup replaces interactive password-based logins with cryptographic key authentication, enabling automation scripts and configuration management tools to operate securely across the server fleet without manual credential entry.

---

## Problem Statement

The xFusionCorp system administration team required a mechanism for the jump host (`jump_host.stratos.xfusioncorp.com`) to connect to all three Nautilus application servers without prompting for passwords. Password-based SSH in automated workflows introduces operational risk: scripts fail on unattended prompts, credentials risk exposure in automation logs, and manual intervention breaks CI/CD pipelines and scheduled maintenance tasks.

**Solution:** Establish RSA key-based authentication from the jump host user (`thor`) to each application server's sudo user account. This provides cryptographic trust without shared secrets, aligns with security hardening best practices, and enables frictionless automation.

---

## Architecture

| Component | Hostname | IP Address | User |
|---|---|---|---|
| Jump Host | `jump_host.stratos.xfusioncorp.com` | -- | `thor` |
| App Server 1 | `stapp01.stratos.xfusioncorp.com` | `172.16.238.10` | `tony` |
| App Server 2 | `stapp02.stratos.xfusioncorp.com` | `172.16.238.11` | `steve` |
| App Server 3 | `stapp03.stratos.xfusioncorp.com` | `172.16.238.12` | `banner` |

**Authentication Flow:**
```
[thor@jumphost] --> RSA Public Key --> [tony@stapp01]
                                  --> [steve@stapp02]
                                  --> [banner@stapp03]
```

---

## Objectives

- Generate an RSA SSH key pair on the jump host
- Distribute the public key to all three application servers using `ssh-copy-id`
- Eliminate password prompts for subsequent SSH sessions
- Validate passwordless access by executing remote commands from the jump host

---

## Prerequisites

- SSH client available on the jump host
- Network connectivity from jump host to all application servers on port 22
- Valid credentials for each application server user (required once for initial key distribution)
- `ssh-copy-id` utility available on the jump host

---

## High-Level Logic

```
LOGIN to jump host as thor
IF SSH key pair does not exist in ~/.ssh/:
    GENERATE RSA key pair with ssh-keygen
    VERIFY key files created with correct permissions

FOR each application server (stapp01, stapp02, stapp03):
    EXECUTE ssh-copy-id to push public key
    ACCEPT host fingerprint on first connection
    AUTHENTICATE once using account password

FOR each application server:
    SSH into server WITHOUT password
    RUN hostname command to confirm identity and access
    CONFIRM output matches expected server FQDN
```

---

## Implementation

### Step 1: Authenticate to Jump Host

Connect to the jump host as user `thor`. This is the origin node from which all subsequent operations are performed.

```bash
ssh thor@jump_host.stratos.xfusioncorp.com
```

> **Operational Note:** All key generation and distribution must be executed from this host. Do not generate keys on individual app servers.

**Screenshot: Successful login to jump host**

<img width="1030" height="607" alt="Jump host login via SSH as user thor" src="https://github.com/user-attachments/assets/1c75275f-e725-4306-8a5a-fb573260cf23" />

---

### Step 2: Generate RSA SSH Key Pair

Generate a 3072-bit RSA key pair on the jump host. Accept the default file location and leave the passphrase empty to allow non-interactive authentication for automation use cases.

```bash
ssh-keygen -t rsa
```

**Prompts and expected responses:**

| Prompt | Action |
|---|---|
| `Enter file in which to save the key` | Press Enter (accept default: `/home/thor/.ssh/id_rsa`) |
| `Enter passphrase` | Press Enter (no passphrase for automation) |
| `Enter same passphrase again` | Press Enter |

> **Security Consideration:** Omitting a passphrase is appropriate when the key is used exclusively for automation from a controlled, access-restricted jump host. For human-interactive use cases, a passphrase with `ssh-agent` is recommended.

**Screenshot: RSA key pair generation with fingerprint and randomart confirmation**

<img width="919" height="662" alt="ssh-keygen RSA key pair generation output showing fingerprint SHA256 and randomart image" src="https://github.com/user-attachments/assets/18f2abe1-aaef-49e0-b596-6f8d2003ea1a" />

---

### Step 3: Verify SSH Key Files

Confirm both the private key and public key files exist with correct permissions before distributing.

```bash
ls -l ~/.ssh/id_rsa*
```

**Expected output:**

```
-rw------- 1 thor thor 2635 Feb 13 17:49 /home/thor/.ssh/id_rsa
-rw-r--r-- 1 thor thor  591 Feb 13 17:49 /home/thor/.ssh/id_rsa.pub
```

> **Permission Validation:** The private key (`id_rsa`) must be `600` (owner read/write only). The public key (`id_rsa.pub`) is `644`. SSH will refuse to use keys with overly permissive file modes.

**Screenshot: Verified SSH key file permissions for private and public keys**

<img width="919" height="662" alt="ls -l output confirming id_rsa at 600 and id_rsa.pub at 644 permissions" src="https://github.com/user-attachments/assets/18f2abe1-aaef-49e0-b596-6f8d2003ea1a" />

---

### Step 4: Distribute Public Key to App Server 1 (stapp01)

Copy the jump host's public key to the `authorized_keys` file of user `tony` on `stapp01`. This is a one-time operation requiring password authentication.

```bash
ssh-copy-id tony@172.16.238.10
```

**During execution:**
- Accept the host fingerprint prompt by typing `yes`
- Enter `tony`'s account password when prompted

**Expected confirmation:**

```
Number of key(s) added: 1

Now try logging into the machine, with: "ssh 'tony@172.16.238.10'"
```

**Screenshot: Public key successfully distributed to stapp01 via ssh-copy-id**

<img width="1030" height="860" alt="ssh-copy-id output for tony@172.16.238.10 showing key added and host fingerprint acceptance" src="https://github.com/user-attachments/assets/486fe75b-b3a5-4b77-bd54-6b559757a4ec" />

---

### Step 5: Distribute Public Key to App Server 2 (stapp02)

Copy the same public key to user `steve` on `stapp02`.

```bash
ssh-copy-id steve@172.16.238.11
```

- Accept the host fingerprint prompt by typing `yes`
- Enter `steve`'s account password when prompted

**Expected confirmation:**

```
Number of key(s) added: 1

Now try logging into the machine, with: "ssh 'steve@172.16.238.11'"
```

**Screenshot: Public key successfully distributed to stapp02 via ssh-copy-id**

<img width="1030" height="855" alt="ssh-copy-id output for steve@172.16.238.11 showing key added and host fingerprint acceptance" src="https://github.com/user-attachments/assets/c2a3ce55-ddbd-44bd-b86e-abec1599d643" />

---

### Step 6: Distribute Public Key to App Server 3 (stapp03)

Copy the same public key to user `banner` on `stapp03`.

```bash
ssh-copy-id banner@172.16.238.12
```

- Accept the host fingerprint prompt by typing `yes`
- Enter `banner`'s account password when prompted

**Expected confirmation:**

```
Number of key(s) added: 1

Now try logging into the machine, with: "ssh 'banner@172.16.238.12'"
```

**Screenshot: Public key successfully distributed to stapp03 via ssh-copy-id**

<img width="1029" height="865" alt="ssh-copy-id output for banner@172.16.238.12 showing key added and host fingerprint acceptance" src="https://github.com/user-attachments/assets/abaed342-6e98-4d5d-b897-8c4cb1dbd8d9" />

---

### Step 7: Validate Passwordless SSH Access

Verify that the jump host can connect to each application server without a password prompt. Remote `hostname` execution confirms both connectivity and correct server identity.

```bash
ssh tony@172.16.238.10 "hostname"
ssh steve@172.16.238.11 "hostname"
ssh banner@172.16.238.12 "hostname"
```

**Expected output:**

```
stapp01.stratos.xfusioncorp.com
stapp02.stratos.xfusioncorp.com
stapp03.stratos.xfusioncorp.com
```

> **Validation Logic:** A successful non-interactive command execution with the correct FQDN response confirms that key-based authentication is active, the `authorized_keys` entry is valid, and SSH daemon on each server is accepting key authentication.

**Screenshot: Passwordless SSH validation -- hostname responses from all three app servers**

<img width="1025" height="872" alt="ssh non-interactive hostname commands returning stapp01, stapp02, stapp03 FQDNs confirming passwordless access" src="https://github.com/user-attachments/assets/3f756692-115f-468e-a2fb-0d9ed14706c8" />

---

## Key Decisions

- **RSA over Ed25519:** RSA was selected for broad compatibility with older OpenSSH versions that may be present on infrastructure nodes. Ed25519 is preferred for modern environments but was not required here.
- **No passphrase on the key:** A passphrase-free key is intentional for automation contexts where `ssh-agent` integration is not available. The jump host itself acts as the security boundary.
- **Single key pair for all servers:** One key pair distributed to all application servers simplifies key lifecycle management and rotation. When key rotation is needed, only one new key must be generated and redistributed.
- **`ssh-copy-id` over manual copy:** `ssh-copy-id` correctly handles `authorized_keys` file creation, permission setting (`600`), and deduplication of existing keys. Manual copying risks permission errors that silently break key auth.
- **IP-based `ssh-copy-id` targets:** Direct IP addresses were used during distribution to avoid DNS resolution dependency during the initial setup phase.

---

## Errors and Resolutions

| Scenario | Symptom | Resolution |
|---|---|---|
| Host key verification failure | `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` | Remove stale entry with `ssh-keygen -R <ip>`, then reconnect |
| Permission too open on private key | `Permissions 0644 for 'id_rsa' are too open` | Run `chmod 600 ~/.ssh/id_rsa` |
| `authorized_keys` missing on target | Key distributed but SSH still prompts for password | Verify `~/.ssh` is `700` and `authorized_keys` is `600` on the target server |
| `ssh-copy-id` not found | Command not found error | Install with `apt install openssh-client` or use manual key copy via `ssh` pipe |
| Host fingerprint prompt blocks automation | Script halts waiting for `yes/no` input | Pre-populate `known_hosts` or use `-o StrictHostKeyChecking=accept-new` |

---

## Lessons Learned

- **One-time password exposure is expected and acceptable.** The initial `ssh-copy-id` requires a password. This is the only moment credentials transit the network. All subsequent connections are cryptographic.
- **Host fingerprint acceptance is a security checkpoint, not an obstacle.** Always verify the fingerprint against the target server's known value before accepting. Blindly accepting fingerprints risks man-in-the-middle exposure.
- **Key distribution is idempotent.** Running `ssh-copy-id` against a server that already has the key installed is safe. It detects existing keys and skips duplicate installation.
- **The jump host is a trust anchor.** Any compromise of the jump host compromises all servers it has key-based access to. Jump host hardening (MFA, restricted login, audit logging) is essential in production.
- **Validate immediately after distribution.** Always run a non-interactive test command (`hostname`, `id`, `uptime`) immediately after key distribution to confirm the configuration before relying on it in automation pipelines.

---

## Outcome

- RSA key pair generated on jump host with correct file permissions
- Public key successfully deployed to all three Nautilus application servers
- Passwordless SSH authentication confirmed for `tony@stapp01`, `steve@stapp02`, and `banner@stapp03`
- Remote hostname commands executed without credential prompts, returning correct server FQDNs
- Infrastructure is ready for unattended automation, configuration management, and scheduled maintenance scripts

---

## Tags

`linux` `ssh` `key-based-authentication` `jump-host` `automation` `devops` `security` `xfusioncorp` `nautilus` `infrastructure`
