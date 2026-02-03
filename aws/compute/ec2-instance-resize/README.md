# EC2 Instance Right-Sizing (Cost Optimization)

## Project Context
The Nautilus DevOps team identified an underutilized EC2 instance during an infrastructure audit.  
To optimize cost and resource utilization, the instance type was safely downgraded while ensuring service continuity.

This lab demonstrates **controlled EC2 lifecycle management**, **change safety**, and **cost-aware decision making**.

---

## Cloud Provider
AWS

## Region
us-east-1 (N. Virginia)

---

## Objective

GOAL:
-  Change EC2 instance type from t2.micro → t2.nano
-  Ensure instance remains healthy and running post-change
Preconditions
## REQUIREMENTS:
-  Instance exists with name "nautilus-ec2"
-  Status checks must pass before modification
-  Instance must be stopped before resizing

## Step 1: Authenticate into AWS Console
- OPEN AWS Console
- LOGIN using provided credentials
- VERIFY region = us-east-1

📸 Screenshot:
<img width="1741" height="945" alt="image" src="https://github.com/user-attachments/assets/a9491e94-5d4f-4216-aca0-4bdef15b1af9" />
AWS Console dashboard with region set to us-east-1

## Step 2: Locate EC2 Instance
- NAVIGATE to EC2 Dashboard
- OPEN Instances
- SEARCH for instance named "nautilus-ec2"

📸 Screenshot:
<img width="1761" height="915" alt="image" src="https://github.com/user-attachments/assets/d80b6ca9-1bfa-46ad-9aa0-9bb11c03664a" />
EC2 instance list showing nautilus-ec2

## Step 3: Verify Instance Health
- CHECK instance status
- WAIT until Status Checks == "2/2 checks passed"
- DO NOT proceed if status == initializing

📸 Screenshot:
<img width="1685" height="949" alt="image" src="https://github.com/user-attachments/assets/34a63182-a9d2-44f3-9ba3-3b3db04b5127" />
Instance status checks showing 2/2 passed

## Step 4: Stop EC2 Instance
- SELECT nautilus-ec2
- CHANGE instance state → Stop
- WAIT until instance state == stopped

📸 Screenshot:
<img width="1917" height="950" alt="image" src="https://github.com/user-attachments/assets/d3931cd9-eb7b-434f-a3d0-2f9cd51dcc58" />
Instance state showing "stopped"

## Step 5: Modify Instance Type
- OPEN Actions menu
- SELECT Instance settings → Change instance type
- SET instance type = t2.nano
- APPLY changes

📸 Screenshot:
<img width="1894" height="902" alt="image" src="https://github.com/user-attachments/assets/4705c764-34e5-4750-bacb-76455e4e63b4" />
Change instance type dialog with t2.nano selected

## Step 6: Start EC2 Instance
- SELECT nautilus-ec2
- CHANGE instance state → Start
- WAIT until instance state = running

📸 Screenshot:
<img width="1904" height="950" alt="image" src="https://github.com/user-attachments/assets/172ffa53-09c9-42ec-b839-589f158e8408" />
Instance state showing "running"

## Step 7: Final Validation

- VERIFY:
  -  Instance name = nautilus-ec2
  -  Instance type == t2.nano
  -  Instance state == running
  -  Region == us-east-1

📸 Screenshot:
<img width="1904" height="950" alt="image" src="https://github.com/user-attachments/assets/172ffa53-09c9-42ec-b839-589f158e8408" />
EC2 instance details showing t2.nano and running state

## Outcome
RESULT:
-  EC2 instance successfully right-sized
-  Cost optimized without service disruption

## Skills Demonstrated
SKILLS:
-  EC2 lifecycle management
-  Cloud cost optimization (FinOps basics)
-  Safe infrastructure change execution
-  AWS Console navigation

## Recruiter Notes
- This project reflects real-world DevOps practices:

  -  Production-safe instance modification

  -  Cost-awareness in cloud environments

  -  Operational discipline using health checks
