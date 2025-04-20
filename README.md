# AWS Budget Automation for EC2 & RDS Shutdown

This guide shows you how to configure an **AWS Budget** that automatically invokes a Lambda function to stop your EC2 instances and RDS databases when your costs exceed a defined threshold.

---

## 📋 Prerequisites

- An AWS account with permissions to manage IAM, Budgets, Lambda, EC2, and RDS
- A deployed Lambda function that:
  - Stops specific EC2 instances (via `ec2:StopInstances`)
  - Stops RDS databases (via `rds:StopDBInstance`)
- The Lambda function’s ARN (e.g., `arn:aws:lambda:<REGION>:<ACCOUNT_ID>:function:<FUNCTION_NAME>`)

---

## 1. Create an IAM Role for AWS Budgets to Invoke Lambda

AWS Budgets must be allowed to call your Lambda function. You’ll create a role that grants this permission.

### 1.1 Create the Role

1. Go to **IAM → Roles → Create role**
2. **Trusted entity type**: AWS service  
3. **Use case**: Budgets  
4. **Skip attaching permissions for now**, click **Next**  
5. Name the role (e.g., `BudgetsInvokeLambdaRole`)  
6. Click **Create role**

### 1.2 Edit the Trust Policy

1. Open your newly created role → **Trust relationships → Edit trust policy**
2. Paste the following, replacing `<ACCOUNT_ID>` with your AWS Account ID:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Service": "budgets.amazonaws.com"
         },
         "Action": "sts:AssumeRole",
         "Condition": {
           "StringEquals": {
             "AWS:SourceAccount": "<ACCOUNT_ID>"
           }
         }
       }
     ]
   }
1.3 Attach Permission to Invoke Lambda
Under the Permissions tab, click Add inline policy

Choose the JSON tab, and paste:

json
Copy
Edit
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:<REGION>:<ACCOUNT_ID>:function:<FUNCTION_NAME>"
    }
  ]
}
Click Review policy, name it (e.g., InvokeLambdaPolicy), then Create policy

2. Create a Cost Budget with Actions
Go to Billing → Budgets → Create budget

Choose Cost budget, then click Next

Name your budget (e.g., EC2RDSShutdownBudget)

Choose Period (e.g., Monthly) and set a Budgeted amount (Fixed or Planned)

Click Next

2.1 Add Budget Filters (Optional)
Under Filter, you may add:

Service = EC2 and RDS

Linked account, Region, or Tags

Click Next

2.2 Configure the Budget Action
Enable “Would you like to configure budget actions?”

Select your IAM role: BudgetsInvokeLambdaRole

Click Add action trigger:

Threshold type: Actual (or Forecasted)

Threshold: e.g., 90%

Action type: Run an AWS Lambda function

Lambda function: Select your function from the dropdown

Click Next

(Optional) Add email or SNS notifications

Review your setup → Create budget

3. Confirm Lambda Permissions
The execution role for your Lambda function must be allowed to stop EC2 and RDS resources.

3.1 Confirm Permissions (Inline Policy Example)
json
Copy
Edit
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:StopInstances",
        "rds:DescribeDBInstances",
        "rds:StopDBInstance"
      ],
      "Resource": "*"
    }
  ]
}
3.2 How to Add It
Go to IAM → Roles

Select the role your Lambda function uses

Click Add inline policy → paste the JSON above → Create policy

4. Test the Budget Automation
Test the Lambda Manually

Go to Lambda → Your Function → Click Test

Use an empty test event {}

Check EC2 & RDS consoles to verify resources were stopped

View logs in CloudWatch Logs

Simulate Budget Breach

Lower your budget (e.g., to $1)

Wait for your usage to exceed the threshold (Budgets evaluates ~every 12 hours)

Revert Your Budget after confirming the shutdown worked

✅ Next Steps
Tag filtering: Modify Lambda to only stop resources with specific tags

SNS alerts: Notify teams when Lambda is triggered

Scheduled restart: Use EventBridge to power resources back up

Multi-region setup: Deploy separate Lambda functions per region or loop through regions in code

Replace all placeholder values (like <REGION>, <ACCOUNT_ID>, and <FUNCTION_NAME>) with your actual configuration before applying.

