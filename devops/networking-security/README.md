# Networking and Security

Host-based and network-level security implementations across Linux server fleets. Work in this directory covers firewall hardening, traffic segmentation, and access control enforcement in multi-tier Linux environments. Each implementation reflects production security posture: explicit rule ordering, defense-in-depth, and durable configuration management.

---

## Directory Structure

```
networking-security/
└── iptables-firewall-hardening/    # Host-based firewall enforcement across a 3-server application tier
```

---

## Projects

### [Host-Based Firewall Hardening with iptables Across Multi-Server Application Tier](./iptables-firewall-hardening/)

**Quick Summary:** Configured and persisted iptables rules across three CentOS Stream 9 application servers to enforce single-path ingress, restricting port 5001 exclusively to a trusted load balancer IP while preserving SSH administrative access.

| | |
|---|---|
| **Purpose** | Enforce host-level access control on an application tier where no firewall rules existed. Port 5001 was exposed to all hosts on the network, bypassing the load balancer entirely. |
| **Approach** | Installed `iptables-services` for systemd-integrated persistence. Applied a three-rule chain per server: SSH ACCEPT, source-restricted ACCEPT for the load balancer IP (`172.16.238.14`), and a catch-all DROP for port 5001. Rules saved to `/etc/sysconfig/iptables` for reboot durability. Configuration replicated manually across `stapp01`, `stapp02`, and `stapp03`. |
| **Key Decisions** | Used `DROP` over `REJECT` to avoid leaking port-state information to unauthorized sources. Chose `iptables` over `firewalld` for deterministic rule ordering and scripting compatibility. SSH rule placed first in all chains to prevent administrative lockout during rollout. |
| **Outcome** | All three servers hardened with a uniform security posture. Port 5001 accessible only from the load balancer; all other inbound connections silently dropped. Rules persist across reboots. Manual process documented with Ansible identified as the production-grade path for fleet-wide idempotent enforcement. |

---

## Technologies and Tools

| Tool / Technology | Role |
|---|---|
| `iptables` | Firewall rule engine; INPUT chain management |
| `iptables-services` | Systemd-integrated rule persistence via `/etc/sysconfig/iptables` |
| `systemctl` | Service lifecycle management (enable, start) |
| `yum` / EPEL | Package installation on CentOS Stream 9 |
| CentOS Stream 9 | Target OS for all server configurations |
| SSH / Jump Host | Bastion-mediated administrative access to internal network hosts |

---

## Key Outcomes and Skills Demonstrated

- Applied host-based firewall hardening aligned with defense-in-depth principles in a multi-tier architecture
- Enforced single-path ingress by restricting application port access to a single trusted source IP
- Managed iptables rule ordering to avoid administrative lockout and traffic bypass
- Configured persistent firewall state using `iptables-services` on a distribution that does not persist rules natively
- Replicated a consistent security configuration across a three-server fleet via structured manual execution
- Applied `DROP` vs `REJECT` trade-off analysis appropriate for production external-facing services

---

## How to Navigate

Each subdirectory contains a self-contained `README.md` with full implementation details: architecture diagrams, step-by-step commands with rationale, key decisions, errors encountered, and lessons learned.

Start with the project README linked above. For the command sequence used per server, see the **Step 10** section of that document, which includes the complete reproducible script.

---

## Related Directories

| Directory | Description |
|---|---|
| [`../linux-administration`](../linux-administration/) | Linux system administration tasks: user management, storage, process control |
| [`../configuration-management`](../configuration-management/) | Ansible-based configuration enforcement across server fleets |
| [`../infrastructure-as-code`](../infrastructure-as-code/) | Terraform provisioning across AWS and Azure |
