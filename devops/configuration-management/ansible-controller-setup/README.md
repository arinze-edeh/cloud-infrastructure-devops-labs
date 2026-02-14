# Ansible Controller Setup on Jump Host (pip3)

## LAB OVERVIEW
- The Nautilus DevOps team selected Ansible as the configuration
management tool due to its simplicity and minimal prerequisites.
- The Jump Host is designated as the Ansible Controller.
- This task installs Ansible version 4.7.0 using pip3 only and ensures
the Ansible binary is globally accessible to all users.

## OBJECTIVES

- Upgrade pip3 on the Jump Host
- Install Ansible version 4.7.0 using pip3
- Resolve pip3 PATH issue
- Verify Ansible is globally executable
- Confirm correct Ansible version

## HIGH-LEVEL LOGIC

- CONNECT to Jump Host
- UPGRADE pip3
- ATTEMPT Ansible installation using pip3
- IF pip3 not found in PATH:
  -  USE absolute pip3 path
- INSTALL Ansible 4.7.0
- VERIFY Ansible binary location
- CONFIRM Ansible runs globally
- VALIDATE Ansible version

## IMPLEMENTATION STEPS

## STEP 1: UPGRADE PIP3

- COMMAND:

`sudo pip3 install --upgrade pip`

- RESULT:
- pip upgraded
- warning about /usr/local/bin not in PATH (acceptable)

## SCREENSHOT:
<img width="1038" height="485" alt="image" src="https://github.com/user-attachments/assets/35ba1ee9-058c-40a1-9711-6b7fc1e9b67b" />

## STEP 2: ATTEMPT ANSIBLE INSTALLATION (FAILURE)
COMMAND:
`sudo pip3 install ansible==4.7.0`

ERROR:
`pip3 command not found`

CAUSE:
- pip3 installed in /usr/local/bin
- sudo PATH does not include this directory

SCREENSHOT:
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/5a235ad7-c556-462b-9925-be48e3be8f21" />

## STEP 3: INSTALL ANSIBLE USING ABSOLUTE PIP3 PATH
- COMMAND:
`sudo /usr/local/bin/pip3 install ansible==4.7.0`

ACTION:
- Downloads Ansible 4.7.0
- Installs ansible-core 2.11.12
- Installs required dependencies

SCREENSHOT:
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/5a235ad7-c556-462b-9925-be48e3be8f21" />
<img width="1035" height="862" alt="image" src="https://github.com/user-attachments/assets/c874d292-26ba-426f-82e4-3fc4eb3de4d1" />

## STEP 4: VERIFY ANSIBLE BINARY LOCATION
- COMMAND:
`ls -l /usr/local/bin/ansible`

- EXPECTED RESULT:
`Executable file owned by root`
- Permissions allow execution by all users

SCREENSHOT:
<img width="1038" height="869" alt="image" src="https://github.com/user-attachments/assets/7ce4deb1-6a7e-42ca-8290-eb81c2c05ee1" />

## STEP 5: VERIFY ANSIBLE INSTALLATION
- COMMAND:
`ansible --version`

EXPECTED RESULT:
- Ansible version 4.7.0
- Executable location: /usr/local/bin/ansible
- Python version 3.9.18

SCREENSHOT:
<img width="1035" height="856" alt="image" src="https://github.com/user-attachments/assets/f4716738-959e-436e-81ba-764aa67ee9a6" />

## FINAL OUTCOME

- Ansible version 4.7.0 installed successfully
- Ansible binary available globally
- Jump Host configured as Ansible Controller
- All users can run Ansible commands

## TAGS

`ansible`
`configuration-management`
`devops`
`automation`
`pip3`
`linux`
