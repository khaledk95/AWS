AWS Automated EC2 Patching Solution

A serverless, automated solution for patching AWS EC2 instances using AWS Step Functions, Lambda, EventBridge Scheduler, and Systems Manager.
Overview

This CloudFormation template deploys a comprehensive solution for automating the patching of EC2 instances. The solution provides:

    Fully automated patching workflow
    Dual operation modes (install with reboot or scan-only)
    Automatic AMI creation for backup
    Detailed patch compliance reporting
    Email notifications at key points in the process
    Flexible scheduling options

Architecture

Architecture Diagram

The solution uses the following AWS services:

    AWS Step Functions: Orchestrates the patching workflow
    AWS Lambda: Executes various functions in the patching process
    Amazon EventBridge Scheduler: Schedules patching operations
    AWS Systems Manager: Executes patching commands on EC2 instances
    Amazon SNS: Sends notifications about patching status
    Amazon S3: Stores patch compliance reports

Prerequisites

    AWS Account with administrative access
    EC2 instances with SSM Agent installed
    Instances must have the AmazonSSMManagedInstanceCore policy attached to their instance profile

Deployment Instructions
Option 1: AWS Console Deployment

    Prepare the CloudFormation Template
        Download the patching-solution.yaml file from this repository

    Deploy via AWS Console
        Sign in to the AWS Management Console
        Navigate to CloudFormation
        Click "Create stack" > "With new resources (standard)"
        Choose "Upload a template file" and select the downloaded YAML file
        Click "Next"

    Configure Stack Parameters
        Stack name: Enter a name for your stack (e.g., automated-patching-solution)
        EmailAddress: Enter your email address to receive patching notifications
        AWSRegion: Select your preferred AWS region
        Click "Next"

    Configure Stack Options
        Add any tags if needed (optional)
        Configure any advanced options if needed (optional)
        Click "Next"

    Review and Create
        Review your configuration
        Check the acknowledgment for IAM resource creation
        Click "Create stack"

    Confirm SNS Subscription
        Check your email for a subscription confirmation message
        Click the confirmation link to receive notifications

Configuration
Tagging EC2 Instances

The solution uses EC2 tags to identify which instances to patch:

    Add PatchGroup Tag to Instances
        Via Console: Select instances > Actions > Tags > Add/Edit Tags
        Via CLI:

        aws ec2 create-tags \
          --resources i-1234567890abcdef0 \
          --tags Key=PatchGroup,Value=production-web-servers

Configuring the Patching Schedule

By default, the EventBridge schedule is disabled. To enable and configure it:

    Navigate to EventBridge Scheduler
        Go to the Amazon EventBridge console
        Select "Scheduler" from the left navigation
        Find the schedule created by the CloudFormation stack

    Edit the Schedule
        Click on the schedule name
        Click "Edit"
        Set your desired schedule expression
        Update the Target input with your PatchGroup value and desired PatchMode:

        {
          "detail": {
            "PatchGroup": "YOUR_PATCH_GROUP_VALUE",
            "PatchMode": "install"
          }
        }

        Set State to "Enabled"
        Click "Save"

Usage
Triggering Patching On-Demand

You can trigger the patching process on-demand using the AWS CLI:
Install Mode (with reboot)

aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT_ID:stateMachine:automated-patching-solution-step-function \
  --input '{"detail":{"PatchGroup":"YOUR_PATCH_GROUP_VALUE","PatchMode":"install"}}'

Scan Mode (no reboot)

aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT_ID:stateMachine:automated-patching-solution-step-function \
  --input '{"detail":{"PatchGroup":"YOUR_PATCH_GROUP_VALUE","PatchMode":"scan"}}'

Monitoring Patching Operations

    Step Functions Console
        Navigate to the AWS Step Functions console
        Select the state machine to view the execution status and visual workflow

    Email Notifications
        Monitor your email for status notifications sent by the solution

    Patch Compliance Reports
        Access the patch compliance report via the pre-signed URL included in the success notification email
        Or directly from the S3 bucket created by the solution

Troubleshooting
Common Issues

    No instances found
        Verify the PatchGroup tag value matches the value in the Step Function input
        Ensure instances are in the running state

    Patching fails
        Check if SSM Agent is installed and running on target instances
        Verify instance IAM role has AmazonSSMManagedInstanceCore policy

    No email notifications
        Confirm you've clicked the SNS subscription confirmation link
        Check spam/junk folders

Checking SSM Agent Status

aws ssm describe-instance-information \
  --filters "Key=tag:PatchGroup,Values=YOUR_PATCH_GROUP_VALUE"

Verifying Instance Patch Compliance

aws ssm list-compliance-items \
  --resource-ids i-1234567890abcdef0 \
  --resource-types ManagedInstance \
  --filters "Key=ComplianceType,Values=Patch"

Customization
Custom Patch Baselines

    Create a custom patch baseline in AWS Systems Manager:

    aws ssm create-patch-baseline \
      --name "Custom-Production-Baseline" \
      --approval-rules "PatchRules=[{PatchFilterGroup={PatchFilters=[{Key=CLASSIFICATION,Values=SecurityUpdates},{Key=SEVERITY,Values=Critical,Important}]},ApproveAfterDays=7}]" \
      --description "Custom patch baseline for production servers"

    Register the patch baseline with your patch group:

    aws ssm register-patch-baseline-for-patch-group \
      --baseline-id pb-0123456789abcdef0 \
      --patch-group "production-web-servers"

Cleanup

To remove the solution completely:

    Empty the S3 bucket

    aws s3 rm s3://STACK_NAME-missing-patches-report --recursive

    Delete the CloudFormation stack

    aws cloudformation delete-stack --stack-name automated-patching-solution

Contributions are welcome! Please feel free to submit a Pull Request.
Acknowledgments

    AWS Systems Manager team for the AWS-RunPatchBaseline document
    AWS Step Functions team for the serverless orchestration capabilities

