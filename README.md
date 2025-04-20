# AWS Budget Automation for ECS & RDS Shutdown

This guide shows you how to configure an **AWS Budget** that automatically invokes a Lambda function to stop your ECS tasks and RDS instances when your costs exceed a threshold.

---

## 📋 Prerequisites

- AWS account with permissions to manage IAM, Budgets, Lambda, ECS, RDS (and SNS if you add notifications)  
- A deployed Lambda function that:
  - Sets ECS service desired count to 0  
  - Stops RDS instances  
- The Lambda function’s ARN (e.g. `arn:aws:lambda:<AWS_REGION>:<AWS_ACCOUNT_ID>:function:<FUNCTION_NAME>`)

---

## 1. Create an IAM Role for AWS Budgets

AWS Budgets must be able to call your Lambda. Create (or reuse) a role with a trust policy for `budgets.amazonaws.com`.

### 1.1 Create the Role

1. **IAM → Roles → Create role**  
2. **Trusted entity**:  
   - Choose **AWS service**  
   - Use case: **Budgets**  
3. **Skip** attaching permissions for now → **Next**  
4. **Name** it (e.g. `BudgetsInvokeLambdaRole`) → **Create role**

### 1.2 Edit the Trust Policy

1. In the role’s page, go to **Trust relationships → Edit trust policy**  
2. Replace with:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": { "Service": "budgets.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
           "StringEquals": {
             "AWS:SourceAccount": "<AWS_ACCOUNT_ID>"
           }
         }
       }
     ]
   }
Save.

1.3 Attach Invoke Permissions
In Permissions policies, click Add inline policy → JSON

Paste:

json
Copy
Edit
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:<AWS_REGION>:<AWS_ACCOUNT_ID>:function:<FUNCTION_NAME>"
    }
  ]
}
Review → name it InvokeLambdaPolicy → Create policy.

2. Create a Cost Budget with Actions
Billing & Cost Management → Budgets → Create budget

Choose Cost budget, click Next

Name your budget (e.g. <BUDGET_NAME>)

Set Period (e.g. Monthly) and Amount (Fixed or Planned) → Next

Under Filter, (optional) scope by:

Service: Amazon Elastic Container Service, Amazon RDS

Tags, accounts, regions, etc.

Under Actions & notifications → Enable actions → Yes

IAM role: select BudgetsInvokeLambdaRole

Trigger:

Threshold type: Actual or Forecasted

Threshold: e.g. 90% or 100%

Action type: Run an AWS Lambda function

Function: select your <FUNCTION_NAME>

(Optional) Add notifications:

Email recipients or an SNS topic

Match threshold settings

Review → Create budget

3. Verify Your Lambda’s Execution Role
Ensure the role your Lambda uses has permissions to stop ECS and RDS:

json
Copy
Edit
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:ListClusters",
        "ecs:ListServices",
        "ecs:DescribeServices",
        "ecs:UpdateService",
        "rds:DescribeDBInstances",
        "rds:StopDBInstance"
      ],
      "Resource": "*"
    }
  ]
}
IAM → Roles → select your Lambda’s execution role

Attach (or add inline) the policy above.

4. Test the Automation
Isolate Lambda:

In Lambda console, Test with an empty event {}

Confirm it stops your ECS & RDS resources

Simulate Budget Breach (carefully):

Lower your budget amount below current spend

Wait up to 24 hrs for AWS Budgets to trigger

Verify:

ECS tasks should scale to 0

RDS instances should stop

Check CloudWatch Logs for your Lambda

Clean up:

Restore your original budget

Restart any ECS/RDS resources as needed

This guide provides a general framework for setting up budget-based automated shutdown of ECS and RDS resources. Remember to replace the placeholder ARNs and account IDs with your actual values.

