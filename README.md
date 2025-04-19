Automated Shutdown of EC2 & RDS on AWS Budget Breach
Below is a complete, end‑to‑end guide—using the AWS Management Console—to automatically stop your running EC2 instances and available RDS instances when your monthly cost exceeds a defined budget. Follow each step carefully; no prior Lambda experience is required.

Prerequisites
AWS account with permissions to manage IAM, Lambda, EC2, RDS, SNS, and Budgets.

(Optional) AWS CLI installed & configured for verification.

Step 1: Gather Region & Instance Info
Select Your AWS Region

In the top‑right of the Console, note the current region. Ensure it matches where your EC2/RDS resources live.

(Optional) List Running EC2 Instance IDs

bash
Copy
Edit
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text \
  --region your-region
Use this later to confirm resources were stopped.

Step 2: Create IAM Role & Policy for Lambda
2.1 Create the “Stop Instances” Policy
IAM → Policies → Create policy

JSON tab → paste:

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
        "ec2:StopInstances"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:StopDBInstance"
      ],
      "Resource": "*"
    }
  ]
}
Next: Review →

Name: LambdaStopInstancesPolicy

Description: Allows Lambda to describe & stop EC2/RDS.

Create policy.

2.2 Create the Lambda Execution Role
IAM → Roles → Create role

Trusted entity: AWS service → Use case: Lambda → Next

Attach policies:

AWSLambdaBasicExecutionRole (for CloudWatch Logs)

LambdaStopInstancesPolicy (custom)

Next → Role name: LambdaBudgetRole → Create role

Step 3: Deploy the Lambda Function
Lambda → Functions → Create function

Author from scratch:

Name: stop-ec2-rds

Runtime: Python 3.x

Permissions: Use existing role → LambdaBudgetRole

Create function.

In Code source, replace code with:

python
Copy
Edit
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    rds = boto3.client('rds')

    # Stop running EC2 instances
    resp = ec2.describe_instances(Filters=[
        {'Name': 'instance-state-name', 'Values': ['running']}
    ])
    ids = [inst['InstanceId']
           for r in resp['Reservations']
           for inst in r['Instances']]
    if ids:
        ec2.stop_instances(InstanceIds=ids)
        print(f"Stopping EC2 instances: {ids}")
    else:
        print("No running EC2 instances found.")

    # Stop available RDS instances
    for db in rds.describe_db_instances()['DBInstances']:
        if db['DBInstanceStatus'] == 'available':
            rds.stop_db_instance(DBInstanceIdentifier=db['DBInstanceIdentifier'])
            print(f"Stopping RDS instance: {db['DBInstanceIdentifier']}")

    return {"status": "Shutdown triggered"}
Configuration → General configuration:

Timeout: 30 s

Memory: 256 MB

Deploy.

Step 4: (Optional) Create SNS Topic for Notifications
If you want email alerts when the budget triggers:

SNS → Topics → Create topic

Type: Standard

Name: BudgetNotifications

Create subscription:

Protocol: Email

Endpoint: your email address

Check your email and Confirm subscription.

Step 5: Create IAM Role for Budget Actions
IAM → Roles → Create role

Trusted entity: AWS service → Use case: Budgets → Next

Attach inline policy (click Create policy if needed):

json
Copy
Edit
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:YOUR-REGION:YOUR-ACCOUNT-ID:function:stop-ec2-rds"
    }
  ]
}
Next → Role name: BudgetsInvokeLambdaRole → Create role

Step 6: Create & Configure Your AWS Budget
Billing & Cost Management → Budgets → Create budget

Budget type: Cost budget → Set budget details:

Period: Monthly

Amount: e.g., $100

Scope (optional): Filter by service (EC2, RDS), region, or tags

Configure actions (under Actions & notifications):

Enable actions: On

IAM role for actions: BudgetsInvokeLambdaRole

Trigger: Actual or Forecasted ≥ 100% (or set at 90%)

Action type: Run an AWS Lambda function

Select function: stop-ec2-rds

(Optional) Notifications:

Email recipients: add stakeholder emails

SNS topic: BudgetNotifications (if created)

Thresholds: match your action triggers

Confirm and create budget.

Step 7: Verify Permissions for Budget to Invoke Lambda
AWS usually adds this automatically. To check:

Lambda → stop-ec2-rds → Permissions

Under Resource-based policy, you should see a statement like:

json
Copy
Edit
{
  "Sid": "AllowBudgetsInvocation",
  "Effect": "Allow",
  "Principal": { "Service": "budgets.amazonaws.com" },
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:YOUR-REGION:YOUR-ACCOUNT-ID:function:stop-ec2-rds",
  "Condition": {
    "ArnLike": { "AWS:SourceArn": "arn:aws:budgets::YOUR-ACCOUNT-ID:budget/*" }
  }
}
If missing, run:

bash
Copy
Edit
aws lambda add-permission \
  --function-name stop-ec2-rds \
  --statement-id AllowBudgetsInvocation \
  --principal budgets.amazonaws.com \
  --action lambda:InvokeFunction \
  --source-arn arn:aws:budgets::YOUR-ACCOUNT-ID:budget/*
Step 8: Test Everything
Test Lambda in isolation:

Lambda → stop-ec2-rds → Test with an empty event {}.

Check CloudWatch Logs for a successful invocation.

Test SNS notification (if used):

SNS → Topics → BudgetNotifications → Publish message.

Confirm your email receives the alert.

Simulate Budget Breach (caution!):

Temporarily lower your budget amount to below your current spend.

Wait up to 24 hrs for AWS Budgets to evaluate and trigger the action.

Confirm Resource Shutdown:

EC2 Console: running instances should now be stopped.

RDS Console: available databases should now be stopped.

Check Lambda logs for the list of stopped resources.

Revert Budget (if you lowered it for testing).

Additional Tips
Multi‑Region: Deploy one Lambda per region or loop through regions in code.

Tag‑Based Control: Modify the code to filter by tags (e.g., AutoShutdown=true).

Error Handling: Add try/except blocks and send failure alerts via SNS.

Scheduling Alternative: Use EventBridge to run the Lambda on a schedule instead of Budgets.


