# Setting-Up-an-AWS-Lambda-Function-to-Stop-EC2-and-RDS-Instances

This guide will walk you through the necessary steps—using the AWS Management Console—to configure your AWS environment so that a Python Lambda function can automatically stop your running EC2 instances and available RDS instances when triggered.

Prerequisites
An AWS account with permissions to manage IAM, Lambda, EC2, RDS, and SNS.

(Optional) AWS CLI installed and configured for your account.

Step 1: Gather Information from Your EC2 Environment
Identify Your AWS Region

The Lambda function runs in one region and will only see resources there.

In the top‑right of the Console, verify you’re in the region that hosts your target EC2/RDS instances.

(Optional) List Running EC2 Instance IDs

bash
Copy
Edit
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text \
  --region your-region
This helps you confirm, after testing, whether instances were actually stopped.

(Optional) Note EC2 IAM Instance Profiles

In the EC2 console, select an instance, go to Details, and note the IAM role in use.

This doesn’t affect your Lambda’s permissions but is good for overall security awareness.

Step 2: Configure IAM Permissions for the Lambda Function
2.1 Create the Custom “Stop Instances” Policy
Go to IAM → Policies → Create policy.

Select the JSON tab and paste:

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
Click Next: Tags (skip), then Next: Review.

Name it LambdaStopInstancesPolicy, add a description, and Create policy.

2.2 Create (or Update) the Lambda Execution Role
Go to IAM → Roles → Create role.

Under Trusted entity, choose AWS service → Lambda → Next.

Attach policies:

AWSLambdaBasicExecutionRole (for CloudWatch Logs)

LambdaStopInstancesPolicy (the custom policy you just created)

Click Next → name the role LambdaBudgetRole → Create role.

Step 3: Deploy the Lambda Function
Go to AWS Lambda → Functions → Create function.

Choose Author from scratch:

Function name: BudgetEnforcer

Runtime: Python 3.x (e.g., 3.9 or 3.12)

Permissions: Use existing role → select LambdaBudgetRole

Click Create function.

In the Code source editor, replace any placeholder with this code:

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
    dbs = rds.describe_db_instances()['DBInstances']
    for db in dbs:
        if db['DBInstanceStatus'] == 'available':
            rds.stop_db_instance(DBInstanceIdentifier=db['DBInstanceIdentifier'])
            print(f"Stopping RDS instance: {db['DBInstanceIdentifier']}")

    return {"status": "Shutdown triggered"}
Under Configuration → General configuration:

Timeout: 30 seconds (or longer if you have many resources)

Memory: 256 MB

Click Deploy.

Step 4: Subscribe Lambda to Your SNS Topic
Go to Amazon SNS → Topics and click your topic (e.g., BudgetAlertTopic).

Under Subscriptions, click Create subscription.

Protocol: AWS Lambda

Endpoint: Select your function BudgetEnforcer by its name or ARN.

Click Create subscription.

Step 5: Grant SNS Permission to Invoke Lambda
In many cases, AWS auto‑adds this, but you can verify or add it manually:

Go to Lambda → Functions → BudgetEnforcer → Permissions.

Under Resource-based policy, ensure there’s a statement like:

json
Copy
Edit
{
  "Effect": "Allow",
  "Principal": { "Service": "sns.amazonaws.com" },
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:…:function:BudgetEnforcer",
  "Condition": {
    "ArnLike": { "AWS:SourceArn": "arn:aws:sns:…:BudgetAlertTopic" }
  }
}
Step 6: Test the End‑to‑End Flow
In SNS → Topics, click your topic → Publish message.

Enter a test subject/body and Publish.

In CloudWatch → Logs → Log groups, open /aws/lambda/BudgetEnforcer and confirm an invocation.

In the EC2 and RDS consoles (same region), verify that running EC2 instances and available RDS instances were stopped.

Important Considerations
Multi‑Region: To cover multiple regions, either iterate through regions in your code or deploy one Lambda per region.

Scheduling: Use EventBridge (CloudWatch Events) to trigger on a schedule if you prefer polling over budget alerts.

Tag Filtering: You can modify the code to only stop resources tagged e.g. AutoShutdown=true.

Error Handling: Add try/except blocks or CloudWatch Alarms for Lambda failures.

