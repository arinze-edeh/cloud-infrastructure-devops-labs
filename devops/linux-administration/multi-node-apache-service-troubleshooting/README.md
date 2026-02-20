
# Multi-Node Apache Service Outage Resolution (Stratos DC)

## Incident Summary
- Production monitoring detected Apache (`httpd`) service unavailability on one
Nautilus application server within Stratos DC.

- The incident was caused by a port-binding conflict that prevented Apache from
starting successfully. The issue was diagnosed, remediated, and validated
across all application nodes to restore service consistency.

---

## Environment
- Platform: Linux (systemd)
- Service: Apache HTTP Server (httpd)
- Scope: Multi-node application tier
- Nodes:
  - stapp01 (172.16.238.10)
  - stapp02 (172.16.238.11)
  - stapp03 (172.16.238.12)

---

## Impact
- Apache service failed to start on one application node
- Monitoring alert triggered due to service downtime
- Risk of inconsistent application-tier behavior

---

## Root Cause Analysis
- Apache failed to start due to an **address already in use** error.

- Investigation revealed that the required application port was already bound
by another system service (`sendmail`), causing Apache to exit with a failure
status during startup.

---

## Detection & Diagnosis

`sudo systemctl status httpd`

- Observed

`httpd.service` in failed state

Exit code due to socket binding failure

📸 Screenshot: `httpd failed service status`
<img width="1033" height="824" alt="image" src="https://github.com/user-attachments/assets/464c0840-06ba-4115-ba60-66af0bf9bfa2" />

`sudo ss -tulnp | grep :8086`

- Finding

`Port 8086 occupied by sendmail`

📸 Screenshot: `sendmail listening on port 8086`


## Remediation Steps

## 1. Stop Conflicting Service
`sudo systemctl stop sendmail`

📸 Screenshot: sendmail stopped


## 2. Restore Apache Service
`sudo systemctl start httpd`

📸 Screenshot: httpd started successfully


## 3. Validate Port Binding
`sudo ss -tulnp | grep :8086`

- Result

Apache successfully listening on required port

📸 Screenshot: `httpd bound to port 8086`


## 4. Enforce Persistence
`sudo systemctl enable httpd`

📸 Screenshot: `httpd enabled at boot`


## 5. Cross-Node Validation

- The following checks were performed on all application servers to ensure
configuration consistency:

`sudo ss -tulnp | grep :8086`
`sudo systemctl status httpd`
`sudo systemctl enable httpd`

📸 Screenshot: `Apache running and enabled on all nodes`
<img width="1038" height="267" alt="image" src="https://github.com/user-attachments/assets/c40573c4-db2c-4afc-81a1-f022ef4e76c0" />


## Post-Incident State

- Apache service running on all application servers

- Port conflict resolved

- Service enabled to persist across reboots

- Monitoring alert cleared



## SRE Competencies Demonstrated

- Incident response and service restoration

- Root cause analysis

- Linux service lifecycle management

- Port conflict diagnosis

- Multi-node consistency validation

- Production reliability mindset




<img width="805" height="830" alt="image" src="https://github.com/user-attachments/assets/9465cb53-f146-4931-a77a-f4416a3b2d86" />
<img width="1027" height="579" alt="image" src="https://github.com/user-attachments/assets/d47fea25-7236-4572-931a-2634fc474d60" />
<img width="1033" height="412" alt="image" src="https://github.com/user-attachments/assets/0c023d6f-1e3c-44c3-9089-f3afe1f2c3a0" />
<img width="1035" height="485" alt="image" src="https://github.com/user-attachments/assets/1d05e776-0e5a-4e46-a7ba-90005b14bd6d" />
<img width="1035" height="528" alt="image" src="https://github.com/user-attachments/assets/fe25907f-faa1-4a43-b83b-6669ee6a4246" />
<img width="1034" height="403" alt="image" src="https://github.com/user-attachments/assets/2bda11df-b624-4009-84af-2276752951b0" />
<img width="1036" height="659" alt="image" src="https://github.com/user-attachments/assets/ae7cbf40-79a3-4d75-98fe-59a9efe2fb96" />
<img width="1030" height="404" alt="image" src="https://github.com/user-attachments/assets/6a7f53c9-36f9-4796-85f6-0258cc754285" />
<img width="1030" height="383" alt="image" src="https://github.com/user-attachments/assets/ee6783e7-1902-485e-925d-17f1a1626530" />






