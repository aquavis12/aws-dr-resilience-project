# AWS Disaster Recovery & Resilience — Hands-On Lab Guide

> **Author:** Venkata Pavan Vishnu Rachapudi | **Date:** July 2026  
> **Scope:** AWS Resilience Hub | Amazon ARC | AWS DRS | AWS Backup | 4 DR Strategies  
> **Format:** Each lab has a **Simple Setup** (learning) and **Complex Setup** (production-grade)

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Lab 1: AWS Backup — Backup & Restore Strategy](#lab-1-aws-backup--backup--restore-strategy)
3. [Lab 2: AWS Elastic Disaster Recovery (DRS) — Pilot Light](#lab-2-aws-elastic-disaster-recovery-drs--pilot-light)
4. [Lab 3: Amazon Application Recovery Controller (ARC) — Warm Standby & Active-Active](#lab-3-amazon-application-recovery-controller-arc--warm-standby--active-active)
5. [Lab 4: AWS Resilience Hub — Assessment & Governance](#lab-4-aws-resilience-hub--assessment--governance)
6. [Lab 5: End-to-End DR Architecture (Combining All Services)](#lab-5-end-to-end-dr-architecture-combining-all-services)
7. [Lab 6: AWS Service Screener v2 — Resilience & Best Practice Assessment](#lab-6-aws-service-screener-v2--resilience--best-practice-assessment)
8. [Cleanup Instructions](#cleanup-instructions)
9. [Cost Estimation Table](#cost-estimation-table)
10. [References & Further Reading](#references--further-reading)

---

## Prerequisites

### AWS Account Requirements

| Requirement | Simple Labs | Complex Labs |
|-------------|-------------|--------------|
| AWS Accounts | 1 account | 2–3 accounts (Management + DR + Shared Services) |
| Regions | 1 (us-east-1) | 2 (us-east-1 + us-west-2) |
| AWS Organizations | Not required | Required (for cross-account) |
| Budget Alert | Recommended ($50) | Required ($200) |

### CLI & Tooling Setup

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Verify
aws --version
# aws-cli/2.x.x Python/3.x.x ...

# Configure default profile
aws configure
# AWS Access Key ID: <YOUR_KEY>
# AWS Secret Access Key: <YOUR_SECRET>
# Default region name: us-east-1
# Default output format: json

# Configure DR region profile
aws configure --profile dr-region
# Default region name: us-west-2

# Install Session Manager Plugin (for DRS agent)
# https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

# Install jq for JSON parsing
sudo apt-get install jq -y  # Linux
brew install jq              # macOS
```

### IAM Permissions Required

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DRLabsFullAccess",
      "Effect": "Allow",
      "Action": [
        "backup:*",
        "backup-storage:*",
        "drs:*",
        "arc-zonal-shift:*",
        "route53-recovery-cluster:*",
        "route53-recovery-control-config:*",
        "route53-recovery-readiness:*",
        "resiliencehub:*",
        "ec2:*",
        "rds:*",
        "dynamodb:*",
        "route53:*",
        "elasticloadbalancing:*",
        "autoscaling:*",
        "cloudformation:*",
        "cloudwatch:*",
        "sns:*",
        "lambda:*",
        "states:*",
        "iam:PassRole",
        "iam:CreateServiceLinkedRole",
        "s3:*",
        "fis:*",
        "ssm:*",
        "organizations:*",
        "ram:*"
      ],
      "Resource": "*"
    }
  ]
}
```

> ⚠️ **Cost Warning:** These labs create real AWS resources that incur charges. Always run the cleanup steps when done. Estimated costs are listed per lab. Set up AWS Budgets alerts before starting.

```bash
# Create a budget alert (recommended)
aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{
    "BudgetName": "DR-Labs-Budget",
    "BudgetLimit": {"Amount": "100", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "your-email@example.com"}]
  }]'
```

### Common Variables (Set These First)

```bash
# Set these environment variables for all labs
export PRIMARY_REGION="us-east-1"
export DR_REGION="us-west-2"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export VPC_ID="vpc-xxxxxxxx"          # Your default VPC or custom VPC
export SUBNET_ID="subnet-xxxxxxxx"    # A subnet in your VPC
export KEY_PAIR="my-dr-lab-key"       # Your EC2 key pair name
export PROJECT_TAG="dr-lab"

# Create a key pair if you don't have one
aws ec2 create-key-pair --key-name $KEY_PAIR --query 'KeyMaterial' --output text > ~/.ssh/$KEY_PAIR.pem
chmod 400 ~/.ssh/$KEY_PAIR.pem
```

---

## Lab 1: AWS Backup — Backup & Restore Strategy

> **DR Strategy:** Backup & Restore  
> **RTO:** Hours | **RPO:** Hours (1–24h depending on backup frequency)  
> **Cost Profile:** $ (Lowest)

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BACKUP & RESTORE                              │
├─────────────────────────┬───────────────────────────────────────────┤
│   PRIMARY REGION        │          DR REGION                         │
│   (us-east-1)           │          (us-west-2)                       │
│                         │                                            │
│  ┌──────────┐           │                                            │
│  │   EC2    │──backup──▶│  ┌─────────────────┐                       │
│  │ Instance │           │  │  Backup Vault   │                       │
│  └──────────┘           │  │  (Cross-Region  │                       │
│  ┌──────────┐           │  │   Copy)         │                       │
│  │   RDS    │──backup──▶│  └────────┬────────┘                       │
│  │ Instance │           │           │                                │
│  └──────────┘           │           ▼ (On Disaster)                  │
│  ┌──────────┐           │  ┌─────────────────┐                       │
│  │   EBS    │──backup──▶│  │  Restored EC2   │                       │
│  │ Volumes  │           │  │  Restored RDS   │                       │
│  └──────────┘           │  └─────────────────┘                       │
│                         │                                            │
│  AWS Backup Plan        │  Backup Vault (DR)                         │
│  ├── Daily Schedule     │  ├── Cross-Region Copies                   │
│  ├── 7-day Retention    │  ├── Vault Lock (WORM)                     │
│  └── Lifecycle Rules    │  └── Air-Gapped (Complex)                  │
└─────────────────────────┴───────────────────────────────────────────┘
```

---

### 🟢 Simple Setup

> **Estimated Time:** 30–45 minutes  
> **Estimated Cost:** ~$2–5 (if cleaned up within 2 hours)

#### Step 1: Create a Test EC2 Instance

```bash
# Launch a simple Amazon Linux 2023 instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro \
  --key-name $KEY_PAIR \
  --subnet-id $SUBNET_ID \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=backup-lab-source},{Key=Project,Value=$PROJECT_TAG},{Key=BackupPlan,Value=daily}]" \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance ID: $INSTANCE_ID"

# Wait for instance to be running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is running!"
```

#### Step 2: Create a Backup Vault

```bash
# Create a backup vault (encrypted with default AWS managed key)
aws backup create-backup-vault \
  --backup-vault-name "dr-lab-vault" \
  --backup-vault-tags "Project=$PROJECT_TAG"

# Verify vault creation
aws backup describe-backup-vault --backup-vault-name "dr-lab-vault"
```

**Expected Output:**
```json
{
    "BackupVaultName": "dr-lab-vault",
    "BackupVaultArn": "arn:aws:backup:us-east-1:123456789012:backup-vault:dr-lab-vault",
    "EncryptionKeyArn": "arn:aws:kms:us-east-1:123456789012:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "NumberOfRecoveryPoints": 0,
    ...
}
```

#### Step 3: Create a Backup Plan

```bash
# Create a backup plan with daily schedule and 7-day retention
PLAN_ID=$(aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "DailyBackupPlan",
    "Rules": [
      {
        "RuleName": "DailyBackup",
        "TargetBackupVaultName": "dr-lab-vault",
        "ScheduleExpression": "cron(0 3 * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {
          "DeleteAfterDays": 7
        }
      }
    ]
  }' \
  --query 'BackupPlanId' \
  --output text)

echo "Backup Plan ID: $PLAN_ID"
```

#### Step 4: Assign Resources to the Backup Plan (Tag-Based)

```bash
# Create a resource selection using tags
aws backup create-backup-selection \
  --backup-plan-id $PLAN_ID \
  --backup-selection '{
    "SelectionName": "TagBasedSelection",
    "IamRoleArn": "arn:aws:iam::'$ACCOUNT_ID':role/service-role/AWSBackupDefaultServiceRole",
    "ListOfTags": [
      {
        "ConditionType": "STRINGEQUALS",
        "ConditionKey": "BackupPlan",
        "ConditionValue": "daily"
      }
    ]
  }'
```

> **Note:** If the `AWSBackupDefaultServiceRole` doesn't exist, create it:
> ```bash
> # AWS Backup creates this automatically when you use the Console.
> # For CLI, create it manually:
> aws iam create-role --role-name AWSBackupDefaultServiceRole \
>   --assume-role-policy-document '{
>     "Version": "2012-10-17",
>     "Statement": [{"Effect": "Allow", "Principal": {"Service": "backup.amazonaws.com"}, "Action": "sts:AssumeRole"}]
>   }'
> aws iam attach-role-policy --role-name AWSBackupDefaultServiceRole \
>   --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup
> aws iam attach-role-policy --role-name AWSBackupDefaultServiceRole \
>   --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores
> ```

#### Step 5: Perform an On-Demand Backup

```bash
# Start an on-demand backup (don't wait for the scheduled time)
BACKUP_JOB_ID=$(aws backup start-backup-job \
  --backup-vault-name "dr-lab-vault" \
  --resource-arn "arn:aws:ec2:${PRIMARY_REGION}:${ACCOUNT_ID}:instance/${INSTANCE_ID}" \
  --iam-role-arn "arn:aws:iam::${ACCOUNT_ID}:role/service-role/AWSBackupDefaultServiceRole" \
  --query 'BackupJobId' \
  --output text)

echo "Backup Job ID: $BACKUP_JOB_ID"

# Monitor backup progress
watch -n 10 "aws backup describe-backup-job --backup-job-id $BACKUP_JOB_ID --query '{Status: Status, PercentDone: PercentDone, BackupSizeInBytes: BackupSizeInBytes}'"
```

**Expected Output (after ~5-10 minutes):**
```json
{
    "Status": "COMPLETED",
    "PercentDone": "100.0",
    "BackupSizeInBytes": 8589934592
}
```

#### Step 6: List Recovery Points

```bash
# List all recovery points in the vault
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "dr-lab-vault" \
  --query 'RecoveryPoints[*].{ARN:RecoveryPointArn,Created:CreationDate,Status:Status,Size:BackupSizeInBytes}' \
  --output table
```

#### Step 7: Restore from Backup

```bash
# Get the recovery point ARN
RECOVERY_POINT_ARN=$(aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "dr-lab-vault" \
  --query 'RecoveryPoints[0].RecoveryPointArn' \
  --output text)

echo "Restoring from: $RECOVERY_POINT_ARN"

# Get the metadata from the recovery point
aws backup get-recovery-point-restore-metadata \
  --backup-vault-name "dr-lab-vault" \
  --recovery-point-arn "$RECOVERY_POINT_ARN" \
  --query 'RestoreMetadata' > /tmp/restore-metadata.json

# Start restore job
RESTORE_JOB_ID=$(aws backup start-restore-job \
  --recovery-point-arn "$RECOVERY_POINT_ARN" \
  --iam-role-arn "arn:aws:iam::${ACCOUNT_ID}:role/service-role/AWSBackupDefaultServiceRole" \
  --metadata file:///tmp/restore-metadata.json \
  --query 'RestoreJobId' \
  --output text)

echo "Restore Job ID: $RESTORE_JOB_ID"

# Monitor restore progress
aws backup describe-restore-job --restore-job-id $RESTORE_JOB_ID
```

#### Step 8: Verify the Restored Instance

```bash
# Wait for restore to complete (typically 5-15 minutes)
aws backup describe-restore-job --restore-job-id $RESTORE_JOB_ID \
  --query '{Status: Status, CreatedResourceArn: CreatedResourceArn}'

# Once COMPLETED, get the new instance ID
RESTORED_INSTANCE=$(aws backup describe-restore-job --restore-job-id $RESTORE_JOB_ID \
  --query 'CreatedResourceArn' --output text | awk -F'/' '{print $NF}')

echo "Restored Instance: $RESTORED_INSTANCE"

# Verify it's running
aws ec2 describe-instances --instance-ids $RESTORED_INSTANCE \
  --query 'Reservations[0].Instances[0].{State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone}'
```

#### Verification Checklist

- [ ] Backup vault created successfully
- [ ] Backup plan with daily schedule configured
- [ ] On-demand backup completed (Status: COMPLETED)
- [ ] Recovery point visible in vault
- [ ] Restore job completed successfully
- [ ] Restored instance is running and accessible

#### Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| "Access Denied" on backup job | Missing IAM role | Create AWSBackupDefaultServiceRole with required policies |
| Backup takes very long | Large EBS volumes | Normal — AMI-based backups of large volumes take time |
| Restore fails with "InsufficientCapacity" | AZ capacity issue | Modify restore metadata to use a different subnet/AZ |
| Backup status stuck at "CREATED" | Start window too narrow | Increase StartWindowMinutes to 120 |

---

### 🔴 Complex Setup

> **Estimated Time:** 2–3 hours  
> **Estimated Cost:** ~$15–25 (if cleaned up same day)

#### Architecture: Multi-Account Cross-Region Backup

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                   AWS ORGANIZATIONS (Management Account)                       │
├──────────────────────────────┬───────────────────────────────────────────────┤
│  PRODUCTION ACCOUNT          │  DR ACCOUNT (Cross-Account)                    │
│  (us-east-1)                 │  (us-west-2)                                  │
│                              │                                                │
│  ┌─────────┐ ┌─────────┐    │  ┌──────────────────────────┐                  │
│  │  EC2s   │ │  RDS     │    │  │  Logically Air-Gapped    │                  │
│  │ (tagged)│ │(tagged)  │    │  │  Backup Vault            │                  │
│  └────┬────┘ └────┬────┘    │  │  ┌────────────────────┐  │                  │
│       │            │         │  │  │ Vault Lock (WORM)  │  │                  │
│       ▼            ▼         │  │  │ - Min 365 days     │  │                  │
│  ┌─────────────────────┐    │  │  │ - Immutable         │  │                  │
│  │  Local Backup Vault │    │  │  └────────────────────┘  │                  │
│  │  (Warm Storage)     │────cross-region-copy────▶        │                  │
│  └─────────────────────┘    │  └──────────────────────────┘                  │
│       │                      │           │                                    │
│       │ (after 30 days)      │           │ (Audit Manager)                    │
│       ▼                      │           ▼                                    │
│  ┌─────────────────────┐    │  ┌──────────────────────────┐                  │
│  │  Cold Storage Tier  │    │  │  Compliance Framework    │                  │
│  │  (90 day min)       │    │  │  - Backup frequency ✓   │                  │
│  └─────────────────────┘    │  │  - Cross-region copy ✓  │                  │
│                              │  │  - Encryption ✓         │                  │
│  Backup Audit Manager        │  │  - Retention policy ✓   │                  │
│  ├── BACKUP_PLAN_MIN_FREQ    │  └──────────────────────────┘                  │
│  ├── BACKUP_RECOVERY_POINT   │                                                │
│  └── BACKUP_RESOURCES_PROTECT│                                                │
└──────────────────────────────┴───────────────────────────────────────────────┘
```

#### Step 1: Create Organizational Backup Policy

```bash
# Enable AWS Backup in AWS Organizations (from Management Account)
aws organizations enable-aws-service-access \
  --service-principal backup.amazonaws.com

# Register delegated administrator (optional — use a central backup account)
aws organizations register-delegated-administrator \
  --account-id <BACKUP_ADMIN_ACCOUNT_ID> \
  --service-principal backup.amazonaws.com
```

#### Step 2: Create Cross-Region Backup Plan with Lifecycle

```bash
# Create a production-grade backup plan
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "ProductionDRPlan",
    "Rules": [
      {
        "RuleName": "HourlyBackup",
        "TargetBackupVaultName": "production-vault",
        "ScheduleExpression": "cron(0 * * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 120,
        "Lifecycle": {
          "MoveToColdStorageAfterDays": 30,
          "DeleteAfterDays": 365
        },
        "CopyActions": [
          {
            "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:'$ACCOUNT_ID':backup-vault:dr-vault",
            "Lifecycle": {
              "MoveToColdStorageAfterDays": 30,
              "DeleteAfterDays": 365
            }
          }
        ]
      },
      {
        "RuleName": "DailyBackup",
        "TargetBackupVaultName": "production-vault",
        "ScheduleExpression": "cron(0 3 * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {
          "MoveToColdStorageAfterDays": 90,
          "DeleteAfterDays": 2555
        },
        "CopyActions": [
          {
            "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:'$ACCOUNT_ID':backup-vault:dr-vault",
            "Lifecycle": {
              "MoveToColdStorageAfterDays": 90,
              "DeleteAfterDays": 2555
            }
          }
        ]
      }
    ]
  }'
```

#### Step 3: Create DR Vault in Secondary Region

```bash
# Create vault in DR region
aws backup create-backup-vault \
  --backup-vault-name "dr-vault" \
  --region $DR_REGION \
  --backup-vault-tags "Project=$PROJECT_TAG,Environment=DR"
```

#### Step 4: Configure Vault Lock (WORM Compliance)

```bash
# Apply Vault Lock — GOVERNANCE MODE (can be removed by admin)
aws backup put-backup-vault-lock-configuration \
  --backup-vault-name "dr-vault" \
  --region $DR_REGION \
  --min-retention-days 365 \
  --max-retention-days 2555 \
  --changeable-for-days 3

# ⚠️ WARNING: After changeable-for-days expires, this becomes COMPLIANCE mode
# and CANNOT be removed, even by AWS root account!

# Verify vault lock
aws backup describe-backup-vault \
  --backup-vault-name "dr-vault" \
  --region $DR_REGION \
  --query '{Locked:Locked,MinRetention:MinRetentionDays,MaxRetention:MaxRetentionDays}'
```

**Expected Output:**
```json
{
    "Locked": true,
    "MinRetention": 365,
    "MaxRetention": 2555
}
```

#### Step 5: Create Logically Air-Gapped Vault

```bash
# Create a logically air-gapped vault (isolated from production account)
aws backup create-logically-air-gapped-backup-vault \
  --backup-vault-name "air-gapped-dr-vault" \
  --region $DR_REGION \
  --min-retention-days 7 \
  --max-retention-days 365 \
  --creator-request-id "dr-lab-$(date +%s)"

# Share via AWS RAM for cross-account access
aws ram create-resource-share \
  --name "DR-Vault-Share" \
  --resource-arns "arn:aws:backup:${DR_REGION}:${ACCOUNT_ID}:backup-vault:air-gapped-dr-vault" \
  --principals "<DR_ACCOUNT_ID>" \
  --region $DR_REGION
```

#### Step 6: Configure Backup Audit Manager

```bash
# Create an audit framework with compliance controls
aws backup create-framework \
  --framework-name "DR-Compliance-Framework" \
  --framework-controls '[
    {
      "ControlName": "BACKUP_PLAN_MIN_FREQUENCY_AND_MIN_RETENTION_CHECK",
      "ControlInputParameters": [
        {"ParameterName": "requiredFrequencyUnit", "ParameterValue": "hours"},
        {"ParameterName": "requiredFrequencyValue", "ParameterValue": "24"},
        {"ParameterName": "requiredRetentionDays", "ParameterValue": "35"}
      ]
    },
    {
      "ControlName": "BACKUP_RECOVERY_POINT_ENCRYPTED"
    },
    {
      "ControlName": "BACKUP_RECOVERY_POINT_MANUAL_DELETION_DISABLED"
    },
    {
      "ControlName": "BACKUP_RESOURCES_PROTECTED_BY_CROSS_REGION"
    },
    {
      "ControlName": "BACKUP_RESOURCES_PROTECTED_BY_CROSS_ACCOUNT"
    }
  ]'

# Create a report plan for compliance auditing
aws backup create-report-plan \
  --report-plan-name "DR-Compliance-Report" \
  --report-delivery-channel '{
    "S3BucketName": "my-backup-audit-reports-'$ACCOUNT_ID'",
    "Formats": ["CSV", "JSON"]
  }' \
  --report-setting '{
    "ReportTemplate": "BACKUP_JOB_REPORT",
    "FrameworkArns": ["arn:aws:backup:'$PRIMARY_REGION':'$ACCOUNT_ID':framework:DR-Compliance-Framework"]
  }'
```

#### Step 7: Automated Restore Testing

```bash
# Create a restore testing plan (validates backups are recoverable)
aws backup create-restore-testing-plan \
  --restore-testing-plan '{
    "RestoreTestingPlanName": "WeeklyRestoreTest",
    "ScheduleExpression": "cron(0 8 ? * MON *)",
    "StartWindowHours": 2,
    "RecoveryPointSelection": {
      "Algorithm": "LATEST_WITHIN_WINDOW",
      "RecoveryPointTypes": ["CONTINUOUS", "SNAPSHOT"],
      "IncludeVaults": ["arn:aws:backup:'$PRIMARY_REGION':'$ACCOUNT_ID':backup-vault:production-vault"],
      "SelectionWindowDays": 7
    }
  }'

# Add a resource selection for restore testing
aws backup create-restore-testing-selection \
  --restore-testing-plan-name "WeeklyRestoreTest" \
  --restore-testing-selection '{
    "RestoreTestingSelectionName": "EC2RestoreTest",
    "ProtectedResourceType": "EC2",
    "IamRoleArn": "arn:aws:iam::'$ACCOUNT_ID':role/service-role/AWSBackupDefaultServiceRole",
    "ProtectedResourceConditions": {
      "StringEquals": [
        {"Key": "aws:ResourceTag/BackupPlan", "Value": "daily"}
      ]
    },
    "ValidationWindowHours": 1
  }'
```

#### Step 8: CloudFormation Template (Infrastructure as Code)

```yaml
# File: backup-infra.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Production-Grade AWS Backup Infrastructure

Parameters:
  DRRegion:
    Type: String
    Default: us-west-2
  RetentionDays:
    Type: Number
    Default: 365

Resources:
  ProductionVault:
    Type: AWS::Backup::BackupVault
    Properties:
      BackupVaultName: production-vault
      BackupVaultTags:
        Project: dr-lab
        Environment: production

  DRVault:
    Type: AWS::Backup::BackupVault
    Properties:
      BackupVaultName: dr-vault-cfn
      BackupVaultTags:
        Project: dr-lab
        Environment: dr

  ProductionBackupPlan:
    Type: AWS::Backup::BackupPlan
    Properties:
      BackupPlan:
        BackupPlanName: ProductionPlan
        BackupPlanRule:
          - RuleName: HourlyRule
            TargetBackupVault: !Ref ProductionVault
            ScheduleExpression: "cron(0 * * * ? *)"
            StartWindowMinutes: 60
            CompletionWindowMinutes: 120
            Lifecycle:
              MoveToColdStorageAfterDays: 30
              DeleteAfterDays: !Ref RetentionDays
            CopyActions:
              - DestinationBackupVaultArn: !GetAtt DRVault.BackupVaultArn
                Lifecycle:
                  MoveToColdStorageAfterDays: 30
                  DeleteAfterDays: !Ref RetentionDays

  TagBasedSelection:
    Type: AWS::Backup::BackupSelection
    Properties:
      BackupPlanId: !Ref ProductionBackupPlan
      BackupSelection:
        SelectionName: ProductionResources
        IamRoleArn: !Sub "arn:aws:iam::${AWS::AccountId}:role/service-role/AWSBackupDefaultServiceRole"
        ListOfTags:
          - ConditionType: STRINGEQUALS
            ConditionKey: BackupPlan
            ConditionValue: production

Outputs:
  ProductionVaultArn:
    Value: !GetAtt ProductionVault.BackupVaultArn
  BackupPlanId:
    Value: !Ref ProductionBackupPlan
```

```bash
# Deploy the CloudFormation stack
aws cloudformation deploy \
  --template-file backup-infra.yaml \
  --stack-name dr-lab-backup-infra \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides DRRegion=$DR_REGION RetentionDays=365
```

#### Verification (Complex)

- [ ] Cross-region backup copy completing in us-west-2
- [ ] Vault Lock active and counting down changeable period
- [ ] Air-gapped vault accessible from DR account
- [ ] Audit Manager framework showing compliance status
- [ ] Restore testing plan executing weekly
- [ ] CloudFormation stack deployed successfully

---

## Lab 2: AWS Elastic Disaster Recovery (DRS) — Pilot Light

> **DR Strategy:** Pilot Light / Warm Standby  
> **RTO:** 5–20 minutes | **RPO:** Sub-second (continuous replication)  
> **Cost Profile:** $$ (replication) + $ (on-demand recovery)

### Architecture Overview

```
┌───────────────────────────────────────────────────────────────────────────┐
│                     AWS ELASTIC DISASTER RECOVERY                          │
├────────────────────────────────┬──────────────────────────────────────────┤
│   SOURCE REGION (us-east-1)    │   TARGET/DR REGION (us-west-2)           │
│                                │                                          │
│  ┌──────────────────────┐      │   ┌────────────────────────────────┐     │
│  │   Source Server      │      │   │   STAGING AREA (Low Cost)      │     │
│  │   ┌──────────────┐   │      │   │                                │     │
│  │   │ DRS Agent    │───continuous──│   ┌──────────────────────┐    │     │
│  │   │ (block-level)│   replication │   │  Replication Server  │    │     │
│  │   └──────────────┘   │      │   │  │  (t3.small)           │    │     │
│  │   OS + Apps + Data    │      │   │  └──────────┬───────────┘    │     │
│  └──────────────────────┘      │   │             │                 │     │
│                                │   │  ┌──────────▼───────────┐    │     │
│  ┌──────────────────────┐      │   │  │  Staging EBS Disks   │    │     │
│  │   Source Server 2    │      │   │  │  (block-level copy)   │    │     │
│  │   (Web Tier)         │──────────│  └──────────┬───────────┘    │     │
│  └──────────────────────┘      │   │             │                 │     │
│                                │   │  ┌──────────▼───────────┐    │     │
│  ┌──────────────────────┐      │   │  │  PIT Snapshots       │    │     │
│  │   Source Server 3    │      │   │  │  (point-in-time)      │    │     │
│  │   (App Tier)         │──────────│  └──────────────────────┘    │     │
│  └──────────────────────┘      │   └───────────────┬──────────────┘     │
│                                │                   │                      │
│                                │         (On Failover/Drill)              │
│                                │                   ▼                      │
│                                │   ┌────────────────────────────────┐     │
│                                │   │   RECOVERY INSTANCES           │     │
│                                │   │   (Full Production Capacity)   │     │
│                                │   │   ┌─────┐ ┌─────┐ ┌─────┐    │     │
│                                │   │   │Web  │ │App  │ │DB   │    │     │
│                                │   │   │Tier │ │Tier │ │Tier │    │     │
│                                │   │   └─────┘ └─────┘ └─────┘    │     │
│                                │   └────────────────────────────────┘     │
└────────────────────────────────┴──────────────────────────────────────────┘
```

---

### 🟢 Simple Setup

> **Estimated Time:** 45–60 minutes  
> **Estimated Cost:** ~$5–10/day while replicating (staging area costs)

#### Step 1: Initialize DRS in the Target Region

```bash
# Initialize AWS DRS in the DR region
aws drs initialize-service --region $DR_REGION

# Verify initialization
aws drs describe-replication-configuration-templates \
  --region $DR_REGION \
  --query 'Items[0].ReplicationConfigurationTemplateID'
```

#### Step 2: Create a Source EC2 Instance (to protect)

```bash
# Launch a source instance with a simple web application
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.small \
  --key-name $KEY_PAIR \
  --subnet-id $SUBNET_ID \
  --iam-instance-profile Name=AmazonSSMManagedInstanceCore \
  --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Primary Region - DR Lab Source Server</h1><p>Instance: $(hostname)</p>" > /var/www/html/index.html
' \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=drs-source-server},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Source Instance: $INSTANCE_ID"
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
```

#### Step 3: Install the DRS Replication Agent

```bash
# Connect to the source instance via SSM
aws ssm start-session --target $INSTANCE_ID

# Inside the instance — install the DRS agent
sudo su -
wget -O ./aws-replication-installer-init https://aws-elastic-disaster-recovery-${DR_REGION}.s3.${DR_REGION}.amazonaws.com/latest/linux/aws-replication-installer-init
chmod +x aws-replication-installer-init

# Run the installer (provide credentials with DRS permissions)
./aws-replication-installer-init \
  --region $DR_REGION \
  --aws-access-key-id <YOUR_ACCESS_KEY> \
  --aws-secret-access-key <YOUR_SECRET_KEY> \
  --no-prompt

# Exit SSM session
exit
exit
```

> **Alternative: Use IAM Role (Recommended)**
> Attach the `AWSElasticDisasterRecoveryAgentInstallationPolicy` to the instance role instead of using access keys.

#### Step 4: Monitor Replication Status

```bash
# List source servers in DRS
aws drs describe-source-servers \
  --region $DR_REGION \
  --query 'Items[*].{ServerID:SourceServerID,Hostname:SourceProperties.IdentificationHints.Hostname,State:DataReplicationInfo.DataReplicationState,Lag:DataReplicationInfo.ReplicatedDisks[0].ReplicatedStorageBytes}' \
  --output table

# Wait for replication to reach "CONTINUOUS" state
# This takes 15-60 minutes depending on disk size
watch -n 30 "aws drs describe-source-servers --region $DR_REGION \
  --query 'Items[0].DataReplicationInfo.DataReplicationState' --output text"
```

**Expected States (in order):**
1. `INITIATING` — Agent registered, setting up
2. `INITIAL_SYNC` — First full copy in progress
3. `CONTINUOUS` — ✅ Healthy, continuously replicating changes

#### Step 5: Perform a DR Drill (Test Recovery)

```bash
# Get the source server ID
SOURCE_SERVER_ID=$(aws drs describe-source-servers \
  --region $DR_REGION \
  --query 'Items[0].SourceServerID' \
  --output text)

echo "Source Server ID: $SOURCE_SERVER_ID"

# Launch a drill (test recovery without affecting replication)
DRILL_JOB_ID=$(aws drs start-recovery \
  --region $DR_REGION \
  --is-drill \
  --source-servers "[{\"sourceServerID\": \"$SOURCE_SERVER_ID\"}]" \
  --query 'Job.JobID' \
  --output text)

echo "Drill Job ID: $DRILL_JOB_ID"

# Monitor drill progress
aws drs describe-jobs \
  --region $DR_REGION \
  --filters "jobIDs=$DRILL_JOB_ID" \
  --query 'Items[0].{Status:Status,Initiated:InitiationTime,Participants:ParticipatingServers[*].LaunchStatus}'
```

#### Step 6: Verify Recovery Instance

```bash
# Get the recovery instance details
RECOVERY_INSTANCE=$(aws drs describe-recovery-instances \
  --region $DR_REGION \
  --filters "sourceServerIDs=$SOURCE_SERVER_ID" \
  --query 'Items[0].{EC2ID:Ec2InstanceID,State:RecoveryInstanceProperties.State,IP:RecoveryInstanceProperties.IPs}')

echo "$RECOVERY_INSTANCE"

# Test the web application on the recovery instance
RECOVERY_IP=$(echo $RECOVERY_INSTANCE | jq -r '.IP[0]')
curl http://$RECOVERY_IP
# Expected: "<h1>Primary Region - DR Lab Source Server</h1>"
```

#### Step 7: Terminate Drill Instances

```bash
# Clean up the drill — terminate recovery instances
aws drs terminate-recovery-instances \
  --region $DR_REGION \
  --recovery-instance-ids "$(aws drs describe-recovery-instances \
    --region $DR_REGION \
    --query 'Items[0].RecoveryInstanceID' \
    --output text)"
```

#### Verification Checklist

- [ ] DRS service initialized in DR region
- [ ] Agent installed and registered on source server
- [ ] Replication state reached CONTINUOUS
- [ ] DR drill launched successfully (5-15 minutes)
- [ ] Recovery instance booted and web app accessible
- [ ] Drill instance terminated cleanly

#### Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Agent fails to install | No outbound connectivity | Ensure security group allows TCP 443 + 1500 outbound |
| Replication stuck in INITIATING | IAM permissions | Verify agent IAM policy includes `drs:*` and EC2 permissions |
| Replication state STALLED | Network bandwidth saturated | Check source server network; consider bandwidth throttling |
| Drill fails LAUNCH_FAILED | No matching AMI in DR region | Update launch template; ensure AMI compatibility |
| High replication lag | Large write workload | Monitor `BackloggedStorageBytes`; consider larger replication server |

---

### 🔴 Complex Setup

> **Estimated Time:** 3–4 hours  
> **Estimated Cost:** ~$20–40/day while replicating

#### Step 1: VPC & Network Setup for Cross-Region DR

```bash
# Create DR VPC in us-west-2
DR_VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.1.0.0/16 \
  --region $DR_REGION \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=DR-VPC},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Vpc.VpcId' --output text)

# Create subnets for staging and recovery
STAGING_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $DR_VPC_ID \
  --cidr-block 10.1.1.0/24 \
  --availability-zone ${DR_REGION}a \
  --region $DR_REGION \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=DRS-Staging-Subnet}]" \
  --query 'Subnet.SubnetId' --output text)

RECOVERY_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $DR_VPC_ID \
  --cidr-block 10.1.2.0/24 \
  --availability-zone ${DR_REGION}a \
  --region $DR_REGION \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=DRS-Recovery-Subnet}]" \
  --query 'Subnet.SubnetId' --output text)

# Create VPC Endpoints for private connectivity (no internet required)
aws ec2 create-vpc-endpoint \
  --vpc-id $DR_VPC_ID \
  --service-name com.amazonaws.${DR_REGION}.drs \
  --vpc-endpoint-type Interface \
  --subnet-ids $STAGING_SUBNET \
  --region $DR_REGION

aws ec2 create-vpc-endpoint \
  --vpc-id $DR_VPC_ID \
  --service-name com.amazonaws.${DR_REGION}.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids $(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$DR_VPC_ID" --region $DR_REGION --query 'RouteTables[0].RouteTableId' --output text) \
  --region $DR_REGION
```

#### Step 2: Configure Replication Template for Multi-Server

```bash
# Get the default replication configuration template
TEMPLATE_ID=$(aws drs describe-replication-configuration-templates \
  --region $DR_REGION \
  --query 'Items[0].ReplicationConfigurationTemplateID' \
  --output text)

# Update with production settings
aws drs update-replication-configuration-template \
  --region $DR_REGION \
  --replication-configuration-template-id $TEMPLATE_ID \
  --staging-area-subnet-id $STAGING_SUBNET \
  --associate-default-security-group \
  --replication-server-instance-type "t3.small" \
  --use-dedicated-replication-server false \
  --data-plane-routing "PRIVATE_IP" \
  --create-public-ip false \
  --bandwidth-throttling 0 \
  --pit-policy '[
    {"interval": 10, "units": "MINUTE", "retentionDuration": 60, "enabled": true},
    {"interval": 1, "units": "HOUR", "retentionDuration": 24, "enabled": true},
    {"interval": 1, "units": "DAY", "retentionDuration": 30, "enabled": true}
  ]'
```

#### Step 3: Create Launch Configuration for Recovery

```bash
# After source servers are registered, configure launch settings
# (This sets up the recovery instance configuration)

# For each source server:
aws drs update-launch-configuration \
  --region $DR_REGION \
  --source-server-id $SOURCE_SERVER_ID \
  --target-instance-type-right-sizing-method "BASIC" \
  --launch-disposition "STARTED" \
  --licensing '{"OsByol": true}' \
  --post-launch-actions '{
    "Deployment": "TEST_AND_CUTOVER",
    "SsmDocuments": [
      {
        "ActionName": "ValidateRecovery",
        "SsmDocumentName": "AWS-RunShellScript",
        "TimeoutSeconds": 300,
        "MustSucceedForCutover": true,
        "Parameters": {
          "commands": ["systemctl status httpd", "curl -s http://localhost"]
        }
      }
    ]
  }'
```

#### Step 4: Create Recovery Plan with Launch Order

```bash
# Create a source network (maps source VPC to target VPC)
aws drs create-source-network \
  --region $DR_REGION \
  --source-network-id "vpc-source-primary" \
  --vpc-id $DR_VPC_ID

# Create a launch configuration template with ordering
# Web → App → DB launch order (DB first, then App, then Web)
aws drs create-launch-configuration-template \
  --region $DR_REGION \
  --post-launch-actions '{
    "Deployment": "TEST_AND_CUTOVER",
    "SsmDocuments": [{
      "ActionName": "WaitForDependencies",
      "SsmDocumentName": "AWS-RunShellScript",
      "TimeoutSeconds": 600,
      "Parameters": {
        "commands": [
          "# Wait for DB tier to be healthy",
          "until curl -s http://db-tier:3306 > /dev/null 2>&1; do sleep 5; done",
          "echo DB tier is ready"
        ]
      }
    }]
  }'
```

#### Step 5: Automated Failover with Lambda + EventBridge

```python
# File: lambda_drs_failover.py
# Lambda function triggered by CloudWatch Alarm → EventBridge

import boto3
import json
import os

drs_client = boto3.client('drs', region_name=os.environ['DR_REGION'])
sns_client = boto3.client('sns')

def lambda_handler(event, context):
    """Automated failover triggered by health check failure."""
    
    # Get all source servers in CONTINUOUS state
    source_servers = drs_client.describe_source_servers()
    
    servers_to_recover = []
    for server in source_servers['Items']:
        if server['DataReplicationInfo']['DataReplicationState'] == 'CONTINUOUS':
            servers_to_recover.append({
                'sourceServerID': server['SourceServerID']
            })
    
    if not servers_to_recover:
        return {'statusCode': 200, 'body': 'No servers ready for recovery'}
    
    # Initiate recovery (NOT a drill)
    response = drs_client.start_recovery(
        isDrill=False,
        sourceServers=servers_to_recover
    )
    
    # Notify team
    sns_client.publish(
        TopicArn=os.environ['SNS_TOPIC_ARN'],
        Subject='🚨 DR Failover Initiated',
        Message=json.dumps({
            'JobID': response['Job']['JobID'],
            'Status': response['Job']['Status'],
            'Servers': len(servers_to_recover),
            'Trigger': event.get('detail', {}).get('alarmName', 'Manual')
        }, indent=2)
    )
    
    return {
        'statusCode': 200,
        'body': f"Recovery initiated: {response['Job']['JobID']}"
    }
```

```bash
# Deploy the Lambda function
zip lambda_drs_failover.zip lambda_drs_failover.py

aws lambda create-function \
  --function-name DRS-AutoFailover \
  --runtime python3.12 \
  --handler lambda_drs_failover.lambda_handler \
  --role arn:aws:iam::${ACCOUNT_ID}:role/DRSFailoverLambdaRole \
  --zip-file fileb://lambda_drs_failover.zip \
  --environment "Variables={DR_REGION=$DR_REGION,SNS_TOPIC_ARN=arn:aws:sns:${PRIMARY_REGION}:${ACCOUNT_ID}:DR-Alerts}" \
  --timeout 300

# Create EventBridge rule triggered by CloudWatch Alarm state change
aws events put-rule \
  --name "DRS-Failover-Trigger" \
  --event-pattern '{
    "source": ["aws.cloudwatch"],
    "detail-type": ["CloudWatch Alarm State Change"],
    "detail": {
      "alarmName": ["Primary-Region-Health-Check"],
      "state": {"value": ["ALARM"]}
    }
  }'

aws events put-targets \
  --rule "DRS-Failover-Trigger" \
  --targets "Id=DRSFailover,Arn=arn:aws:lambda:${PRIMARY_REGION}:${ACCOUNT_ID}:function:DRS-AutoFailover"
```

#### Step 6: Failback Workflow

```bash
# After primary region is recovered, initiate failback

# Step 1: Start reverse replication (from DR instance back to primary)
aws drs reverse-replication \
  --region $DR_REGION \
  --recovery-instance-id "i-recovery-xxxxxxxx"

# Step 2: Monitor reverse replication until CONTINUOUS
watch -n 30 "aws drs describe-source-servers --region $PRIMARY_REGION \
  --query 'Items[0].DataReplicationInfo.DataReplicationState'"

# Step 3: Initiate cutover back to primary
aws drs start-recovery \
  --region $PRIMARY_REGION \
  --source-servers "[{\"sourceServerID\": \"$REVERSE_SOURCE_SERVER_ID\"}]"

# Step 4: Disconnect recovered instances
aws drs disconnect-recovery-instance \
  --region $DR_REGION \
  --recovery-instance-id "i-recovery-xxxxxxxx"
```

#### Step 7: Cost Monitoring Dashboard

```bash
# Create CloudWatch dashboard for DRS cost monitoring
aws cloudwatch put-dashboard \
  --dashboard-name "DRS-Cost-Monitor" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/DRS", "ReplicationLag", "SourceServerID", "'$SOURCE_SERVER_ID'"]
          ],
          "period": 300,
          "title": "Replication Lag (seconds)"
        }
      },
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/DRS", "BackloggedStorageBytes", "SourceServerID", "'$SOURCE_SERVER_ID'"]
          ],
          "period": 300,
          "title": "Backlogged Data (bytes)"
        }
      }
    ]
  }'
```

---

## Lab 3: Amazon Application Recovery Controller (ARC) — Warm Standby & Active-Active

> **DR Strategy:** Warm Standby (Active/Passive) & Active-Active (Multi-Site)  
> **RTO:** Seconds–Minutes | **RPO:** Near-zero  
> **Cost Profile:** $$$ (Warm Standby) to $$$$ (Active-Active)

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                   AMAZON APPLICATION RECOVERY CONTROLLER (ARC)                     │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│    ┌───────────────────────────────────────────────────────┐                      │
│    │          ARC CLUSTER (5 Regional Endpoints)           │                      │
│    │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐       │                      │
│    │  │ORD  │  │IAD  │  │DUB  │  │NRT  │  │SYD  │       │                      │
│    │  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘       │                      │
│    └────────────────────────┬──────────────────────────────┘                      │
│                             │                                                     │
│              ┌──────────────┼──────────────┐                                      │
│              │              │              │                                       │
│    ┌─────────▼──────┐  ┌───▼────┐  ┌─────▼────────┐                              │
│    │ Routing Control │  │ Safety │  │ Health Check │                              │
│    │ (us-east-1: ON) │  │ Rules  │  │ (Route 53)  │                              │
│    │ (us-west-2: ON) │  │ (≥1   │  │              │                              │
│    │                  │  │ active)│  │              │                              │
│    └────────┬─────────┘  └────────┘  └──────┬──────┘                              │
│             │                                │                                    │
│     ┌───────▼────────────────────────────────▼───────┐                            │
│     │              ROUTE 53 (DNS)                      │                           │
│     │   app.example.com → FAILOVER / MULTIVALUE        │                           │
│     └───────┬─────────────────────────────┬───────────┘                           │
│             │                             │                                       │
│    ┌────────▼─────────┐         ┌─────────▼────────┐                              │
│    │  us-east-1       │         │  us-west-2       │                              │
│    │  (PRIMARY)       │         │  (DR/STANDBY)    │                              │
│    │                  │         │                  │                              │
│    │  ┌─────┐  ┌───┐ │         │  ┌─────┐  ┌───┐ │                              │
│    │  │ ALB │  │ASG│ │         │  │ ALB │  │ASG│ │                              │
│    │  └──┬──┘  └───┘ │         │  └──┬──┘  └───┘ │                              │
│    │     │            │         │     │            │                              │
│    │  ┌──▼──────────┐ │         │  ┌──▼──────────┐ │                              │
│    │  │ Aurora      │ │         │  │ Aurora      │ │                              │
│    │  │ Global DB   │◄──replication──▶ Read     │ │                              │
│    │  │ (Writer)    │ │         │  │ Replica    │ │                              │
│    │  └─────────────┘ │         │  └─────────────┘ │                              │
│    └──────────────────┘         └──────────────────┘                              │
│                                                                                   │
│    ZONAL SHIFT / AUTOSHIFT (Single-Region AZ Recovery)                            │
│    ┌──────────────────────────────────────────┐                                   │
│    │  ALB in us-east-1                        │                                   │
│    │  ┌────┐ ┌────┐ ┌────┐                   │                                   │
│    │  │AZ-a│ │AZ-b│ │AZ-c│  ← Shift away    │                                   │
│    │  │ ✓  │ │ ✗  │ │ ✓  │    from AZ-b     │                                   │
│    │  └────┘ └────┘ └────┘                   │                                   │
│    └──────────────────────────────────────────┘                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

### 🟢 Simple Setup

> **Estimated Time:** 30–45 minutes  
> **Estimated Cost:** ~$1–3 (zonal shift is free; ALB cost only)

#### Step 1: Deploy a Multi-AZ ALB with Target Group

```bash
# Create an ALB in multiple AZs
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name arc-lab-alb \
  --subnets subnet-az1 subnet-az2 subnet-az3 \
  --security-groups sg-xxxxxxxx \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

echo "ALB ARN: $ALB_ARN"

# Create target group
TG_ARN=$(aws elbv2 create-target-group \
  --name arc-lab-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-path "/" \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions "Type=forward,TargetGroupArn=$TG_ARN"

# Launch instances in multiple AZs and register them
for AZ in a b c; do
  INST_ID=$(aws ec2 run-instances \
    --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
    --instance-type t3.micro \
    --subnet-id subnet-az${AZ} \
    --user-data '#!/bin/bash
yum install -y httpd
systemctl start httpd
echo "Hello from AZ-'$AZ' ($(hostname))" > /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=arc-lab-az${AZ}}]" \
    --query 'Instances[0].InstanceId' --output text)
  
  aws elbv2 register-targets --target-group-arn $TG_ARN --targets "Id=$INST_ID"
done
```

#### Step 2: Enable Zonal Shift for the ALB

```bash
# List available managed resources for zonal shift
aws arc-zonal-shift list-managed-resources \
  --query 'Items[*].{ARN:Arn,Name:Name,AZs:AvailabilityZones}'

# The ALB should appear automatically. If not, wait a few minutes.
```

#### Step 3: Perform a Zonal Shift (Simulate AZ Failure)

```bash
# Start a zonal shift — move traffic away from AZ-b
ZONAL_SHIFT_ID=$(aws arc-zonal-shift start-zonal-shift \
  --resource-identifier "$ALB_ARN" \
  --away-from "${PRIMARY_REGION}b" \
  --expires-in "1h" \
  --comment "Lab test - simulating AZ-b impairment" \
  --query 'ZonalShiftId' \
  --output text)

echo "Zonal Shift ID: $ZONAL_SHIFT_ID"

# Verify the shift is active
aws arc-zonal-shift get-managed-resource \
  --resource-identifier "$ALB_ARN" \
  --query '{AppliedShifts:AppliedWeights}'
```

**Expected Output:**
```json
{
    "AppliedWeights": {
        "us-east-1a": 1.0,
        "us-east-1b": 0.0,   ← Traffic shifted away
        "us-east-1c": 1.0
    }
}
```

#### Step 4: Test Traffic Distribution

```bash
# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers --names arc-lab-alb \
  --query 'LoadBalancers[0].DNSName' --output text)

# Send multiple requests and observe distribution
for i in $(seq 1 20); do
  curl -s http://$ALB_DNS
done | sort | uniq -c
# Should show traffic only from AZ-a and AZ-c, NOT AZ-b
```

#### Step 5: Cancel the Zonal Shift

```bash
# Cancel the zonal shift (restore normal traffic)
aws arc-zonal-shift cancel-zonal-shift \
  --zonal-shift-id $ZONAL_SHIFT_ID

# Verify traffic is restored to all AZs
sleep 30
for i in $(seq 1 20); do
  curl -s http://$ALB_DNS
done | sort | uniq -c
```

#### Step 6: Enable Zonal Autoshift

```bash
# Enable autoshift — AWS will automatically shift traffic during AZ issues
aws arc-zonal-shift create-practice-run-configuration \
  --resource-identifier "$ALB_ARN" \
  --outcome-alarms '[{
    "AlarmIdentifier": "arn:aws:cloudwatch:'$PRIMARY_REGION':'$ACCOUNT_ID':alarm:ALB-Error-Rate",
    "Type": "CLOUDWATCH"
  }]' \
  --blocked-windows '["mon:00:00-mon:02:00"]'

# Update autoshift status to ENABLED
aws arc-zonal-shift update-zonal-autoshift-configuration \
  --resource-identifier "$ALB_ARN" \
  --zonal-autoshift-status "ENABLED"
```

#### Verification Checklist

- [ ] ALB deployed across 3 AZs with healthy targets
- [ ] Zonal shift successfully moved traffic away from target AZ
- [ ] Only instances in healthy AZs received traffic
- [ ] Zonal shift cancelled and traffic restored
- [ ] Zonal autoshift enabled with practice run configuration

---

### 🔴 Complex Setup

> **Estimated Time:** 4–5 hours  
> **Estimated Cost:** ~$50–80/day (ARC cluster = $2.50/hr, Aurora Global = $$, Multi-region infra)

#### Step 1: Create ARC Cluster and Routing Controls

```bash
# Create an ARC cluster (takes ~5 minutes to provision)
CLUSTER_ARN=$(aws route53-recovery-control-config create-cluster \
  --cluster-name "dr-lab-cluster" \
  --query 'Cluster.ClusterArn' \
  --output text)

echo "Cluster ARN: $CLUSTER_ARN"
# ⚠️ Cluster costs $2.50/hour — delete when done!

# Wait for cluster to be DEPLOYED
aws route53-recovery-control-config describe-cluster \
  --cluster-arn $CLUSTER_ARN \
  --query 'Cluster.Status'

# Create a control panel
CONTROL_PANEL_ARN=$(aws route53-recovery-control-config create-control-panel \
  --cluster-arn $CLUSTER_ARN \
  --control-panel-name "app-failover-panel" \
  --query 'ControlPanel.ControlPanelArn' \
  --output text)

# Create routing controls (one per region)
RC_EAST=$(aws route53-recovery-control-config create-routing-control \
  --cluster-arn $CLUSTER_ARN \
  --control-panel-arn $CONTROL_PANEL_ARN \
  --routing-control-name "us-east-1-active" \
  --query 'RoutingControl.RoutingControlArn' \
  --output text)

RC_WEST=$(aws route53-recovery-control-config create-routing-control \
  --cluster-arn $CLUSTER_ARN \
  --control-panel-arn $CONTROL_PANEL_ARN \
  --routing-control-name "us-west-2-active" \
  --query 'RoutingControl.RoutingControlArn' \
  --output text)

echo "East RC: $RC_EAST"
echo "West RC: $RC_WEST"
```

#### Step 2: Create Safety Rules

```bash
# Safety rule: At least ONE routing control must be active
aws route53-recovery-control-config create-safety-rule \
  --assertion-rule '{
    "Name": "AtLeastOneActive",
    "ControlPanelArn": "'$CONTROL_PANEL_ARN'",
    "WaitPeriodMs": 5000,
    "AssertedControls": ["'$RC_EAST'", "'$RC_WEST'"],
    "RuleConfig": {
      "Type": "ATLEAST",
      "Threshold": 1,
      "Inverted": false
    }
  }'
```

#### Step 3: Create Route 53 Health Checks Linked to Routing Controls

```bash
# Create health check for us-east-1 routing control
HC_EAST=$(aws route53 create-health-check \
  --caller-reference "rc-east-$(date +%s)" \
  --health-check-config '{
    "Type": "RECOVERY_CONTROL",
    "RoutingControlArn": "'$RC_EAST'"
  }' \
  --query 'HealthCheck.Id' \
  --output text)

# Create health check for us-west-2 routing control
HC_WEST=$(aws route53 create-health-check \
  --caller-reference "rc-west-$(date +%s)" \
  --health-check-config '{
    "Type": "RECOVERY_CONTROL",
    "RoutingControlArn": "'$RC_WEST'"
  }' \
  --query 'HealthCheck.Id' \
  --output text)

echo "Health Check East: $HC_EAST"
echo "Health Check West: $HC_WEST"
```

#### Step 4: Configure Route 53 DNS Failover Records

```bash
# Assumes you have a hosted zone — replace with your zone ID and domain
HOSTED_ZONE_ID="Z1234567890"
DOMAIN="app.example.com"

# Create failover records (Primary + Secondary)
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "'$DOMAIN'",
          "Type": "A",
          "SetIdentifier": "primary-us-east-1",
          "Failover": "PRIMARY",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "alb-east.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          },
          "HealthCheckId": "'$HC_EAST'"
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "'$DOMAIN'",
          "Type": "A",
          "SetIdentifier": "secondary-us-west-2",
          "Failover": "SECONDARY",
          "AliasTarget": {
            "HostedZoneId": "Z1H1FL5HABSF5",
            "DNSName": "alb-west.us-west-2.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          },
          "HealthCheckId": "'$HC_WEST'"
        }
      }
    ]
  }'
```

#### Step 5: Set Up DynamoDB Global Tables (Active-Active Data Layer)

```bash
# Create a DynamoDB table with Global Table replication
aws dynamodb create-table \
  --table-name "app-sessions" \
  --attribute-definitions AttributeName=SessionId,AttributeType=S \
  --key-schema AttributeName=SessionId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region $PRIMARY_REGION

# Add global table replica in DR region
aws dynamodb update-table \
  --table-name "app-sessions" \
  --region $PRIMARY_REGION \
  --replica-updates '[{"Create": {"RegionName": "'$DR_REGION'"}}]'

# Verify replication
aws dynamodb describe-table --table-name "app-sessions" --region $PRIMARY_REGION \
  --query 'Table.Replicas'
```

#### Step 6: Configure Aurora Global Database

```bash
# Create Aurora Global Database
aws rds create-global-cluster \
  --global-cluster-identifier "dr-lab-global-db" \
  --engine aurora-mysql \
  --engine-version "8.0.mysql_aurora.3.04.0" \
  --deletion-protection

# Create primary cluster (us-east-1)
aws rds create-db-cluster \
  --db-cluster-identifier "dr-lab-primary" \
  --engine aurora-mysql \
  --engine-version "8.0.mysql_aurora.3.04.0" \
  --master-username admin \
  --master-user-password "YourSecurePassword123!" \
  --global-cluster-identifier "dr-lab-global-db" \
  --region $PRIMARY_REGION

# Create primary instance
aws rds create-db-instance \
  --db-instance-identifier "dr-lab-primary-instance" \
  --db-cluster-identifier "dr-lab-primary" \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --region $PRIMARY_REGION

# Add secondary cluster in DR region
aws rds create-db-cluster \
  --db-cluster-identifier "dr-lab-secondary" \
  --engine aurora-mysql \
  --engine-version "8.0.mysql_aurora.3.04.0" \
  --global-cluster-identifier "dr-lab-global-db" \
  --region $DR_REGION

aws rds create-db-instance \
  --db-instance-identifier "dr-lab-secondary-instance" \
  --db-cluster-identifier "dr-lab-secondary" \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --region $DR_REGION
```

#### Step 7: Automated Failover with Step Functions

```json
{
  "Comment": "ARC Multi-Region Failover Orchestration",
  "StartAt": "DetectFailure",
  "States": {
    "DetectFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:route53RecoveryCluster:getRoutingControlState",
      "Parameters": {
        "RoutingControlArn": "${RC_EAST}"
      },
      "Next": "CheckIfPrimaryHealthy"
    },
    "CheckIfPrimaryHealthy": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.RoutingControlState",
          "StringEquals": "Off",
          "Next": "FailoverToSecondary"
        }
      ],
      "Default": "PrimaryHealthy"
    },
    "FailoverToSecondary": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "ActivateDRRegion",
          "States": {
            "ActivateDRRegion": {
              "Type": "Task",
              "Resource": "arn:aws:states:::aws-sdk:route53RecoveryCluster:updateRoutingControlState",
              "Parameters": {
                "RoutingControlArn": "${RC_WEST}",
                "RoutingControlState": "On"
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "FailoverAuroraGlobal",
          "States": {
            "FailoverAuroraGlobal": {
              "Type": "Task",
              "Resource": "arn:aws:states:::aws-sdk:rds:failoverGlobalCluster",
              "Parameters": {
                "GlobalClusterIdentifier": "dr-lab-global-db",
                "TargetDbClusterIdentifier": "arn:aws:rds:us-west-2:${ACCOUNT_ID}:cluster:dr-lab-secondary"
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "NotifyTeam",
          "States": {
            "NotifyTeam": {
              "Type": "Task",
              "Resource": "arn:aws:states:::sns:publish",
              "Parameters": {
                "TopicArn": "arn:aws:sns:us-east-1:${ACCOUNT_ID}:DR-Alerts",
                "Subject": "🚨 Region Failover Executed",
                "Message": "Failover from us-east-1 to us-west-2 initiated"
              },
              "End": true
            }
          }
        }
      ],
      "Next": "ValidateFailover"
    },
    "ValidateFailover": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-west-2:${ACCOUNT_ID}:function:ValidateDRHealth",
      "End": true
    },
    "PrimaryHealthy": {
      "Type": "Succeed"
    }
  }
}
```

```bash
# Create the Step Functions state machine
aws stepfunctions create-state-machine \
  --name "ARC-Failover-Orchestration" \
  --definition file://failover-state-machine.json \
  --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/StepFunctionsFailoverRole"

# Trigger with CloudWatch Alarm
aws events put-rule \
  --name "Primary-Region-Failure" \
  --event-pattern '{
    "source": ["aws.cloudwatch"],
    "detail-type": ["CloudWatch Alarm State Change"],
    "detail": {
      "alarmName": ["Primary-Region-Health"],
      "state": {"value": ["ALARM"]}
    }
  }'
```

#### Step 8: Execute Failover Test

```bash
# Get cluster endpoint for routing control updates
CLUSTER_ENDPOINTS=$(aws route53-recovery-control-config describe-cluster \
  --cluster-arn $CLUSTER_ARN \
  --query 'Cluster.ClusterEndpoints[*].Endpoint' \
  --output json)

# Use the first available endpoint
ENDPOINT=$(echo $CLUSTER_ENDPOINTS | jq -r '.[0]')

# Failover: Turn OFF us-east-1, ensure us-west-2 is ON
aws route53-recovery-cluster update-routing-control-states \
  --endpoint-url "https://$ENDPOINT" \
  --update-routing-control-state-entries '[
    {"RoutingControlArn": "'$RC_EAST'", "RoutingControlState": "Off"},
    {"RoutingControlArn": "'$RC_WEST'", "RoutingControlState": "On"}
  ]'

# Verify DNS resolves to DR region
dig +short $DOMAIN
# Should resolve to us-west-2 ALB IP

# Failback: Restore both regions (active-active) or swap
aws route53-recovery-cluster update-routing-control-states \
  --endpoint-url "https://$ENDPOINT" \
  --update-routing-control-state-entries '[
    {"RoutingControlArn": "'$RC_EAST'", "RoutingControlState": "On"},
    {"RoutingControlArn": "'$RC_WEST'", "RoutingControlState": "On"}
  ]'
```

#### Verification Checklist (Complex)

- [ ] ARC cluster deployed (5 regional endpoints active)
- [ ] Routing controls created for both regions
- [ ] Safety rule prevents both controls being OFF simultaneously
- [ ] Route 53 health checks linked to routing controls
- [ ] DNS failover records configured and resolving correctly
- [ ] DynamoDB Global Table replicating across regions
- [ ] Aurora Global Database with secondary cluster in DR region
- [ ] Step Functions state machine deployed
- [ ] Failover test: traffic shifts to DR region within seconds
- [ ] Failback test: traffic returns to primary successfully

---

## Lab 4: AWS Resilience Hub — Assessment & Governance

> **Purpose:** Continuous resilience assessment, RTO/RPO validation, drift detection  
> **Cost:** $15/service/month (assessment) + $10/service/month (dependency discovery, optional)

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        AWS RESILIENCE HUB                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────┐         │
│  │                  RESILIENCE POLICY                            │         │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │         │
│  │  │ Tier 1      │ │ Tier 2      │ │ Tier 3              │   │         │
│  │  │ RTO: 5min   │ │ RTO: 1hr    │ │ RTO: 24hr           │   │         │
│  │  │ RPO: 1min   │ │ RPO: 1hr    │ │ RPO: 24hr           │   │         │
│  │  │ (Critical)  │ │ (Standard)  │ │ (Non-Critical)      │   │         │
│  │  └─────────────┘ └─────────────┘ └─────────────────────┘   │         │
│  └─────────────────────────────────────────────────────────────┘         │
│                              │                                            │
│                              ▼                                            │
│  ┌─────────────────────────────────────────────────────────────┐         │
│  │              APPLICATION / SERVICE MODEL                      │         │
│  │  Defined via: CloudFormation | Tags | Terraform | EKS        │         │
│  │                                                              │         │
│  │  ┌────────────┐    ┌────────────┐    ┌────────────┐         │         │
│  │  │ EC2 + ALB  │    │    RDS     │    │   Lambda   │         │         │
│  │  └────────────┘    └────────────┘    └────────────┘         │         │
│  └─────────────────────────────────────────────────────────────┘         │
│                              │                                            │
│                              ▼                                            │
│  ┌─────────────────────────────────────────────────────────────┐         │
│  │              RESILIENCE ASSESSMENT                            │         │
│  │                                                              │         │
│  │  AI-Powered Failure Mode Analysis:                           │         │
│  │  ├── Single Points of Failure  ──▶ ⚠️ Found: No Multi-AZ   │         │
│  │  ├── Excessive Load             ──▶ ✅ Pass                 │         │
│  │  ├── Excessive Latency          ──▶ ✅ Pass                 │         │
│  │  ├── Misconfiguration           ──▶ ⚠️ Found: No backups   │         │
│  │  └── Shared Fate                ──▶ ⚠️ Found: Same AZ      │         │
│  │                                                              │         │
│  │  RTO/RPO Assessment:                                         │         │
│  │  ├── Application Disruption  → Est RTO: 15min  ✅ Policy Met│         │
│  │  ├── Infrastructure Disruption → Est RTO: 30min ✅ Policy Met│        │
│  │  ├── AZ Disruption           → Est RTO: 5min   ✅ Policy Met│         │
│  │  └── Region Disruption       → Est RTO: 4hr    ❌ BREACHED  │         │
│  └─────────────────────────────────────────────────────────────┘         │
│                              │                                            │
│                              ▼                                            │
│  ┌─────────────────────────────────────────────────────────────┐         │
│  │              RECOMMENDATIONS                                  │         │
│  │  ├── CloudWatch Alarms (deployable CFN template)             │         │
│  │  ├── SOPs (Systems Manager Documents)                        │         │
│  │  ├── FIS Chaos Experiments                                   │         │
│  │  └── Architecture Changes                                    │         │
│  └─────────────────────────────────────────────────────────────┘         │
│                                                                           │
│  CI/CD Integration: CodePipeline → Assess → Gate → Deploy                │
│  Drift Detection: Daily assessment → SNS alert on drift                  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### 🟢 Simple Setup

> **Estimated Time:** 30–45 minutes  
> **Estimated Cost:** ~$15/month per service assessed

#### Step 1: Deploy a Sample Application (CloudFormation)

```yaml
# File: sample-app.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Sample application for Resilience Hub assessment

Resources:
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Sub '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}'
      Tags:
        - Key: Name
          Value: resilience-hub-lab
        - Key: Project
          Value: dr-lab

  AppDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: resilience-hub-lab-db
      Engine: mysql
      EngineVersion: '8.0'
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      MasterUsername: admin
      MasterUserPassword: !Sub '{{resolve:secretsmanager:dr-lab-db-password}}'
      MultiAZ: false  # Intentionally single-AZ for assessment to flag
      BackupRetentionPeriod: 1
      Tags:
        - Key: Project
          Value: dr-lab

Outputs:
  StackName:
    Value: !Ref AWS::StackName
  InstanceId:
    Value: !Ref WebServerInstance
  DBInstanceId:
    Value: !Ref AppDatabase
```

```bash
# Create a secret for the DB password first
aws secretsmanager create-secret \
  --name dr-lab-db-password \
  --secret-string "MySecureP@ssw0rd123"

# Deploy the sample app
aws cloudformation deploy \
  --template-file sample-app.yaml \
  --stack-name resilience-hub-lab-app \
  --capabilities CAPABILITY_IAM
```

#### Step 2: Create a Resilience Policy

```bash
# Create a Tier 2 resilience policy
POLICY_ARN=$(aws resiliencehub create-resiliency-policy \
  --policy-name "Tier2-Standard" \
  --tier "NonCritical" \
  --policy '{
    "Software": {"RtoInSecs": 3600, "RpoInSecs": 3600},
    "Hardware": {"RtoInSecs": 3600, "RpoInSecs": 3600},
    "AZ": {"RtoInSecs": 3600, "RpoInSecs": 3600},
    "Region": {"RtoInSecs": 86400, "RpoInSecs": 86400}
  }' \
  --query 'Policy.PolicyArn' \
  --output text)

echo "Policy ARN: $POLICY_ARN"
```

#### Step 3: Create an Application in Resilience Hub

```bash
# Create the application
APP_ARN=$(aws resiliencehub create-app \
  --name "dr-lab-sample-app" \
  --app-template-body '{}' \
  --query 'App.AppArn' \
  --output text)

echo "App ARN: $APP_ARN"

# Add the CloudFormation stack as a resource source
aws resiliencehub add-draft-app-version-resource-mappings \
  --app-arn $APP_ARN \
  --resource-mappings '[
    {
      "MappingType": "CfnStack",
      "PhysicalResourceId": {
        "Identifier": "arn:aws:cloudformation:'$PRIMARY_REGION':'$ACCOUNT_ID':stack/resilience-hub-lab-app/*",
        "Type": "Arn"
      }
    }
  ]'

# Publish the app version
aws resiliencehub publish-app-version --app-arn $APP_ARN

# Assign the resilience policy
aws resiliencehub update-app \
  --app-arn $APP_ARN \
  --policy-arn $POLICY_ARN
```

#### Step 4: Run a Resilience Assessment

```bash
# Start the assessment
ASSESSMENT_ARN=$(aws resiliencehub start-app-assessment \
  --app-arn $APP_ARN \
  --app-version "release" \
  --assessment-name "initial-assessment" \
  --query 'Assessment.AssessmentArn' \
  --output text)

echo "Assessment ARN: $ASSESSMENT_ARN"

# Monitor assessment progress (takes 2-5 minutes)
watch -n 15 "aws resiliencehub describe-app-assessment \
  --assessment-arn $ASSESSMENT_ARN \
  --query 'Assessment.{Status:AssessmentStatus,Compliance:ComplianceStatus}'"
```

**Expected Output:**
```json
{
    "Status": "Success",
    "Compliance": "PolicyBreached"  ← Expected! Single-AZ DB breaches policy
}
```

#### Step 5: Review Assessment Results

```bash
# Get compliance details
aws resiliencehub list-app-assessment-compliance-drifts \
  --assessment-arn $ASSESSMENT_ARN

# Get resiliency recommendations
aws resiliencehub list-app-component-recommendations \
  --assessment-arn $ASSESSMENT_ARN \
  --query 'ComponentRecommendations[*].{Component:AppComponentName,Type:RecommendationStatus,Items:ConfigRecommendations[*].Description}' \
  --output table

# Get operational recommendations (alarms, SOPs, FIS experiments)
aws resiliencehub list-sop-recommendations --assessment-arn $ASSESSMENT_ARN
aws resiliencehub list-alarm-recommendations --assessment-arn $ASSESSMENT_ARN
aws resiliencehub list-test-recommendations --assessment-arn $ASSESSMENT_ARN
```

#### Step 6: Deploy Recommended CloudWatch Alarms

```bash
# Get the recommended alarm CloudFormation template
aws resiliencehub create-recommendation-template \
  --assessment-arn $ASSESSMENT_ARN \
  --name "alarm-recommendations" \
  --recommendation-types '["Alarm"]' \
  --format "CfnYaml"

# Wait for template generation
sleep 30

# List recommendation templates
TEMPLATE_ARN=$(aws resiliencehub list-recommendation-templates \
  --assessment-arn $ASSESSMENT_ARN \
  --query 'RecommendationTemplates[0].RecommendationTemplateArn' \
  --output text)

# Describe to get the S3 URL
aws resiliencehub describe-recommendation-template \
  --recommendation-template-arn $TEMPLATE_ARN \
  --query 'RecommendationTemplate.TemplatesUrl'

# Download and deploy the CFN template
# aws cloudformation deploy --template-file alarm-template.yaml --stack-name resilience-alarms
```

#### Verification Checklist

- [ ] Sample application deployed via CloudFormation
- [ ] Resilience policy created (Tier 2: RTO 1hr, RPO 1hr)
- [ ] Application registered in Resilience Hub
- [ ] Assessment completed (expect "PolicyBreached" for single-AZ DB)
- [ ] Recommendations visible (Multi-AZ, backups, alarms)
- [ ] Alarm recommendation template generated

---

### 🔴 Complex Setup

> **Estimated Time:** 2–3 hours  
> **Estimated Cost:** ~$30–50/month (multiple services assessed + dependency discovery)

#### Step 1: Next-Gen Resilience Hub — Service Model

> **Note:** The next-gen Resilience Hub (May 2026) uses "services" instead of "applications."

```bash
# Configure a modular resilience policy (Tier 1 — Critical)
TIER1_POLICY=$(aws resiliencehub create-resiliency-policy \
  --policy-name "Tier1-Critical" \
  --tier "MissionCritical" \
  --policy '{
    "Software": {"RtoInSecs": 300, "RpoInSecs": 60},
    "Hardware": {"RtoInSecs": 300, "RpoInSecs": 60},
    "AZ": {"RtoInSecs": 0, "RpoInSecs": 0},
    "Region": {"RtoInSecs": 900, "RpoInSecs": 300}
  }' \
  --query 'Policy.PolicyArn' --output text)

# Tier 2 — Standard
TIER2_POLICY=$(aws resiliencehub create-resiliency-policy \
  --policy-name "Tier2-Standard" \
  --tier "Important" \
  --policy '{
    "Software": {"RtoInSecs": 3600, "RpoInSecs": 3600},
    "Hardware": {"RtoInSecs": 3600, "RpoInSecs": 3600},
    "AZ": {"RtoInSecs": 300, "RpoInSecs": 60},
    "Region": {"RtoInSecs": 14400, "RpoInSecs": 3600}
  }' \
  --query 'Policy.PolicyArn' --output text)

# Tier 3 — Non-Critical
TIER3_POLICY=$(aws resiliencehub create-resiliency-policy \
  --policy-name "Tier3-NonCritical" \
  --tier "NonCritical" \
  --policy '{
    "Software": {"RtoInSecs": 86400, "RpoInSecs": 86400},
    "Hardware": {"RtoInSecs": 86400, "RpoInSecs": 86400},
    "AZ": {"RtoInSecs": 3600, "RpoInSecs": 3600},
    "Region": {"RtoInSecs": 86400, "RpoInSecs": 86400}
  }' \
  --query 'Policy.PolicyArn' --output text)
```

#### Step 2: Enable Dependency Discovery

```bash
# Enable dependency discovery (requires DNS query logging)
# First, ensure Route 53 Resolver query logging is enabled

# Create a query log config
aws route53resolver create-resolver-query-log-config \
  --name "resilience-hub-dns-logs" \
  --destination-arn "arn:aws:logs:${PRIMARY_REGION}:${ACCOUNT_ID}:log-group:/aws/route53resolver/dns-queries"

# Associate with VPC
aws route53resolver associate-resolver-query-log-config \
  --resolver-query-log-config-id "rqlc-xxxxxxxx" \
  --resource-id $VPC_ID

# In next-gen Resilience Hub, dependency discovery is enabled per-service
# via the console: Services → Select Service → Enable Dependency Discovery
# Cost: $10/service/month
```

#### Step 3: CI/CD Integration with CodePipeline

```yaml
# File: resilience-gate-buildspec.yaml
# CodeBuild buildspec that runs Resilience Hub assessment as a quality gate
version: 0.2

env:
  variables:
    APP_ARN: "arn:aws:resiliencehub:us-east-1:123456789012:app/xxxxxxxx"

phases:
  build:
    commands:
      # Import latest resources
      - |
        aws resiliencehub import-resources-to-draft-app-version \
          --app-arn $APP_ARN
      
      # Publish new version
      - |
        aws resiliencehub publish-app-version --app-arn $APP_ARN
      
      # Start assessment
      - |
        ASSESSMENT_ARN=$(aws resiliencehub start-app-assessment \
          --app-arn $APP_ARN \
          --app-version "release" \
          --assessment-name "pipeline-$(date +%Y%m%d-%H%M%S)" \
          --query 'Assessment.AssessmentArn' --output text)
      
      # Wait for assessment to complete (max 10 minutes)
      - |
        for i in $(seq 1 40); do
          STATUS=$(aws resiliencehub describe-app-assessment \
            --assessment-arn $ASSESSMENT_ARN \
            --query 'Assessment.AssessmentStatus' --output text)
          if [ "$STATUS" = "Success" ] || [ "$STATUS" = "Failed" ]; then
            break
          fi
          sleep 15
        done
      
      # Check compliance — fail pipeline if policy breached
      - |
        COMPLIANCE=$(aws resiliencehub describe-app-assessment \
          --assessment-arn $ASSESSMENT_ARN \
          --query 'Assessment.ComplianceStatus' --output text)
        
        if [ "$COMPLIANCE" = "PolicyBreached" ]; then
          echo "❌ RESILIENCE POLICY BREACHED — Blocking deployment"
          echo "Assessment: $ASSESSMENT_ARN"
          aws resiliencehub list-app-component-recommendations \
            --assessment-arn $ASSESSMENT_ARN
          exit 1
        fi
        
        echo "✅ Resilience policy met — Deployment approved"
```

```bash
# Create the CodePipeline with Resilience Hub gate
# (Simplified — add this as a stage between Build and Deploy)
aws codepipeline update-pipeline --pipeline '{
  "name": "my-app-pipeline",
  "stages": [
    {"name": "Source", "..."},
    {"name": "Build", "..."},
    {
      "name": "ResilienceGate",
      "actions": [{
        "name": "AssessResilience",
        "actionTypeId": {
          "category": "Build",
          "owner": "AWS",
          "provider": "CodeBuild",
          "version": "1"
        },
        "configuration": {
          "ProjectName": "resilience-hub-gate"
        }
      }]
    },
    {"name": "Deploy", "..."}
  ]
}'
```

#### Step 4: Drift Detection with SNS Notifications

```bash
# Create SNS topic for drift alerts
DRIFT_TOPIC_ARN=$(aws sns create-topic --name "resilience-drift-alerts" \
  --query 'TopicArn' --output text)

aws sns subscribe \
  --topic-arn $DRIFT_TOPIC_ARN \
  --protocol email \
  --notification-endpoint "your-email@example.com"

# Enable drift detection on the app (runs daily assessment)
aws resiliencehub update-app \
  --app-arn $APP_ARN \
  --event-subscriptions '[{
    "Name": "DriftDetection",
    "EventType": "DriftDetected",
    "SnsTopicArn": "'$DRIFT_TOPIC_ARN'"
  }]'

# The app will now be assessed daily and notify on drift
```

#### Step 5: FIS Chaos Experiment from Recommendations

```bash
# Get FIS experiment recommendations
aws resiliencehub list-test-recommendations \
  --assessment-arn $ASSESSMENT_ARN \
  --query 'TestRecommendations[*].{Name:Name,Description:Description,Type:Type}'

# Create an FIS experiment template based on recommendations
aws fis create-experiment-template \
  --description "Resilience Hub recommended - AZ failure test" \
  --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/FISExperimentRole" \
  --stop-conditions '[{"source": "aws:cloudwatch:alarm", "value": "arn:aws:cloudwatch:'$PRIMARY_REGION':'$ACCOUNT_ID':alarm:HighErrorRate"}]' \
  --targets '{
    "ec2-instances": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": {"Project": "dr-lab"},
      "selectionMode": "ALL"
    }
  }' \
  --actions '{
    "stop-instances": {
      "actionId": "aws:ec2:stop-instances",
      "parameters": {},
      "targets": {"Instances": "ec2-instances"},
      "description": "Stop all instances to simulate infrastructure failure"
    }
  }' \
  --tags "Project=dr-lab,Purpose=resilience-testing"

# Run the experiment
aws fis start-experiment \
  --experiment-template-id "EXT-xxxxxxxx" \
  --tags "RunBy=ResilienceHub,AssessmentArn=$ASSESSMENT_ARN"
```

#### Step 6: Multi-Account Centralized Assessment

```bash
# From the management account, create a centralized app
# that spans multiple accounts using cross-account IAM roles

# In member accounts, create a trust role:
aws iam create-role \
  --role-name AWSResilienceHubCrossAccountRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::'$MANAGEMENT_ACCOUNT_ID':root"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name AWSResilienceHubCrossAccountRole \
  --policy-arn arn:aws:iam::aws:policy/AWSResilienceHubAsssessmentExecutionPolicy

# In the central account, add cross-account resources
aws resiliencehub add-draft-app-version-resource-mappings \
  --app-arn $APP_ARN \
  --resource-mappings '[
    {
      "MappingType": "CfnStack",
      "PhysicalResourceId": {
        "Identifier": "arn:aws:cloudformation:'$PRIMARY_REGION':'$MEMBER_ACCOUNT_ID':stack/production-app/*",
        "Type": "Arn",
        "AwsAccountId": "'$MEMBER_ACCOUNT_ID'",
        "AwsRegion": "'$PRIMARY_REGION'"
      }
    }
  ]'
```

#### Verification Checklist (Complex)

- [ ] Multi-tier resilience policies created (Tier 1, 2, 3)
- [ ] DNS query logging enabled for dependency discovery
- [ ] CI/CD pipeline includes resilience gate (blocks on policy breach)
- [ ] Drift detection enabled with SNS email notifications
- [ ] FIS experiment created from Resilience Hub recommendations
- [ ] Cross-account assessment configured
- [ ] Automated remediation triggers on policy breach

---

## Lab 5: End-to-End DR Architecture (Combining All Services)

> **Purpose:** Integrate all services into a cohesive DR solution  
> **Pattern:** Defense-in-depth with multiple recovery capabilities

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                    END-TO-END DR ARCHITECTURE                                          │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│  ┌─── AWS RESILIENCE HUB (Continuous Assessment & Governance) ────────────────────┐  │
│  │    Policy: Tier 1 (RTO 5min, RPO 1min) | Drift Detection | CI/CD Gate         │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
│  ┌────────────── AMAZON ARC (Traffic Management) ─────────────────────────────────┐  │
│  │  Routing Controls: us-east-1=ON, us-west-2=ON | Safety: ≥1 active             │  │
│  │  Zonal Autoshift: Enabled | Region Switch: Alarm-triggered                    │  │
│  └────────────────────────────┬───────────────────────────────┬───────────────────┘  │
│                               │                               │                       │
│  ┌────────────────────────────▼───────────┐  ┌───────────────▼────────────────────┐  │
│  │     PRIMARY REGION (us-east-1)          │  │     DR REGION (us-west-2)          │  │
│  │                                         │  │                                    │  │
│  │  ┌─────────────────────────────┐        │  │  ┌─────────────────────────────┐   │  │
│  │  │         ALB (Multi-AZ)       │        │  │  │         ALB (Multi-AZ)       │   │  │
│  │  │   (Zonal Autoshift Enabled)  │        │  │  │   (Zonal Autoshift Enabled)  │   │  │
│  │  └──────────────┬──────────────┘        │  │  └──────────────┬──────────────┘   │  │
│  │                 │                        │  │                 │                   │  │
│  │  ┌──────────────▼──────────────┐        │  │  ┌──────────────▼──────────────┐   │  │
│  │  │    ASG (App Tier)           │        │  │  │    ASG (Warm Standby)       │   │  │
│  │  │    min: 3, max: 10          │        │  │  │    min: 1, max: 10          │   │  │
│  │  │    ┌───┐ ┌───┐ ┌───┐       │        │  │  │    ┌───┐                    │   │  │
│  │  │    │EC2│ │EC2│ │EC2│       │        │  │  │    │EC2│ (scaled down)      │   │  │
│  │  │    └─┬─┘ └─┬─┘ └─┬─┘       │        │  │  │    └─┬─┘                    │   │  │
│  │  └──────┼─────┼─────┼─────────┘        │  │  └──────┼─────────────────────┘   │  │
│  │         │     │     │                    │  │         │                         │  │
│  │         │ DRS Agent (replicating) ───────────────▶ Staging Area              │  │
│  │         │                                │  │         │                         │  │
│  │  ┌──────▼─────────────────────┐         │  │  ┌──────▼─────────────────────┐  │  │
│  │  │  Aurora Global DB (Writer) │─replication─▶│ Aurora (Read Replica)    │  │  │
│  │  └────────────────────────────┘         │  │  └────────────────────────────┘  │  │
│  │                                         │  │                                    │  │
│  │  ┌────────────────────────────┐         │  │  ┌────────────────────────────┐   │  │
│  │  │  DynamoDB Global Table     │◄──replication──▶ DynamoDB Global Table   │   │  │
│  │  └────────────────────────────┘         │  │  └────────────────────────────┘   │  │
│  │                                         │  │                                    │  │
│  └─────────────────────────────────────────┘  └────────────────────────────────────┘  │
│                                                                                       │
│  ┌─── AWS BACKUP (Compliance & Data Protection) ──────────────────────────────────┐  │
│  │  Hourly snapshots | Cross-region copy | Vault Lock | 365-day retention          │  │
│  │  Audit Manager: Compliance controls | Restore Testing: Weekly                  │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
│  ┌─── AWS DRS (Server-Level Recovery) ────────────────────────────────────────────┐  │
│  │  Continuous block-level replication | Sub-second RPO | 5-min RTO               │  │
│  │  PIT Recovery: 30-day retention | Automated post-launch validation             │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

### 🟢 Simple Setup

> **Estimated Time:** 1–2 hours  
> **Estimated Cost:** ~$10–15/day

#### Step 1: Deploy a Simple Web App

```bash
# Create a simple EC2 + RDS application
# Use the CloudFormation template:
cat << 'EOF' > simple-dr-app.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple DR Lab - Web App with Database

Parameters:
  KeyPairName:
    Type: String
  VpcId:
    Type: AWS::EC2::VPC::Id
  SubnetId:
    Type: AWS::EC2::Subnet::Id

Resources:
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server SG
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.small
      ImageId: !Sub '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}'
      KeyName: !Ref KeyPairName
      SubnetId: !Ref SubnetId
      SecurityGroupIds: [!Ref WebSecurityGroup]
      Tags:
        - Key: Name
          Value: simple-dr-web
        - Key: BackupPlan
          Value: daily
        - Key: Project
          Value: dr-lab
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y httpd mysql
          systemctl start httpd && systemctl enable httpd
          echo "<h1>DR Lab - Primary Region</h1><p>DB: simple-dr-db.xxxxx.us-east-1.rds.amazonaws.com</p>" > /var/www/html/index.html

  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: simple-dr-db
      Engine: mysql
      EngineVersion: '8.0'
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      MasterUsername: admin
      MasterUserPassword: '{{resolve:secretsmanager:dr-lab-db-password}}'
      BackupRetentionPeriod: 7
      Tags:
        - Key: BackupPlan
          Value: daily
        - Key: Project
          Value: dr-lab

Outputs:
  WebServerPublicIP:
    Value: !GetAtt WebServer.PublicIp
  DatabaseEndpoint:
    Value: !GetAtt Database.Endpoint.Address
EOF

aws cloudformation deploy \
  --template-file simple-dr-app.yaml \
  --stack-name simple-dr-app \
  --parameter-overrides KeyPairName=$KEY_PAIR VpcId=$VPC_ID SubnetId=$SUBNET_ID
```

#### Step 2: Configure AWS Backup for Daily Snapshots

```bash
# Create backup plan (reuse from Lab 1)
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "SimpleDR-DailyBackup",
    "Rules": [{
      "RuleName": "Daily",
      "TargetBackupVaultName": "dr-lab-vault",
      "ScheduleExpression": "cron(0 3 * * ? *)",
      "Lifecycle": {"DeleteAfterDays": 30}
    }]
  }'
```

#### Step 3: Set Up Route 53 Health Check & Failover

```bash
# Create a health check for the primary web server
PRIMARY_IP=$(aws cloudformation describe-stacks --stack-name simple-dr-app \
  --query 'Stacks[0].Outputs[?OutputKey==`WebServerPublicIP`].OutputValue' --output text)

HC_ID=$(aws route53 create-health-check \
  --caller-reference "simple-dr-$(date +%s)" \
  --health-check-config '{
    "IPAddress": "'$PRIMARY_IP'",
    "Port": 80,
    "Type": "HTTP",
    "ResourcePath": "/",
    "RequestInterval": 10,
    "FailureThreshold": 3
  }' \
  --query 'HealthCheck.Id' --output text)

echo "Health Check ID: $HC_ID"

# Create failover DNS records (if you have a hosted zone)
# Primary → points to us-east-1 instance
# Secondary → will point to restored instance in DR scenario
```

#### Step 4: Simulate Disaster & Restore

```bash
# Simulate disaster — stop the primary instance
aws ec2 stop-instances --instance-ids $INSTANCE_ID

# Health check will go unhealthy in ~30 seconds

# Restore from backup (as in Lab 1)
RECOVERY_POINT_ARN=$(aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "dr-lab-vault" \
  --query 'RecoveryPoints[0].RecoveryPointArn' --output text)

aws backup start-restore-job \
  --recovery-point-arn "$RECOVERY_POINT_ARN" \
  --iam-role-arn "arn:aws:iam::${ACCOUNT_ID}:role/service-role/AWSBackupDefaultServiceRole" \
  --metadata file:///tmp/restore-metadata.json

# Update Route 53 to point to restored instance (manual failover)
```

#### Verification

- [ ] Web app accessible at primary IP
- [ ] AWS Backup creating daily snapshots
- [ ] Route 53 health check monitoring primary
- [ ] After stopping instance: health check goes UNHEALTHY
- [ ] Restore from backup creates new instance
- [ ] Manual DNS update redirects traffic to restored instance

---

### 🔴 Complex Setup

> **Estimated Time:** 6–8 hours (full implementation)  
> **Estimated Cost:** ~$100–150/day (Aurora Global DB + ARC cluster + DRS + multi-region infra)

#### Step 1: Deploy Multi-Region Infrastructure

```bash
# This builds on Labs 1-4. Deploy the full stack:

# 1. Primary Region (us-east-1)
aws cloudformation deploy \
  --template-file primary-region-stack.yaml \
  --stack-name dr-primary-stack \
  --capabilities CAPABILITY_IAM \
  --region $PRIMARY_REGION

# 2. DR Region (us-west-2) 
aws cloudformation deploy \
  --template-file dr-region-stack.yaml \
  --stack-name dr-secondary-stack \
  --capabilities CAPABILITY_IAM \
  --region $DR_REGION
```

#### Step 2: Layer the DR Services

```bash
# Layer 1: AWS Backup (compliance & long-term protection)
# → Hourly backups, cross-region copy, vault lock, audit manager
# (See Lab 1 Complex Setup)

# Layer 2: AWS DRS (server-level rapid recovery)
# → Continuous replication of app tier EC2 instances
# → Sub-second RPO, 5-minute RTO
# (See Lab 2 Complex Setup)

# Layer 3: Amazon ARC (traffic management & failover orchestration)
# → Routing controls + safety rules
# → Zonal autoshift for AZ-level resilience
# → Region Switch for automated failover
# (See Lab 3 Complex Setup)

# Layer 4: AWS Resilience Hub (continuous governance)
# → Tier 1 policy enforcement
# → CI/CD gate
# → Drift detection with auto-remediation
# (See Lab 4 Complex Setup)
```

#### Step 3: Full Disaster Simulation

```bash
echo "=== DISASTER SIMULATION STARTING ==="
echo "Scenario: Complete us-east-1 region failure"
echo ""

# Step 3a: Simulate primary region failure
# (In real life: Region goes down. For testing: turn off routing control)

# Turn off primary region routing control
aws route53-recovery-cluster update-routing-control-state \
  --routing-control-arn $RC_EAST \
  --routing-control-state "Off" \
  --endpoint-url "https://$ENDPOINT"

echo "✅ Step 1: Primary region routing control OFF"
echo "   → Route 53 health check unhealthy"
echo "   → DNS failover to us-west-2 (seconds)"

# Step 3b: Verify DNS failover
sleep 10
echo ""
echo "Verifying DNS resolution..."
dig +short app.example.com
echo "Should resolve to us-west-2 ALB"

# Step 3c: Launch DRS recovery instances (for app tier)
echo ""
echo "Launching DRS recovery instances..."
aws drs start-recovery \
  --region $DR_REGION \
  --source-servers "[
    {\"sourceServerID\": \"$WEB_SERVER_ID\"},
    {\"sourceServerID\": \"$APP_SERVER_ID\"}
  ]"

echo "✅ Step 2: DRS recovery instances launching (5-15 min)"

# Step 3d: Promote Aurora secondary to primary
echo ""
echo "Promoting Aurora Global DB secondary..."
aws rds failover-global-cluster \
  --global-cluster-identifier "dr-lab-global-db" \
  --target-db-cluster-identifier "arn:aws:rds:${DR_REGION}:${ACCOUNT_ID}:cluster:dr-lab-secondary"

echo "✅ Step 3: Aurora Global DB failover initiated"

# Step 3e: Scale up DR region ASG
echo ""
echo "Scaling up DR region auto-scaling group..."
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "dr-app-asg" \
  --min-size 3 --max-size 10 --desired-capacity 3 \
  --region $DR_REGION

echo "✅ Step 4: DR region scaled to production capacity"

echo ""
echo "=== DISASTER RECOVERY COMPLETE ==="
echo "Total expected RTO: 5-15 minutes"
echo "Data loss (RPO): < 1 second (DRS) / < 1 second (Aurora async)"
```

#### Step 4: Validate Recovery

```bash
# Validate the full stack is operational in DR region
echo "=== VALIDATING DR REGION ==="

# Check ALB health
aws elbv2 describe-target-health \
  --target-group-arn $DR_TG_ARN \
  --region $DR_REGION \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Health:TargetHealth.State}'

# Check Aurora connectivity
aws rds describe-db-clusters \
  --db-cluster-identifier "dr-lab-secondary" \
  --region $DR_REGION \
  --query 'DBClusters[0].{Status:Status,Endpoint:Endpoint,ReaderEndpoint:ReaderEndpoint}'

# Check DynamoDB Global Table
aws dynamodb describe-table \
  --table-name "app-sessions" \
  --region $DR_REGION \
  --query 'Table.TableStatus'

# Run Resilience Hub assessment on DR deployment
aws resiliencehub start-app-assessment \
  --app-arn $APP_ARN \
  --app-version "release" \
  --assessment-name "post-failover-validation"
```

#### Step 5: Failback Procedure

```bash
echo "=== FAILBACK TO PRIMARY REGION ==="

# 1. Restore primary region infrastructure
# (Assumes primary region is back online)

# 2. Reverse DRS replication (DR → Primary)
aws drs reverse-replication \
  --region $PRIMARY_REGION \
  --recovery-instance-id "i-dr-recovery-xxx"

# 3. Wait for reverse replication to sync
echo "Waiting for reverse replication to reach CONTINUOUS..."
# Monitor until state = CONTINUOUS

# 4. Failover Aurora Global back to primary
aws rds failover-global-cluster \
  --global-cluster-identifier "dr-lab-global-db" \
  --target-db-cluster-identifier "arn:aws:rds:${PRIMARY_REGION}:${ACCOUNT_ID}:cluster:dr-lab-primary"

# 5. Scale up primary, scale down DR
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "primary-app-asg" \
  --min-size 3 --max-size 10 --desired-capacity 3 \
  --region $PRIMARY_REGION

aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "dr-app-asg" \
  --min-size 1 --max-size 10 --desired-capacity 1 \
  --region $DR_REGION

# 6. Restore routing controls (both ON for active-active, or just primary)
aws route53-recovery-cluster update-routing-control-states \
  --endpoint-url "https://$ENDPOINT" \
  --update-routing-control-state-entries '[
    {"RoutingControlArn": "'$RC_EAST'", "RoutingControlState": "On"},
    {"RoutingControlArn": "'$RC_WEST'", "RoutingControlState": "On"}
  ]'

echo "✅ Failback complete — traffic restored to primary region"
```

---

## Cleanup Instructions

> ⚠️ **IMPORTANT:** Always clean up resources to avoid ongoing charges!

### Lab 1 Cleanup

```bash
# Delete backup plan and selections
aws backup delete-backup-plan --backup-plan-id $PLAN_ID

# Delete recovery points (required before vault deletion)
for RP in $(aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "dr-lab-vault" \
  --query 'RecoveryPoints[*].RecoveryPointArn' --output text); do
  aws backup delete-recovery-point \
    --backup-vault-name "dr-lab-vault" \
    --recovery-point-arn "$RP"
done

# Delete backup vault
aws backup delete-backup-vault --backup-vault-name "dr-lab-vault"

# Terminate EC2 instances
aws ec2 terminate-instances --instance-ids $INSTANCE_ID $RESTORED_INSTANCE

# Delete CloudFormation stacks
aws cloudformation delete-stack --stack-name dr-lab-backup-infra

# Delete restore testing plan
aws backup delete-restore-testing-plan --restore-testing-plan-name "WeeklyRestoreTest"

# Delete audit framework
aws backup delete-framework --framework-name "DR-Compliance-Framework"
```

### Lab 2 Cleanup

```bash
# Disconnect source servers
aws drs disconnect-from-service \
  --region $DR_REGION \
  --source-server-id $SOURCE_SERVER_ID

# Delete source server from DRS
aws drs delete-source-server \
  --region $DR_REGION \
  --source-server-id $SOURCE_SERVER_ID

# Terminate any running recovery/drill instances
# (Check DRS console for active instances)

# Delete Lambda function
aws lambda delete-function --function-name DRS-AutoFailover

# Delete EventBridge rule
aws events remove-targets --rule "DRS-Failover-Trigger" --ids "DRSFailover"
aws events delete-rule --name "DRS-Failover-Trigger"

# Delete VPC endpoints and subnets in DR region
# (if created for complex setup)

# Terminate source EC2 instances
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

### Lab 3 Cleanup

```bash
# ⚠️ ARC Cluster costs $2.50/hour — delete immediately when done!

# Delete routing controls
aws route53-recovery-control-config delete-routing-control \
  --routing-control-arn $RC_EAST
aws route53-recovery-control-config delete-routing-control \
  --routing-control-arn $RC_WEST

# Delete safety rules
# (List and delete each)

# Delete control panel
aws route53-recovery-control-config delete-control-panel \
  --control-panel-arn $CONTROL_PANEL_ARN

# Delete cluster (IMPORTANT — stops $2.50/hr charges)
aws route53-recovery-control-config delete-cluster \
  --cluster-arn $CLUSTER_ARN

# Delete Route 53 health checks
aws route53 delete-health-check --health-check-id $HC_EAST
aws route53 delete-health-check --health-check-id $HC_WEST

# Delete ALB and instances
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
# Terminate all lab instances

# Delete DynamoDB Global Table
aws dynamodb delete-table --table-name "app-sessions" --region $PRIMARY_REGION

# Delete Aurora Global Database
aws rds delete-db-instance --db-instance-identifier "dr-lab-secondary-instance" --skip-final-snapshot --region $DR_REGION
aws rds delete-db-cluster --db-cluster-identifier "dr-lab-secondary" --skip-final-snapshot --region $DR_REGION
aws rds delete-db-instance --db-instance-identifier "dr-lab-primary-instance" --skip-final-snapshot --region $PRIMARY_REGION
aws rds delete-db-cluster --db-cluster-identifier "dr-lab-primary" --skip-final-snapshot --region $PRIMARY_REGION
aws rds delete-global-cluster --global-cluster-identifier "dr-lab-global-db"
```

### Lab 4 Cleanup

```bash
# Delete Resilience Hub app
aws resiliencehub delete-app --app-arn $APP_ARN

# Delete policies
aws resiliencehub delete-resiliency-policy --policy-arn $POLICY_ARN

# Delete FIS experiment templates
aws fis delete-experiment-template --id "EXT-xxxxxxxx"

# Delete CloudFormation stacks
aws cloudformation delete-stack --stack-name resilience-hub-lab-app

# Delete SNS topics
aws sns delete-topic --topic-arn $DRIFT_TOPIC_ARN

# Delete secrets
aws secretsmanager delete-secret --secret-id dr-lab-db-password --force-delete-without-recovery
```

### Lab 5 Cleanup

```bash
# Delete all resources from Labs 1-4 (above)
# Plus delete the end-to-end stacks:
aws cloudformation delete-stack --stack-name simple-dr-app
aws cloudformation delete-stack --stack-name dr-primary-stack --region $PRIMARY_REGION
aws cloudformation delete-stack --stack-name dr-secondary-stack --region $DR_REGION
```

---

## Cost Estimation Table

| Lab | Setup | Running Cost | Estimated Lab Duration | Total Cost |
|-----|-------|-------------|----------------------|------------|
| **Lab 1** | Simple | ~$0.50/hr (EC2 + EBS snapshots) | 1 hour | **~$2–5** |
| **Lab 1** | Complex | ~$2/hr (multi-region + vault) | 3 hours | **~$15–25** |
| **Lab 2** | Simple | ~$3/day (DRS replication + staging) | 2 hours | **~$5–10** |
| **Lab 2** | Complex | ~$10/day (multi-server + VPC endpoints) | 4 hours | **~$20–40** |
| **Lab 3** | Simple | ~$1/hr (ALB + instances) | 1 hour | **~$1–3** |
| **Lab 3** | Complex | ~$5/hr (ARC cluster $2.50 + Aurora Global + infra) | 5 hours | **~$50–80** |
| **Lab 4** | Simple | ~$15/month (per service assessed) | 1 hour | **~$1–2** |
| **Lab 4** | Complex | ~$50/month (multi-service + dependency) | 3 hours | **~$5–10** |
| **Lab 5** | Simple | ~$5/day | 2 hours | **~$10–15** |
| **Lab 5** | Complex | ~$100/day (all services running) | 8 hours | **~$100–150** |

### Key Cost Drivers

| Service | Primary Cost | Notes |
|---------|-------------|-------|
| **ARC Cluster** | $2.50/hour (~$60/day) | **Delete immediately after lab!** |
| **Aurora Global DB** | ~$0.58/hr (2× db.r6g.large) | Primary + secondary |
| **DRS Replication** | $0.028/server/hour | Plus staging area EBS + replication server |
| **AWS Backup** | ~$0.05/GB-month (warm) | Cross-region doubles storage cost |
| **Resilience Hub** | $15/service/month | Pro-rated; dependency discovery +$10 |

---

## Lab 6: AWS Service Screener v2 — Resilience & Best Practice Assessment

> **GitHub:** https://github.com/aws-samples/service-screener-v2  
> **Purpose:** Automated evaluation of AWS service configurations against Well-Architected best practices  
> **Cost:** Free (AWS Free Tier)  
> **Relevance:** Validates Reliability pillar configurations — backups, replication, multi-AZ, DR readiness

### How Service Screener Complements Your DR Project

```
┌─────────────────────────────────────────────────────────────────┐
│                    DR Assessment Stack                           │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: SERVICE SCREENER v2                                    │
│  → Service-level configuration checks                           │
│  → Immediate findings (misconfigurations, missing backups)       │
│  → Well-Architected integration                                 │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: AWS RESILIENCE HUB                                     │
│  → Application-level RTO/RPO assessment                          │
│  → AI failure mode analysis                                      │
│  → Dependency discovery                                          │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: WELL-ARCHITECTED TOOL                                  │
│  → Workload-level architectural review                           │
│  → Strategic improvement plan                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

### Simple Setup: Single-Region Scan in CloudShell

**⏱️ Time:** 15–25 minutes | **💰 Cost:** $0 (Free Tier)

#### Step 1: Open CloudShell

1. Log in to your AWS Console
2. Click the CloudShell icon (terminal icon) in the top navigation bar
3. Wait for the environment to initialize

#### Step 2: Install Service Screener v2

```bash
# Update Python to 3.13
sudo yum install python3.13 -y

# Create virtual environment and install
cd /tmp
python3.13 -m venv .
source bin/activate
python3.13 -m pip install --upgrade pip

# Clone the repository
rm -rf service-screener-v2
git clone https://github.com/aws-samples/service-screener-v2.git
cd service-screener-v2
pip install -r requirements.txt
python3.13 unzip_botocore_lambda_runtime.py

# Build Cloudscape UI (for beta mode)
cd cloudscape-ui
npm install
npm run build
cd ..

# Create alias for easy execution
alias screener='python3 $(pwd)/main.py'
```

#### Step 3: Run a Basic Scan (Single Region)

```bash
# Scan all services in your primary region with new Cloudscape UI
screener --regions us-east-1 --beta 1
```

**Expected output:**
```
[INFO] Starting Service Screener v2.5.0
[INFO] Scanning region: us-east-1
[INFO] Services: ALL
[INFO] Checking EC2...
[INFO] Checking S3...
[INFO] Checking RDS...
...
[INFO] Report generated: ~/service-screener-v2/output.zip
```

#### Step 4: Run Targeted DR-Relevant Services Only

```bash
# Focus on services critical for DR assessment
screener --regions us-east-1 --services ec2,rds,s3,iam,lambda,efs,dynamodb --beta 1
```

#### Step 5: Download and Review the Report

1. In CloudShell, click **Actions** → **Download file**
2. Enter path: `~/service-screener-v2/output.zip`
3. Unzip locally and open `index.html`
4. Navigate to the **Reliability** pillar findings

#### Step 6: Review Key DR-Related Findings

Focus on these categories in the report:

| Service | What to Look For |
|---------|-----------------|
| **EC2** | Multi-AZ deployment, Auto Scaling, EBS snapshot configuration |
| **RDS** | Multi-AZ enabled, automated backups, read replicas, encryption |
| **S3** | Versioning enabled, cross-region replication, lifecycle policies |
| **EFS** | Backup configuration, replication to another region |
| **DynamoDB** | Point-in-time recovery, global tables, on-demand backup |
| **IAM** | MFA enforcement, least privilege (security for DR accounts) |

#### Verification Checklist

- [ ] Service Screener installed and running in CloudShell
- [ ] Scan completed without errors
- [ ] Report downloaded and viewable in browser
- [ ] Reliability pillar findings reviewed
- [ ] At least 3 DR-related recommendations identified

#### Cleanup

```bash
# No cleanup needed — CloudShell is free and temporary
# The CloudFormation audit stack (ssv2-xxxx) auto-deletes after scan
deactivate  # Exit virtual environment
```

---

### Complex Setup: Multi-Region, Multi-Account with Well-Architected Integration

**⏱️ Time:** 1–2 hours | **💰 Cost:** $0–5

#### Step 1: Install Service Screener (same as Simple Setup)

Follow Steps 1–2 from Simple Setup above.

#### Step 2: Multi-Region Scan

```bash
# Scan both primary and DR regions
screener --regions us-east-1,us-west-2 --beta 1
```

#### Step 3: Scan ALL Regions (Comprehensive)

```bash
# Full environment scan across all regions
screener --regions ALL --beta 1
```

#### Step 4: Tag-Based Filtering for DR Resources

```bash
# Scan only production DR resources (tagged with env=production)
screener --regions us-east-1,us-west-2 --tags env=production%tier=critical --beta 1
```

#### Step 5: Well-Architected Tool Integration

Create a Well-Architected workload directly from Service Screener findings:

```bash
# Generate Well-Architected workload with milestone
screener --regions us-east-1,us-west-2 --beta 1 \
  --others '{"WA": {"region": "us-east-1", "reportName": "DR_Resilience_Assessment", "newMileStone": 1}}'
```

This creates:
- A Well-Architected workload named "DR_Resilience_Assessment"
- A milestone capturing the current state
- Reliability pillar answers pre-populated from scan results

#### Step 6: Cross-Account Scanning

For multi-account DR environments:

```bash
# First, ensure cross-account IAM role exists in target accounts
# The role needs read permissions + sts:AssumeRole trust from management account

# Scan DR account from management account
aws sts assume-role \
  --role-arn arn:aws:iam::DR_ACCOUNT_ID:role/ServiceScreenerRole \
  --role-session-name screener-scan

# Export temporary credentials and run
export AWS_ACCESS_KEY_ID=<from-assume-role>
export AWS_SECRET_ACCESS_KEY=<from-assume-role>
export AWS_SESSION_TOKEN=<from-assume-role>

screener --regions us-west-2 --beta 1
```

#### Step 7: Suppression File for Known Exceptions

Create `suppressions.json` for known acceptable findings:

```json
{
  "metadata": {
    "version": "1.0",
    "description": "DR Lab Environment Suppressions"
  },
  "suppressions": [
    {
      "service": "s3",
      "rule": "BucketReplication",
      "resource_id": ["Bucket::dev-test-bucket"]
    },
    {
      "service": "ec2",
      "rule": "EBSEncryption",
      "resource_id": ["Instance::i-0dev12345"]
    }
  ]
}
```

```bash
# Run with suppressions
screener --regions us-east-1,us-west-2 --suppress_file ./suppressions.json --beta 1
```

#### Step 8: Fast CI/CD Mode (for Pipeline Integration)

```bash
# Quick security-focused scan without custom pages (2-3 min faster)
screener --regions us-east-1 --services ec2,iam,s3,rds \
  --disable-custom-pages 1 --beta 1
```

#### Step 9: Analyze JSON Output Programmatically

```bash
# After scan, inspect raw findings
cd output/<account-id>/us-east-1/

# View raw API findings
cat api-raw.json | python3 -m json.tool | head -100

# Count findings by severity
cat api-full.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
# Analyze reliability-specific findings
for service, checks in data.items():
    print(f'{service}: {len(checks)} checks')
"
```

#### Step 10: Compare Primary vs DR Region Findings

```python
# compare_regions.py — Run locally after downloading output
import json
import os

primary_path = 'output/ACCOUNT_ID/us-east-1/api-full.json'
dr_path = 'output/ACCOUNT_ID/us-west-2/api-full.json'

with open(primary_path) as f:
    primary = json.load(f)
with open(dr_path) as f:
    dr = json.load(f)

print("=== Primary Region (us-east-1) ===")
for svc, checks in primary.items():
    print(f"  {svc}: {len(checks)} findings")

print("\n=== DR Region (us-west-2) ===")
for svc, checks in dr.items():
    print(f"  {svc}: {len(checks)} findings")

print("\n=== Gap Analysis ===")
print("Services in Primary but NOT in DR:")
for svc in set(primary.keys()) - set(dr.keys()):
    print(f"  ⚠️  {svc} — not deployed in DR region!")
```

#### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Complex Assessment Architecture                    │
│                                                                      │
│  ┌─────────────────┐     ┌─────────────────┐                       │
│  │  Management Acct │     │    DR Account    │                       │
│  │  (us-east-1)     │     │  (us-west-2)     │                       │
│  │                   │     │                   │                       │
│  │  CloudShell       │────▶│  ServiceScreener │                       │
│  │  + Service        │     │  Role (ReadOnly) │                       │
│  │    Screener v2    │     │                   │                       │
│  └────────┬──────────┘     └──────────────────┘                       │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  Well-Architected Tool                   │                         │
│  │  Workload: "DR_Resilience_Assessment"    │                         │
│  │  → Milestone 1: Baseline scan            │                         │
│  │  → Milestone 2: Post-remediation         │                         │
│  └─────────────────────────────────────────┘                         │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  Resilience Hub (complements screener)   │                         │
│  │  → Application-level RTO/RPO validation  │                         │
│  │  → Dependency discovery                  │                         │
│  └─────────────────────────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────┘
```

#### Verification Checklist

- [ ] Multi-region scan completed (us-east-1 + us-west-2)
- [ ] Well-Architected workload created with milestone
- [ ] Cross-account scan verified (if multi-account setup)
- [ ] Suppression file configured for known exceptions
- [ ] JSON findings analyzed programmatically
- [ ] Gap analysis between primary and DR regions completed
- [ ] Reliability pillar findings prioritized for remediation
- [ ] Comparison with Resilience Hub assessment documented

#### Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| `ModuleNotFoundError` | Python packages not installed | Re-run `pip install -r requirements.txt` |
| `AccessDenied` errors | Insufficient IAM permissions | Add read permissions per [sample policy](https://github.com/aws-samples/service-screener-v2) |
| CloudShell session timeout | Idle > 20 min | Re-run `source bin/activate && cd service-screener-v2` |
| Empty DR region report | No resources in us-west-2 | Deploy test resources first or check region name |
| Beta UI not loading | Cloudscape build skipped | Run `cd cloudscape-ui && npm install && npm run build` |
| WA workload creation fails | Missing permissions | Add `wellarchitected:*` to IAM policy |

#### Cleanup

```bash
# CloudShell cleanup (optional — environment resets after 120 days idle)
deactivate
cd /tmp
rm -rf service-screener-v2

# If Well-Architected workload was created and no longer needed:
aws wellarchitected delete-workload \
  --workload-id <workload-id-from-output> \
  --region us-east-1
```

---

### Service Screener Key Commands Quick Reference

| Command | Purpose |
|---------|---------|
| `screener --regions us-east-1 --beta 1` | Basic scan with Cloudscape UI |
| `screener --regions ALL --beta 1` | All regions |
| `screener --regions us-east-1 --services ec2,rds,s3` | Specific services |
| `screener --regions us-east-1 --tags env=prod` | Tag-filtered |
| `screener --regions us-east-1 --disable-custom-pages 1` | Fast mode (CI/CD) |
| `screener --regions us-east-1 --suppress_file ./sup.json` | With suppressions |
| `screener --regions us-east-1 --others '{"WA":{...}}'` | Well-Architected integration |

---

## References & Further Reading

### Official AWS Documentation

| Resource | URL |
|----------|-----|
| AWS Disaster Recovery Whitepaper | https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/ |
| Well-Architected Reliability Pillar (REL13) | https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/ |
| AWS Resilience Hub User Guide | https://docs.aws.amazon.com/resilience-hub/latest/userguide/ |
| Amazon ARC Developer Guide | https://docs.aws.amazon.com/r53recovery/latest/dg/ |
| AWS DRS User Guide | https://docs.aws.amazon.com/drs/latest/userguide/ |
| AWS Backup Developer Guide | https://docs.aws.amazon.com/aws-backup/latest/devguide/ |

### Architecture Blog Posts

| Topic | URL |
|-------|-----|
| DR Architecture Part I: Overview | https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-i-strategies-for-recovery-in-the-cloud/ |
| DR Architecture Part II: Backup & Restore | https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-ii-backup-and-restore-with-rapid-recovery/ |
| DR Architecture Part III: Pilot Light & Warm Standby | https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iii-pilot-light-and-warm-standby/ |
| DR Architecture Part IV: Multi-Site Active-Active | https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active/ |
| Cross-Region DR with AWS DRS | https://aws.amazon.com/blogs/storage/cross-region-disaster-recovery-using-aws-elastic-disaster-recovery/ |
| ARC + Step Functions DR Automation | https://aws.amazon.com/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/ |
| Next-Gen Resilience Hub (May 2026) | https://aws.amazon.com/blogs/aws/introducing-the-next-generation-of-aws-resilience-hub-for-generative-ai-based-sre-resilience-journey/ |

### Pricing Pages

| Service | URL |
|---------|-----|
| AWS Backup Pricing | https://aws.amazon.com/backup/pricing/ |
| AWS DRS Pricing | https://aws.amazon.com/disaster-recovery/pricing/ |
| Amazon ARC Pricing | https://aws.amazon.com/application-recovery-controller/pricing/ |
| AWS Resilience Hub Pricing | https://aws.amazon.com/resilience-hub/pricing/ |

### Certification Relevance

These labs cover topics tested in:
- **AWS Solutions Architect Associate (SAA-C03)** — DR strategies, multi-region architectures
- **AWS Solutions Architect Professional (SAP-C02)** — Complex DR, multi-account, failover orchestration
- **AWS SysOps Administrator Associate (SOA-C02)** — Backup management, monitoring, automation
- **AWS DevOps Engineer Professional (DOP-C02)** — CI/CD resilience gates, automation
- **AWS Security Specialty (SCS-C02)** — Vault lock, encryption, cross-account security

---

> **Next Steps:**  
> 1. Start with Lab 1 Simple Setup to understand basics  
> 2. Progress through Labs 2–4 Simple Setups  
> 3. Attempt Lab 5 Simple Setup (integration)  
> 4. Then tackle Complex Setups for production-ready architectures  
> 5. Use Lab 5 Complex as your capstone project

---

*Last updated: July 2026 | All commands verified against AWS CLI v2.x*
