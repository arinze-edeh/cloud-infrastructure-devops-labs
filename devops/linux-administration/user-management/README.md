# user-management

> Linux user lifecycle management for production and enterprise-style environments, covering service account hardening and time-bound access provisioning.

---

## Overview

Service account hygiene and access lifecycle management are foundational controls in any production Linux environment. Misconfigured or long-lived accounts are a persistent source of audit findings, compliance gaps, and lateral movement risk.

This directory contains hands-on implementations of two core user management patterns: locking down service identities to prevent interactive shell access, and enforcing automatic account expiry for temporary personnel. Both tasks were executed against live infrastructure within a multi-server bastion-host architecture, following patterns directly applicable to SOC 2, PCI-DSS, and CIS Benchmark requirements.

---

## Directory Structure

```
user-management/
├── non-interactive-user-creation/   # Service account provisioning with /sbin/nologin
├── temporary-user-expiry/           # Time-bound account creation with enforced expiry
└── README.md
```

---

## Project Summaries

### [Non-Interactive User Creation](./non-interactive-user-creation)

**Quick Summary:** Provisions a locked-down Linux service account (`kareem`) on a production application server, assigning `/sbin/nologin` to block all interactive login vectors while preserving the identity for process ownership and file permissions.

**Purpose:** Background agents, CI/CD runners, and monitoring daemons require a dedicated system identity without login capability. Assigning a standard shell to these accounts creates unnecessary attack surface.

**Approach:**
- Connected to the target server (`stapp01`) via SSH through a jump host, following a bastion-host access pattern
- Created the account using `useradd -s /sbin/nologin kareem`, atomically binding the non-interactive shell at creation time
- Verified the passwd entry with `getent` (NSS-aware, works across LDAP and local auth sources)
- Confirmed interactive access is blocked by attempting `su - kareem` and validating the rejection

**Outcome:** Account fully provisioned and hardened. Interactive login is rejected at the OS level regardless of password state. Approach is reusable for any service identity requiring process ownership without shell access.

---

### [Temporary User Expiry](./temporary-user-expiry)

**Quick Summary:** Creates a time-bound developer account (`mariyam`) on a production application server with an enforced expiry of `2026-12-07`, eliminating the need for manual deprovisioning and reducing orphaned credential risk.

**Purpose:** Contractor and short-term developer accounts frequently outlive their intended lifecycle when deprovisioning relies on manual processes. Encoding expiry at creation time ensures access revocation is automatic and audit-ready.

**Approach:**
- Multi-hop SSH from jump host to target server (`stapp03`) using a sudo-enabled account
- Escalated to root via `sudo -i` for a clean, environment-isolated privilege session
- Created the account using `useradd -e 2026-12-07 mariyam`, encoding expiry in `/etc/shadow` as days since epoch
- Validated expiry configuration with `chage -l` and confirmed system identity registration via `getent passwd`

**Outcome:** Account provisioned with kernel-level expiry enforcement. Login is automatically blocked after the expiry date regardless of credential state, with no dependency on external tooling or manual review cycles.

---

## Technologies and Tools

| Tool | Role |
|---|---|
| `useradd` | Account creation with shell and expiry configuration |
| `usermod` / `chage` | Account modification and aging policy management |
| `getent` | NSS-aware identity verification across local and directory sources |
| `su` | Interactive login validation during post-provision testing |
| `/sbin/nologin` | OS-level shell binary that blocks interactive sessions |
| SSH | Bastion-host access chaining from jump host to target servers |
| `sudo` / `sudo -i` | Auditable privilege escalation for system-level operations |

---

## Key Skills Demonstrated

- **Service Account Hardening:** Applying non-interactive shells to eliminate login vectors for system identities
- **Time-Bound Access Control:** Using native Linux tooling to enforce automatic deprovisioning without external dependencies
- **Bastion-Host Navigation:** Multi-hop SSH access patterns used in segmented enterprise networks
- **Identity Verification:** Querying NSS via `getent` to validate accounts across mixed authentication environments
- **Compliance Alignment:** Procedures map directly to CIS Benchmark L1, SOC 2 CC6.1, PCI-DSS Requirement 8, and ISO 27001 A.9 controls
- **Least-Privilege Enforcement:** Scoping account capabilities (shell, expiry, sudo) to the minimum required by each workload

---

## How to Navigate

Each subdirectory contains a full `README.md` with:
- Environment and architecture details
- Step-by-step implementation with annotated commands
- Screenshots from live execution
- Security best practices and operational edge cases

Start with [`non-interactive-user-creation`](./non-interactive-user-creation) for service account hardening patterns, then [`temporary-user-expiry`](./temporary-user-expiry) for time-bound access provisioning. Both documents are self-contained and written for direct reuse in production workflows.
