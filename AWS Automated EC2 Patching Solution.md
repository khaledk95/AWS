# AWS Automated EC2 Patching Solution

A serverless, automated solution for patching AWS EC2 instances using AWS Step Functions, Lambda, EventBridge Scheduler, and Systems Manager.

## Overview

This CloudFormation template deploys a comprehensive solution for automating the patching of EC2 instances. The solution provides:

- Fully automated patching workflow
- Dual operation modes (install with reboot or scan-only)
- Automatic AMI creation for backup
- Detailed patch compliance reporting
- Email notifications at key points in the process
- Flexible scheduling options

## Architecture

![Architecture Diagram](architecture-diagram.png)

The solution uses the following AWS services:

- **AWS Step Functions**: Orchestrates the patching workflow
- **AWS Lambda**: Executes various functions in the patching process
- **Amazon EventBridge Scheduler**: Schedules patching operations
- **AWS Systems Manager**: Executes patching commands on EC2 instances
- **Amazon SNS**: Sends notifications about patching status
- **Amazon S3**: Stores patch compliance reports

## Prerequisites

- AWS Account with administrative access
- EC2 instances with SSM Agent installed
- Instances must have the AmazonSSMManagedInstanceCore policy attached to their instance profile

## Deployment Instructions

### Option 1: AWS Console Deployment

1. **Prepare the CloudFormation Template**
   - Download the `patching-solution-final.yaml` file from this repository

2. **Deploy via AWS Console**
   - Sign in to the AWS Management Console
   - Navigate to CloudFormation
   - Click "Create stack" > "With new resources (standard)"
   - Choose "Upload a template file" and select the downloaded YAML file
   - Click "Next"

3. **Configure Stack Parameters**
   - Stack name: Enter a name for your stack (e.g., `automated-patching-solution`)
   - EmailAddress: Enter your email address to receive patching notifications
   - AWSRegion: Select your preferred AWS region
   - Click "Next"

4. **Configure Stack Options**
   - Add any tags if needed (optional)
   - Configure any advanced options if needed (optional)
   - Click "Next"

5. **Review and Create**
   - Review your configuration
   - Check the acknowledgment for IAM resource creation
   - Click "Create stack"

6. **Confirm SNS Subscription**
   - Check your email for a subscription confirmation message
   - Click the confirmation link to receive notifications

### Option 2: AWS CLI Deployment

1. **Download the CloudFormation Template**
   ```bash
   curl -O https://raw.githubusercontent.com/yourusername/aws-patching-solution/main/patching-solution-final.yaml
   ```

2. **Deploy Using AWS CLI**
   ```bash
   aws cloudformation create-stack \
     --stack-name automated-patching-solution \
     --template-body file://patching-solution-final.yaml \
     --parameters ParameterKey=EmailAddress,ParameterValue=your-email@example.com \
                  ParameterKey=AWSRegion,ParameterValue=me-central-1 \
     --capabilities CAPABILITY_NAMED_IAM
   ```

3. **Check Deployment Status**
   ```bash
   aws cloudformation describe-stacks \
     --stack-name automated-patching-solution \
     --query "Stacks[0].StackStatus"
   ```

4. **Confirm SNS Subscription**
   - Check your email for a subscription confirmation message
   - Click the confirmation link to receive notifications

## Configuration

### Tagging EC2 Instances

The solution uses EC2 tags to identify which instances to patch:

1. **Add PatchGroup Tag to Instances**
   - Via Console: Select instances > Actions > Tags > Add/Edit Tags
   - Via CLI:
     ```bash
     aws ec2 create-tags \
       --resources i-1234567890abcdef0 \
       --tags Key=PatchGroup,Value=production-web-servers
     ```

### Configuring the Patching Schedule

By default, the EventBridge schedule is disabled. To enable and configure it:

1. **Navigate to EventBridge Scheduler**
   - Go to the Amazon EventBridge console
   - Select "Scheduler" from the left navigation
   - Find the schedule created by the CloudFormation stack

2. **Edit the Schedule**
   - Click on the schedule name
   - Click "Edit"
   - Set your desired schedule expression
   - Update the Target input with your PatchGroup value and desired PatchMode:
     ```json
     {
       "detail": {
         "PatchGroup": "YOUR_PATCH_GROUP_VALUE",
         "PatchMode": "install"
       }
     }
     ```
   - Set State to "Enabled"
   - Click "Save"

## Usage

### Triggering Patching On-Demand

You can trigger the patching process on-demand using the AWS CLI:

#### Install Mode (with reboot)

```bash
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT_ID:stateMachine:automated-patching-solution-step-function \
  --input '{"detail":{"PatchGroup":"YOUR_PATCH_GROUP_VALUE","PatchMode":"install"}}'
```

#### Scan Mode (no reboot)

```bash
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT_ID:stateMachine:automated-patching-solution-step-function \
  --input '{"detail":{"PatchGroup":"YOUR_PATCH_GROUP_VALUE","PatchMode":"scan"}}'
```

### Monitoring Patching Operations

1. **Step Functions Console**
   - Navigate to the AWS Step Functions console
   - Select the state machine to view the execution status and visual workflow

2. **Email Notifications**
   - Monitor your email for status notifications sent by the solution

3. **Patch Compliance Reports**
   - Access the patch compliance report via the pre-signed URL included in the success notification email
   - Or directly from the S3 bucket created by the solution

## Troubleshooting

### Common Issues

1. **No instances found**
   - Verify the PatchGroup tag value matches the value in the Step Function input
   - Ensure instances are in the running state

2. **Patching fails**
   - Check if SSM Agent is installed and running on target instances
   - Verify instance IAM role has AmazonSSMManagedInstanceCore policy

3. **No email notifications**
   - Confirm you've clicked the SNS subscription confirmation link
   - Check spam/junk folders

### Checking SSM Agent Status

```bash
aws ssm describe-instance-information \
  --filters "Key=tag:PatchGroup,Values=YOUR_PATCH_GROUP_VALUE"
```

### Verifying Instance Patch Compliance

```bash
aws ssm list-compliance-items \
  --resource-ids i-1234567890abcdef0 \
  --resource-types ManagedInstance \
  --filters "Key=ComplianceType,Values=Patch"
```

## Customization

### Custom Patch Baselines

1. Create a custom patch baseline in AWS Systems Manager:

   ```bash
   aws ssm create-patch-baseline \
     --name "Custom-Production-Baseline" \
     --approval-rules "PatchRules=[{PatchFilterGroup={PatchFilters=[{Key=CLASSIFICATION,Values=SecurityUpdates},{Key=SEVERITY,Values=Critical,Important}]},ApproveAfterDays=7}]" \
     --description "Custom patch baseline for production servers"
   ```

2. Register the patch baseline with your patch group:

   ```bash
   aws ssm register-patch-baseline-for-patch-group \
     --baseline-id pb-0123456789abcdef0 \
     --patch-group "production-web-servers"
   ```

## Cleanup

To remove the solution completely:

1. **Empty the S3 bucket**
   ```bash
   aws s3 rm s3://STACK_NAME-missing-patches-report --recursive
   ```

2. **Delete the CloudFormation stack**
   ```bash
   aws cloudformation delete-stack --stack-name automated-patching-solution
   ```

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Acknowledgments

- AWS Systems Manager team for the AWS-RunPatchBaseline document
- AWS Step Functions team for the serverless orchestration capabilities
