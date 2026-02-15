# AWS EC2 Instance Cleanup (us-east-1)

## PROJECT OVERVIEW
- During infrastructure migration, obsolete AWS resources were identified.
- An EC2 instance named "devops-ec2" is no longer required and must be deleted.
- The task ensures proper cleanup and confirms the instance reaches
the TERMINATED state.

## OBJECTIVES
- Identify EC2 instance named devops-ec2
- Ensure correct AWS region (us-east-1)
- Terminate the instance
- Verify instance is fully terminated

## HIGH-LEVEL WORKFLOW

- LOGIN to AWS Console
- SET region to us-east-1
- OPEN EC2 service
- SEARCH for instance named devops-ec2
- IF instance exists:
  -  TERMINATE instance
  -  WAIT for termination
- VERIFY instance state == terminated
- END task

## IMPLEMENTATION STEPS

## STEP 1: LOGIN TO AWS CONSOLE
- ACTION:
  -  OPEN browser
  -  NAVIGATE to AWS Console URL
  -  LOGIN using provided credentials

SCREENSHOT:
<img width="1819" height="942" alt="image" src="https://github.com/user-attachments/assets/302cb83e-2856-49fd-93f8-c423c4c1384b" />

## STEP 2: SET AWS REGION
- ACTION:
  -  SELECT region dropdown
  -  CHOOSE `us-east-1`

SCREENSHOT:
<img width="1818" height="945" alt="image" src="https://github.com/user-attachments/assets/ed4b9efc-85bd-47cc-9542-62b65d24f501" />

## STEP 3: OPEN EC2 DASHBOARD
- ACTION:
  -  NAVIGATE to Services
  -  SELECT EC2

SCREENSHOT:
<img width="1839" height="889" alt="image" src="https://github.com/user-attachments/assets/df932739-dbd6-42e2-99a7-2acce0c7841c" />

## STEP 4: LOCATE TARGET INSTANCE
- ACTION:
  -  OPEN Instances
  -  SEARCH for instance name "devops-ec2"

- EXPECTED RESULT:
  -  Instance visible in instance list

SCREENSHOT:
<img width="1840" height="893" alt="image" src="https://github.com/user-attachments/assets/8b9886c6-3473-40a2-be02-9db11770c87e" />

## STEP 5: TERMINATE INSTANCE
- ACTION:
  -  SELECT devops-ec2
  -  CLICK Instance State
  -  SELECT Terminate Instance
  -  CONFIRM termination

SCREENSHOT:
<img width="1805" height="892" alt="image" src="https://github.com/user-attachments/assets/543dedde-d4b9-4136-ba33-84c4aee0c6d4" />

## STEP 6: VERIFY TERMINATION
- ACTION:
  -  WAIT until instance state changes
  -  CONFIRM state == terminated

SCREENSHOT:
<img width="1844" height="897" alt="image" src="https://github.com/user-attachments/assets/7ca6003f-9dfd-481d-9779-6a114a7ce22b" />

## FINAL VERIFICATION

- INSTANCE NAME:
`devops-ec2`

REGION:
`us-east-1`

FINAL STATE:
`terminated`

TASK STATUS:
`SUCCESSFUL`

## TAGS

`aws`
`ec2`
`cloud-cleanup`
`infrastructure`
`devops`







<img width="1802" height="892" alt="image" src="https://github.com/user-attachments/assets/51017336-e867-4939-b36a-deaf5995b441" />





