# AWS Lambda Serverless Function Deployment
#### Deploying a Python-based Greeting Function Using AWS Console and IAM Role Configuration

---

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Resolution Walkthrough](#resolution-walkthrough)
  * [Phase 1 - Console Login and Region Lock](#phase-1---console-login-and-region-lock)
  * [Phase 2 - IAM Execution Role Creation](#phase-2---iam-execution-role-creation)
  * [Phase 3 - Lambda Function Creation](#phase-3---lambda-function-creation)
  * [Phase 4 - Function Code Deployment](#phase-4---function-code-deployment)
  * [Phase 5 - Test and Validation](#phase-5---test-and-validation)
* [Verification Checklist](#verification-checklist)
* [Common Errors and Resolutions](#common-errors-and-resolutions)
* [Key Takeaways](#key-takeaways)

---

## Overview

This document details the end-to-end process of deploying a serverless AWS Lambda function named `nautilus-lambda` using the AWS Management Console. The function is authored in Python 3.12, secured with a least-privilege IAM execution role, and validated through a synchronous invocation test.

This lab demonstrates core serverless deployment competencies required in production DevOps environments: identity scoping, runtime selection, inline code authoring, and functional verification.

| Parameter | Value |
|---|---|
| **Function Name** | `nautilus-lambda` |
| **Runtime** | Python 3.12 |
| **Region** | `us-east-1` (N. Virginia) |
| **IAM Role** | `lambda_execution_role` |
| **Response Body** | `Welcome to KKE AWS Labs!` |
| **Status Code** | `200` |

---

## Problem Statement

The Nautilus DevOps team required a live demonstration of serverless architecture capabilities using AWS Lambda. The objective was to:

1. Provision a scoped IAM execution role with the minimum permissions required for Lambda invocation and CloudWatch logging.
2. Deploy a Python Lambda function that returns a structured JSON response with a custom greeting and HTTP status code `200`.
3. Validate the function end-to-end through a test invocation confirming both the response body and status code.

**Challenge:** No pre-existing IAM role or Lambda function existed. All resources had to be provisioned from scratch in the correct region, in the correct order, with exact naming to satisfy lab grading criteria.

---

## Architecture

```
Invoker (Test Event)
        |
        v
+-------------------------+
|   AWS Lambda            |
|   nautilus-lambda       |
|   Runtime: Python 3.12  |
|   Region: us-east-1     |
+-------------------------+
        |
        | Assumes
        v
+-------------------------+
|   IAM Role              |
|   lambda_execution_role |
|   Policy: AWSLambda     |
|   BasicExecutionRole    |
+-------------------------+
        |
        | Writes logs to
        v
+-------------------------+
|   Amazon CloudWatch     |
|   Logs                  |
+-------------------------+
```

**Execution Flow:**
1. A test event triggers the `lambda_handler` function synchronously.
2. Lambda assumes the `lambda_execution_role` IAM role.
3. The function executes and returns `statusCode: 200` with the greeting body.
4. Execution logs are written to Amazon CloudWatch Logs via the attached policy.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS Console Access | IAM user with Lambda and IAM provisioning permissions |
| Browser | Chrome or Firefox without iframe-blocking extensions |
| Region | Must be locked to `us-east-1` for all operations |
| Permissions | `iam:CreateRole`, `iam:AttachRolePolicy`, `lambda:CreateFunction`, `lambda:UpdateFunctionCode`, `lambda:InvokeFunction` |

> **Note:** If your browser blocks sandboxed iframes, the inline code editor may fail to load. The VS Code-style editor embedded in the Lambda console will still function as a fallback.

---

## Resolution Walkthrough

---

### Phase 1 - Console Login and Region Lock

**Step 1.1** Navigate to the account-specific console URL:

```
https://316890205783.signin.aws.amazon.com/console?region=us-east-1
```

**Step 1.2** Enter the IAM credentials:

```
IAM user name : kk_labs_user_258590
Password      : 
```

**Step 1.3** Confirm the region in the top-right navigation bar reads:

```
United States (N. Virginia) -- us-east-1
```

> **Critical:** IAM is a global service and displays "Global" in the region selector. Once you navigate to Lambda, the region selector must revert to `us-east-1`. Verify this before every resource creation action.

***Screenshot: AWS Console landing page showing `N. Virginia` region confirmed in top-right navigation bar***

<img width="1819" height="942" alt="image" src="https://github.com/user-attachments/assets/7791a788-f67c-4c21-bc7c-ea4268e34c33" />

---

### Phase 2 - IAM Execution Role Creation

Lambda requires an IAM role to assume at execution time. This role must exist before the function is created.

**Step 2.1** Navigate to IAM:

```
Top search bar -> type: IAM -> click "IAM" under Services
```

**Step 2.2** Open the Roles section:

```
Left sidebar -> Roles -> Create role
```

**Step 2.3** Configure the trusted entity:

```
Trusted entity type : AWS service
Use case            : Lambda  (select "Lambda" radio button)
```

***Screenshot: IAM "Select trusted entity" screen with `AWS service` selected and `Lambda` use case highlighted***

<img width="1840" height="941" alt="image" src="https://github.com/user-attachments/assets/50adeaee-ed24-4af2-8e79-ef59f75f7477" />

**Step 2.4** Attach the execution policy:

```
Search box -> type: AWSLambdaBasicExecutionRole -> press Enter
Check the checkbox next to: AWSLambdaBasicExecutionRole
Type column must confirm: AWS managed
```

> **Warning:** Do not select `AWSLambdaFullAccess` or `AWSLambdaVPCAccessExecutionRole`. Only `AWSLambdaBasicExecutionRole` satisfies least-privilege requirements for this function.

**Step 2.5** Name the role:

```
Role name : lambda_execution_role
```

> **Exact spelling required.** All lowercase. Underscores between words. No spaces. No trailing characters.

**Step 2.6** Click **Create role** and confirm the success banner:

```
"Role lambda_execution_role created."
```

***Screenshot: IAM role summary page showing green success banner, `lambda_execution_role` name, `AWSLambdaBasicExecutionRole` policy attached, and Type column confirming `AWS managed`***

<img width="1823" height="947" alt="image" src="https://github.com/user-attachments/assets/d89b3ef7-25f3-4a85-9052-f3270dcc81de" />

---

### Phase 3 - Lambda Function Creation

**Step 3.1** Navigate to Lambda:

```
Top search bar -> type: Lambda -> click "Lambda" under Services
```

**Step 3.2** Confirm region before proceeding:

```
Top-right corner must show: United States (N. Virginia)
```

**Step 3.3** Begin function creation:

```
Click: "Create function"
```

**Step 3.4** Set the authoring method:

```
Selected option : Author from scratch   (default -- do not change)
```

**Step 3.5** Fill in Basic Information:

```
Function name : nautilus-lambda
Runtime       : Python 3.12
Architecture  : x86_64   (default -- do not change)
```

**Step 3.6** Expand **Change default execution role** and configure:

```
Execution role  : Use an existing role
Existing role   : lambda_execution_role
```

***Screenshots: Lambda "Create function" form showing function name `nautilus-lambda`, runtime `Python 3.12`, and `lambda_execution_role` selected in the execution role dropdown***

<img width="1824" height="946" alt="image" src="https://github.com/user-attachments/assets/a37da81c-9b56-436e-b105-3cb5493ae6e0" />
<img width="1821" height="948" alt="image" src="https://github.com/user-attachments/assets/2468a4bd-107f-4da2-a1f0-64f1050bc652" />

**Step 3.7** Click **Create function** and confirm:

```
"Successfully created the function nautilus-lambda"
```

***Screenshot: Lambda function overview page showing green success banner, function ARN `arn:aws:lambda:us-east-1:316890205783:function:nautilus-lambda`, and region confirming `us-east-1`***
<img width="1818" height="949" alt="image" src="https://github.com/user-attachments/assets/71cf299b-3d7b-4958-9f99-45711dbb15b1" />

---

### Phase 4 - Function Code Deployment

**Step 4.1** On the function detail page, click the **Code** tab.

> **Note:** If the browser reports "We couldn't set up the code editor because your browser is blocking a sandboxed HTML iframe," the VS Code-style fallback editor below the banner is fully functional. Use it directly.

**Step 4.2** Clear all existing code in the editor:

```
Click inside the editor
Windows/Linux : Ctrl + A -> Delete
Mac           : Cmd  + A -> Delete
```

**Step 4.3** Enter the following code exactly:

```python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps('Welcome to KKE AWS Labs!')
    }
```

**Line-by-line validation:**

| Line | Expected Content | Notes |
|---|---|---|
| 1 | `import json` | Lowercase, no trailing characters |
| 2 | *(blank)* | Required blank line for PEP 8 compliance |
| 3 | `def lambda_handler(event, context):` | Exact function signature |
| 4 | `    return {` | 4-space indent |
| 5 | `        'statusCode': 200,` | Integer `200`, no quotes |
| 6 | `        'body': json.dumps('Welcome to KKE AWS Labs!')` | Exact string, capital W and K |
| 7 | `    }` | Closing brace, 4-space indent |

> **Critical:** `statusCode` must be the integer `200` without quotes. `"200"` (string) will cause the grader validation to fail.

**Step 4.4** Deploy the code:

```
Click: "Deploy"   (button above the code editor)
```

Confirm the status indicator changes to:

```
DEPLOY -- Current
```

And the success message:

```
"Successfully updated the function nautilus-lambda"
```

***Screenshot: Lambda VS Code editor showing the complete 7-line function code with `DEPLOY - Current` status confirmed in the left panel and green success banner visible at the top***

<img width="1817" height="949" alt="image" src="https://github.com/user-attachments/assets/8a701c51-ba0e-43a4-8a2b-9080bb280ea6" />

---

### Phase 5 - Test and Validation

**Step 5.1** Create a test event by clicking **+ Create new test event** in the left panel under TEST EVENTS.

**Step 5.2** Configure the test event:

```
Event Name      : testEvent
Invocation type : Synchronous   (default -- do not change)
Event sharing   : Private       (default -- do not change)
Template        : hello-world   (default -- do not change)
Event JSON      : leave default payload as-is
```

**Step 5.3** Save the event:

```
Click: "Save"
```

Confirm:

```
"The test event 'testEvent' was successfully saved."
```

**Step 5.4** Invoke the function:

```
Click: "Invoke"
```

**Step 5.5** Validate the execution result in the OUTPUT panel:

```json
{
  "statusCode": 200,
  "body": "\"Welcome to KKE AWS Labs!\""
}
```

Expected terminal output:

```
Status: Succeeded
Test Event Name: testEvent

Response:
{
  "statusCode": 200,
  "body": "\"Welcome to KKE AWS Labs!\""
}
```

> **Note:** The escaped quotes `\"` surrounding the body string are expected behavior. `json.dumps()` serializes the Python string into a JSON-safe format. This is not an error.

***Screenshot: Lambda test OUTPUT panel showing `Status: Succeeded`, `statusCode: 200`, and `body: "\"Welcome to KKE AWS Labs!\""` with the `DEPLOY - Current` and `Lambda Deployed` status confirmations visible***

<img width="1818" height="944" alt="image" src="https://github.com/user-attachments/assets/0c452a80-1d5a-4243-bb14-411c59155aee" />

---

## Verification Checklist

```
 1.  Logged in as kk_labs_user_258590
 2.  Region locked to us-east-1 throughout all phases
 3.  IAM role lambda_execution_role created successfully
 4.  AWSLambdaBasicExecutionRole policy attached (AWS managed)
 5.  Lambda function named nautilus-lambda
 6.  Runtime set to Python 3.12
 7.  Function execution role set to lambda_execution_role
 8.  Code deployed -- DEPLOY status shows "Current"
 9.  Test event testEvent created and saved
 10. Test invocation status shows "Succeeded"
 11. statusCode is integer 200 (not string "200")
 12. body contains exact text: Welcome to KKE AWS Labs!
```

---

## Common Errors and Resolutions

| Error | Root Cause | Resolution |
|---|---|---|
| Region shows "Global" on Lambda page | Navigated from IAM which is global | Manually select `us-east-1` from region dropdown before creating the function |
| `statusCode` returns `"200"` (string) | Quotes accidentally placed around the integer | Remove quotes in code editor, redeploy, retest |
| Body text mismatch | Typo in greeting string | Copy-paste the exact string, redeploy |
| Code editor fails to load (iframe error) | Browser extension blocking sandboxed iframe | Use the VS Code fallback editor below the error banner |
| Role not found in Lambda dropdown | Role name misspelled during creation | Navigate back to IAM, verify role name is exactly `lambda_execution_role` |
| `errorMessage` in test result | Python indentation or syntax error | Check that all indentation uses 4 spaces, not tabs |
| Access Denied on function creation | IAM user lacks `lambda:CreateFunction` permission | Escalate to lab environment support |
| Function not visible after creation | Function created in wrong region | Delete and recreate in `us-east-1` |

---

## Key Takeaways

**IAM Role Must Precede Function Creation**
Lambda cannot be assigned a role that does not yet exist. Always provision the IAM execution role as Phase 1 before opening the Lambda console.

**Region Consistency is Non-Negotiable**
IAM is a global service and resets the region selector to "Global." Every navigation to a regional service (Lambda, CloudWatch) requires an explicit region confirmation before any resource action.

**Deploy is Not Save**
The Lambda console distinguishes between saving a file and deploying the function package. A file save without a subsequent Deploy leaves the live function running the previous code version.

**Least Privilege Execution**
`AWSLambdaBasicExecutionRole` grants only the permissions required for Lambda to write logs to CloudWatch. No additional policies are necessary for a stateless greeting function with no downstream service dependencies.

**json.dumps() Serialization**
Wrapping the response body in `json.dumps()` ensures the Lambda response conforms to the API Gateway proxy integration contract, even when API Gateway is not present. This is a production best practice for all Python Lambda handlers.

---

## References

* [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
* [AWS IAM Roles for Lambda](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)
* [AWSLambdaBasicExecutionRole Policy Reference](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSLambdaBasicExecutionRole.html)
* [Python json.dumps() Documentation](https://docs.python.org/3/library/json.html#json.dumps)
* [AWS Lambda Quotas and Limits](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

---

*AWS Region: us-east-1 | Runtime: Python 3.12*
