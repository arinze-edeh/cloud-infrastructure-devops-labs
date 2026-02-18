# Apache Service Troubleshooting on Port 8087

PROJECT TYPE: `Linux Administration`

CATEGORY: `Service Troubleshooting`

SERVICE: `Apache HTTP Server`

PORT: `8087`

## Project Overview
- This project demonstrates troubleshooting an Apache web service that became unreachable on a non-default port (8087).

- The monitoring system reported that Apache was not accessible, which was discovered to be caused by a combination of a port conflict with an existing process, a configuration syntax error, and firewall restrictions.

- The issue was diagnosed and resolved by terminating the conflicting process, repairing the configuration file, and updating security rules.

## Problem Statement
- Apache service not reachable on port 8087.

- Port 8087 occupied by a conflicting sendmail process (PID 491).

- Syntax error detected on line 45 of httpd.conf.

- Access from jump host failed due to firewall settings.

- index.html must not be modified.

## Objectives

- Verify Apache service status and error logs.

- Identify and resolve port conflicts on 8087.

- Fix Apache configuration syntax errors.

- Ensure Apache listens on port 8087.

- Validate and update firewall configuration.

- Confirm resolution using curl from the jump host.

## Environment Details
JUMP HOST: jumphost

APPLICATION SERVER: stapp01 (172.16.238.10)

USER: tony

PORT: 8087

## Tools Used
`ssh`

`systemctl`

`netstat`

`kill`

`vi / sed`

`iptables`

`curl`

## Step-by-Step Implementation

## Step 1: Initial Diagnosis and Identification of Port Conflict
COMMAND:

RESULT:
`Found process sendmail with PID 491 listening on 127.0.0.1:8087.`

SCREENSHOT:
[Screenshot: Netstat output showing PID 491 using port 8087]

## Step 2: Termination of Conflicting Process
COMMAND:

RESULT:
`Port 8087 was freed for Apache use.`

SCREENSHOT:
<img width="1029" height="715" alt="image" src="https://github.com/user-attachments/assets/27bd64ab-ec14-46b4-80ad-c986ccd8223f" />


## Step 3: Repairing Apache Configuration and Syntax Error
COMMAND:

RESULT:
`Removed syntax error and confirmed "Syntax OK".`

SCREENSHOT:
[Screenshot: Terminal output showing Syntax OK]
<img width="1031" height="316" alt="image" src="https://github.com/user-attachments/assets/55832b85-84bb-44bc-af62-5e5a03c13776" />

## Step 4: Starting and Verifying the Apache Service
COMMAND:

RESULT:
`Service status changed to active (running).`

SCREENSHOT:
[Screenshot: Systemctl status showing active running]
<img width="1033" height="868" alt="image" src="https://github.com/user-attachments/assets/e5dfc3f4-a745-47d0-9e60-20af9e0f84ec" />

## Step 5: Configuring the Firewall for Accessibility
COMMAND:

RESULT:
`Allowed incoming traffic on port 8087, resolving the "not reachable" error reported by monitoring.`

SCREENSHOT:
<img width="1039" height="860" alt="image" src="https://github.com/user-attachments/assets/d88aa054-3921-47f3-ac82-052e4b8387a2" />


## Step 6: Final Validation From Jump Host
COMMAND:

RESULT:
`Received the CentOS HTTP Server Test Page HTML content.`

SCREENSHOT:
[Screenshot: Successful curl output]
<img width="1044" height="879" alt="image" src="https://github.com/user-attachments/assets/3efad4a6-b97c-4916-b246-2d64e32c2501" />


## Validation Checklist
- Apache service running

- Port 8087 listening

- Conflicting sendmail process terminated

- Configuration syntax errors resolved

- Firewall rules updated for 8087

- Service reachable from jump host

- index.html untouched

## Outcome
- The Apache service was successfully restored on port 8087. The port conflict with PID 491 was resolved, the configuration file was repaired, and network accessibility was enabled via iptables.


