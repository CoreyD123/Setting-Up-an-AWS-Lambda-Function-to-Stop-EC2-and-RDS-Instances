# Setting Up AWS Budget with Automated ECS and RDS Shutdown

This guide explains how to create an AWS Budget that automatically stops your Amazon Elastic Container Service (ECS) tasks and Amazon Relational Database Service (RDS) instances when a specified cost threshold is exceeded.

## Prerequisites

* An AWS account with permissions to manage IAM, Lambda, Budgets, ECS, and RDS.
* A deployed AWS Lambda function configured to stop your ECS services (by setting desired task count to 0) and available RDS instances.
* The ARN (Amazon Resource Name) of your Lambda function.

## Step 1: Create an IAM Role for AWS Budgets Actions

AWS Budgets needs permission to invoke your Lambda function. You'll create an IAM role for this purpose.

1.  Navigate to the IAM service in the AWS Management Console.
2.  Click on "Roles" in the left-hand menu and then "Create role".
3.  For the "Trusted entity type", choose "AWS service".
4.  Under "Choose a use case", select "Budgets" and click "Next".
5.  Click "Next" again to skip permissions policies for now.
6.  Give your role a descriptive name (e.g., `BudgetsInvokeLambda`).
7.  Click "Create role".
8.  Select the newly created role.
9.  Go to the "Trust relationships" tab and click "Edit trust policy".
10. Replace the existing content with the following JSON, **replacing `your-aws-account-id` with your actual AWS account ID**:

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
              "AWS:SourceAccount": "your-aws-account-id"
            }
          }
        }
      ]
    }
    ```

11. Click "Save policy".
12. Go to the "Permissions policies" tab and click "Attach policies".
13. Create an inline policy (or attach an existing one) to grant this role the `lambda:InvokeFunction` permission for your specific Lambda function. Click "Add inline policy", select the "JSON" tab, and enter the following (replace placeholders):

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "lambda:InvokeFunction",
                "Resource": "arn:aws:lambda:your-aws-region:your-aws-account-id:function:your-lambda-function-name"
            }
        ]
    }
    ```

14. Click "Review policy" and give it a name (e.g., `BudgetsInvokePermission`) before clicking "Create policy".

## Step 2: Create an AWS Budget

1.  Navigate to the AWS Budgets service in the AWS Management Console.
2.  Click "Create budget".
3.  Choose "Cost budget" and click "Next".
4.  Give your budget a descriptive name (e.g., `ECSRDSShutdownBudget`).
5.  Set the "Time period" (e.g., "Monthly") and "Budgeted amount" (either "Fixed" or "Planned"). Click "Next".
6.  Under "Filter", add filters for the "Service" dimension and select "Amazon Elastic Container Service" and "Amazon Relational Database Service". Add any other desired filters (e.g., by tag or specific resource identifiers). Click "Next".
7.  Under "Would you like to configure budget actions?", select "Yes".
8.  For "IAM role for actions", choose the IAM role you created in Step 1 (e.g., `BudgetsInvokeLambda`).
9.  In "Define action triggers", click "Add action trigger".
10. Set the "Threshold type" to "Actual" and enter the percentage or absolute value of your budget that should trigger the action (e.g., 95%).
11. For "Action type", select "Run an AWS Lambda function".
12. In "Select Lambda function", choose the ARN of your Lambda function from the dropdown.
13. (Optional) Configure notifications for budget thresholds.
14. Click "Next".
15. Review your budget configuration and click "Create budget".

## Step 3: Verify Lambda Function Permissions

Ensure your Lambda function's execution role has the necessary permissions to interact with ECS and RDS to stop resources. This includes permissions like `ecs:ListClusters`, `ecs:ListServices`, `ecs:DescribeServices`, `ecs:UpdateService` (for ECS) and `rds:DescribeDBInstances`, `rds:StopDBInstance` (for RDS).

## Step 4: Testing the Budget Action

Testing the automated shutdown involves incurring costs that exceed your budget threshold. You can achieve this by:

* **Temporarily lowering your budget amount (with caution).**
* **Waiting for your normal usage to exceed the threshold.**
* **(Carefully) Generating temporary load on your ECS and RDS resources.**

Monitor your AWS Cost Explorer and the CloudWatch logs for your Lambda function to confirm that the function is invoked when the budget threshold is crossed and that your ECS tasks and RDS instances are being stopped. **Remember to revert any temporary budget or resource changes after testing.**

This guide provides a general framework for setting up budget-based automated shutdown of ECS and RDS resources. Remember to replace the placeholder ARNs and account IDs with your actual values.

