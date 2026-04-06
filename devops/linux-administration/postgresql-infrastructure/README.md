# PostgreSQL User and Database Provisioning on Nautilus Infrastructure

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure Context](#infrastructure-context)
- [Task Requirements](#task-requirements)
- [Solution Approach](#solution-approach)
- [Implementation](#implementation)
  - [Step 1: Connect to Database Server via Jump Host](#step-1-connect-to-database-server-via-jump-host)
  - [Step 2: Switch to PostgreSQL System User](#step-2-switch-to-postgresql-system-user)
  - [Step 3: Create PostgreSQL User](#step-3-create-postgresql-user)
  - [Step 4: Create PostgreSQL Database](#step-4-create-postgresql-database)
  - [Step 5: Grant Database Privileges](#step-5-grant-database-privileges)
  - [Step 6: Verify Database Creation](#step-6-verify-database-creation)
  - [Step 7: Verify User Creation](#step-7-verify-user-creation)
  - [Step 8: Correct Password Case (Final Compliance Step)](#step-8-correct-password-case-final-compliance-step)
- [Validation Summary](#validation-summary)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Security and Best Practices](#security-and-best-practices)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)

---

## Overview

This document details the provisioning of a PostgreSQL database user and dedicated database on the Nautilus infrastructure database server (`stdb01`). The operation was performed to support a newly onboarded application requiring isolated database access with scoped credentials.

All changes were applied to the live PostgreSQL instance **without a service restart**, preserving zero downtime in compliance with the operational constraint imposed by the task.

---

## Problem Statement

A new application being deployed within the Nautilus infrastructure required a dedicated PostgreSQL user and database with full access privileges. The provisioning had to meet the following non-negotiable constraints:

- A named database user with a specific password must be created
- A dedicated database must be created and owned within the PostgreSQL cluster
- Full database-level privileges must be granted to that user
- **The PostgreSQL service must not be restarted at any point during or after provisioning**

Failure to respect the no-restart requirement risked disrupting any active connections or dependent services on the shared database host.

---

## Infrastructure Context

| Component | Details |
|---|---|
| Environment | Nautilus Infrastructure (xFusion DevOps) |
| Database Server | `stdb01.stratos.xfusioncorp.com` |
| Server IP | `172.17.0.4` |
| Access Path | Jump Host (`thor@jumphost`) to DB Server |
| Database Engine | PostgreSQL |
| Privilege Scope | Database-level (full access) |
| Operational Constraint | No PostgreSQL service restart permitted |

---

## Task Requirements

| Requirement | Value |
|---|---|
| PostgreSQL Username | `kodekloud_joy` |
| Password | `TmPcZjtRQx` |
| Database Name | `kodekloud_db4` |
| Privilege Level | `ALL PRIVILEGES` on `kodekloud_db4` |
| Service Restart | Not permitted |

---

## Solution Approach

PostgreSQL supports live DDL operations via the `psql` client without requiring a service restart. The approach taken was:

1. SSH from the jump host into the database server using the application account `peter`
2. Escalate to the `postgres` OS user (the PostgreSQL superuser) using `sudo`
3. Execute all provisioning commands non-interactively via `psql -c` to avoid entering the interactive shell
4. Validate all changes using PostgreSQL meta-commands (`\l`, `\du`)
5. Correct a password case mismatch identified post-provisioning via `ALTER USER`

This approach is safe, auditable, and production-appropriate.

---

## Implementation

### Step 1: Connect to Database Server via Jump Host

SSH from the jump host (`thor@jumphost`) into the database server using the application account `peter`. Accept the host key fingerprint on first connection; this permanently adds the host to the known hosts list.

```bash
ssh peter@stdb01.stratos.xfusioncorp.com
```

**Expected output:**
- Host authenticity warning (first-time connection)
- ED25519 fingerprint confirmation prompt
- Successful shell session as `peter@stdb01`

> **Operational Note:** On first connection, SSH presents the host key fingerprint for manual verification. In production environments, this should be validated against a known-good fingerprint before accepting.

**Screenshot: SSH connection from jump host to database server**

![SSH connection from jumphost to stdb01 — host key accepted and session established as peter](https://github.com/user-attachments/assets/4b22da20-6c2c-45c2-bb61-60c417359241)

---

### Step 2: Switch to PostgreSQL System User

Escalate from the `peter` user to the `postgres` OS user. The `postgres` system account is the default superuser for the PostgreSQL database engine on Linux systems and has the authority to perform all administrative DDL operations.

```bash
sudo -i -u postgres
```

**Expected output:**
- Sudo lecture displayed on first use (informational only)
- Shell prompt changes to `postgres@stdb01`

> **Why `-i`?** The `-i` flag simulates a login shell, ensuring that `postgres` user environment variables (including `PATH` and `PGDATA`) are correctly initialized before executing any `psql` commands.

**Screenshot: Escalated to postgres system user via sudo**

![sudo -i -u postgres executed successfully — shell context switched to postgres@stdb01](https://github.com/user-attachments/assets/02b9e10f-ce9a-49d8-a791-3519db91f1a6)

---

### Step 3: Create PostgreSQL User

Create the required database role using the `CREATE USER` DDL statement. The `-c` flag passes the SQL inline without entering the interactive `psql` shell.

```bash
psql -c "CREATE USER kodekloud_joy WITH PASSWORD 'TmPcZjtrQx';"
```

**Expected output:**

```
CREATE ROLE
```

> **Technical Note:** In PostgreSQL, `CREATE USER` is syntactic sugar for `CREATE ROLE WITH LOGIN`. The role is immediately usable for authentication upon creation, with no service restart required.

> **Password Case Sensitivity:** PostgreSQL passwords are case-sensitive and stored as hashed values. A lowercase `r` was used at this stage; this is corrected in [Step 8](#step-8-correct-password-case-final-compliance-step) to match the exact required value.

**Screenshot: PostgreSQL user kodekloud_joy created successfully**

![psql CREATE USER command executed — CREATE ROLE confirmation returned](https://github.com/user-attachments/assets/46c1ef45-0507-43e9-a54d-4fbda452526a)

---

### Step 4: Create PostgreSQL Database

Create the dedicated database for the application using the `CREATE DATABASE` DDL statement.

```bash
psql -c "CREATE DATABASE kodekloud_db4;"
```

**Expected output:**

```
CREATE DATABASE
```

> **Ownership:** By default, the database is owned by the `postgres` superuser. Ownership can be transferred to `kodekloud_joy` if required, though `GRANT ALL PRIVILEGES` (Step 5) achieves the same effective access without changing ownership.

**Screenshot: Database kodekloud_db4 created successfully**

![psql CREATE DATABASE command executed — CREATE DATABASE confirmation returned](https://github.com/user-attachments/assets/7f13cb10-fcd4-446f-a613-2553591ca77d)

---

### Step 5: Grant Database Privileges

Grant full database-level privileges on `kodekloud_db4` to `kodekloud_joy`. This gives the user unrestricted access to connect, create schemas, and manage objects within the database.

```bash
psql -c "GRANT ALL PRIVILEGES ON DATABASE kodekloud_db4 TO kodekloud_joy;"
```

**Expected output:**

```
GRANT
```

> **Privilege Scope:** `GRANT ALL PRIVILEGES ON DATABASE` covers the `CONNECT`, `CREATE`, and `TEMPORARY` privileges at the database level. For schema-level or table-level access, additional grants within the database would be required as objects are created.

> **Principle of Least Privilege:** In production environments, consider scoping grants more narrowly (e.g., granting only `CONNECT` and `TEMPORARY`, with object-level grants applied post-schema creation). For this use case, full database-level access was specified in requirements.

**Screenshot: Full privileges granted on kodekloud_db4 to kodekloud_joy**

![GRANT ALL PRIVILEGES command executed successfully — GRANT confirmation returned](https://github.com/user-attachments/assets/81ed52da-892e-4d2e-9eaa-8940e8fd4f29)

---

### Step 6: Verify Database Creation

Validate the existence of `kodekloud_db4` and confirm that the access privilege for `kodekloud_joy` is correctly reflected in the PostgreSQL catalog.

```bash
psql -c "\l"
```

**Expected output (relevant row):**

```
kodekloud_db4 | postgres | SQL_ASCII | C | C | =Tc/postgres
                                                 postgres=CTc/postgres
                                                 kodekloud_joy=CTc/postgres
```

- `C` = CONNECT privilege
- `T` = TEMPORARY privilege
- `c` = CREATE privilege (abbreviated in access privileges notation)
- The presence of `kodekloud_joy=CTc/postgres` confirms successful privilege assignment

**Screenshot: Database list confirming kodekloud_db4 existence and privilege assignment**

![psql \l output showing kodekloud_db4 listed with kodekloud_joy access privileges confirmed](https://github.com/user-attachments/assets/2729a174-9a48-4431-ae36-cdbe1bcf8ab5)

---

### Step 7: Verify User Creation

Confirm that the `kodekloud_joy` role exists in the PostgreSQL role catalog and review its assigned attributes.

```bash
psql -c "\du"
```

**Expected output (relevant rows):**

```
Role name      | Attributes                                            | Member of
kodekloud_joy  |                                                       | {}
postgres       | Superuser, Create role, Create DB, Replication, ...  | {}
```

- `kodekloud_joy` appears with no superuser attributes, confirming it is a standard login role
- The empty `Member of` column (`{}`) confirms no group membership, consistent with a purpose-scoped application account

**Screenshot: Role list confirming kodekloud_joy exists as a non-privileged login role**

![psql \du output listing kodekloud_joy alongside the postgres superuser role](https://github.com/user-attachments/assets/3a2d2f30-dcfe-4ba0-a1e4-d556c1b88cf3)

---

### Step 8: Correct Password Case (Final Compliance Step)

During post-provisioning review, the password set in Step 3 was identified as containing a lowercase `r` (`TmPcZjtrQx`) rather than the required uppercase `R` (`TmPcZjtRQx`). PostgreSQL passwords are case-sensitive at the authentication layer, making this a compliance-critical correction.

The password was updated in-place using `ALTER USER`, with no service restart required:

```bash
psql -c "ALTER USER kodekloud_joy WITH PASSWORD 'TmPcZjtRQx';"
```

**Expected output:**

```
ALTER ROLE
```

> **Why this matters:** Case mismatches in credentials are a common source of authentication failures that are difficult to trace. Correcting the password before handing off the environment prevents downstream connection issues for the application team.

> **Security Note:** `ALTER USER ... WITH PASSWORD` updates the stored password hash atomically. All new connections after this point use the updated credential; existing sessions are unaffected.

**Screenshot: Password corrected via ALTER USER with confirmed ALTER ROLE response**

![ALTER USER command executed to correct password case mismatch — ALTER ROLE confirmation returned](https://github.com/user-attachments/assets/6713a9c5-662b-4bb2-821a-9f0189ecc1ef)

---

## Validation Summary

| Validation Check | Result |
|---|---|
| User `kodekloud_joy` created | Confirmed via `\du` |
| Database `kodekloud_db4` created | Confirmed via `\l` |
| Full privileges granted on `kodekloud_db4` | Confirmed in access privileges column (`\l`) |
| Password matches exact requirement (`TmPcZjtRQx`) | Corrected and confirmed via `ALTER USER` |
| PostgreSQL service restarted | Not restarted (constraint satisfied) |

---

## Errors and Resolutions

| Issue | Root Cause | Resolution |
|---|---|---|
| Password set with lowercase `r` (`TmPcZjtrQx`) instead of uppercase `R` (`TmPcZjtRQx`) | Manual transcription error during `CREATE USER` | Identified via post-provisioning review; corrected in-place using `ALTER USER kodekloud_joy WITH PASSWORD 'TmPcZjtRQx';` |

---

## Key Decisions

- **`psql -c` over interactive shell:** All SQL was executed non-interactively using `psql -c`. This approach is more auditable, scriptable, and less prone to interactive input errors than entering the `psql` REPL.
- **No service restart:** All DDL operations in PostgreSQL (role and database creation, privilege grants, password updates) are applied to the live catalog without requiring a restart. This is a deliberate design property of the PostgreSQL architecture.
- **`sudo -i -u postgres` over `sudo -u postgres`:** The `-i` flag was used to load the full login environment for the `postgres` OS user, ensuring `psql` resolves the correct socket path and environment variables.
- **Post-provisioning password audit:** A deliberate review of the provisioned credentials against requirements was performed before marking the task complete, catching the case mismatch in the password.

---

## Security and Best Practices

- **Passwords are hashed:** PostgreSQL stores all passwords as salted hashes (MD5 or SCRAM-SHA-256 depending on `pg_hba.conf` configuration). Plaintext passwords are never persisted.
- **Zero-downtime provisioning:** All operations were performed live with no service disruption, demonstrating that PostgreSQL DDL can be safely applied to running systems.
- **Scoped access:** The `kodekloud_joy` role holds no superuser attributes, no `CREATE ROLE`, and no `CREATE DB` permissions. Access is strictly limited to `kodekloud_db4`.
- **Audit trail:** All commands were executed as the `postgres` OS user via `sudo`, which is logged by the system audit daemon. This provides traceability for compliance purposes.
- **Credential hygiene:** Post-provisioning credential verification against requirements is standard practice and should be incorporated into any database provisioning checklist.

---

## Lessons Learned

- **Case sensitivity in credentials is a silent failure vector.** PostgreSQL will accept both `TmPcZjtrQx` and `TmPcZjtRQx` as valid password strings during `CREATE USER`, but authentication against either will only succeed with the exact value used at connection time. Always perform a case-accurate review against documented requirements before handoff.
- **`GRANT ALL PRIVILEGES ON DATABASE` is not table-level access.** Downstream engineers should be aware that this grant covers connection and schema creation within the database, but not pre-existing table objects. Schema-level grants (`GRANT ALL ON ALL TABLES IN SCHEMA`) would be required if objects are pre-seeded.
- **`sudo -i` is preferable to `sudo su`** when switching to the `postgres` OS user. It is cleaner, more auditable, and avoids subshell nesting issues.

---

## Outcome

The PostgreSQL role `kodekloud_joy` and database `kodekloud_db4` were successfully provisioned on `stdb01.stratos.xfusioncorp.com`. Full database-level privileges were granted and confirmed via catalog inspection. A post-provisioning password case correction was applied and verified. The PostgreSQL service remained uninterrupted throughout. The environment is ready for application-level database integration.
