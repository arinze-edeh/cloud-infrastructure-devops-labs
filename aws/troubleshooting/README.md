# Troubleshooting

Production incident investigations and remediation walkthroughs covering AWS networking, compute, and access layer failures. Each lab simulates a real-world breakage scenario requiring structured diagnosis, root cause identification, and verified resolution.

---

## Directory Structure
```
aws/troubleshooting/
└── internet-gateway-routing-remediation/
    └── README.md
```

---

## Project Summaries

### [VPC Internet Egress Restoration via Internet Gateway Attachment and Subnet Reachability Remediation](./internet-gateway-routing-remediation/README.md)

**Quick Summary:** Diagnosed and resolved public internet inaccessibility on an EC2-hosted Nginx application. Root cause was a detached Internet Gateway combined with a secondary subnet misconfiguration blocking public IP assignment.

| | |
|---|---|
| **Purpose** | Restore HTTP access to a running Nginx application in a public VPC after deployment left the instance unreachable from the internet despite a correctly configured security group. |
| **Approach** | Executed a structured layer-by-layer investigation: IGW attachment state, route table coverage, subnet public IP attribute, security group rules, and application-layer verification via EC2 Instance Connect. Changes were scoped to confirmed root causes only. |
| **Outcome** | Full public HTTP access restored. `HTTP 200 OK` confirmed via curl validation against the instance public IP. Two misconfigurations resolved: IGW detachment (primary) and `MapPublicIpOnLaunch: False` (secondary). SSH access also unblocked via security group rule addition after SSM was confirmed unavailable. |

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Cloud Platform | AWS (EC2, VPC, IGW, Route Tables, Subnets, Security Groups) |
| CLI | AWS CLI v2 |
| Access | EC2 Instance Connect, SSH (RSA 2048), SSM Session Manager (attempted) |
| Application | Nginx on Amazon Linux 2023 |
| Validation | `curl`, `ss`, `systemctl` |

---

## Key Skills Demonstrated

- VPC networking diagnosis across all layers: IGW, route tables, subnet attributes, security groups
- Structured incident response methodology: collect, diagnose, change, verify at each phase
- EC2 Instance Connect key push with 60-second TTL awareness
- Distinguishing primary root cause from secondary misconfigurations without over-engineering the fix
- CLI-only resolution with no console access

---

## Navigation

Each project folder contains a self-contained `README.md` with full CLI walkthroughs, expected outputs, root cause analysis, and lessons learned. All steps are reproducible from a configured AWS CLI environment with appropriate IAM permissions.

---

*Region: us-east-1*
