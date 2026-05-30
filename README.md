# ✅ AWS Config: Compliance & Governance

> **AWS Lab Project** | AWS Config · Managed Rules · S3 · EC2 · Compliance Evaluation · Resource Inventory

---

## 📋 Project Overview

This project enables AWS Config to continuously monitor and evaluate the configuration of AWS resources against defined compliance rules. AWS Config is the foundation of cloud governance — it records every configuration change and evaluates whether resources comply with your security and operational standards.

This is the tool that answers: "Is my environment configured the way I think it is?"

---

## 🏗️ Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       AWS Config                                │
│                                                                 │
│  RECORDING                   EVALUATION                        │
│  ┌───────────────┐           ┌───────────────────────────────┐  │
│  │ Configuration │           │      AWS Managed Rules        │  │
│  │ Recorder      │──records─▶│                               │  │
│  │               │           │  ✅ s3-bucket-public-read-    │  │
│  │ Tracks ALL    │           │     prohibited                │  │
│  │ resource      │           │  ✅ s3-bucket-versioning-     │  │
│  │ changes       │           │     enabled                   │  │
│  └───────────────┘           │  ✅ ec2-instance-no-public-ip │  │
│                              │  ✅ restricted-ssh            │  │
│  ┌───────────────┐           │  ✅ iam-password-policy       │  │
│  │ Delivery      │           └───────────────────────────────┘  │
│  │ Channel       │                        │                     │
│  │               │           ┌────────────▼──────────────────┐  │
│  │ Snapshots     │           │      Compliance Results       │  │
│  │ → S3          │           │                               │  │
│  │ Changes       │           │  Resource A: COMPLIANT ✅     │  │
│  │ → SNS         │           │  Resource B: NON_COMPLIANT ❌  │  │
│  └───────────────┘           │  Resource C: COMPLIANT ✅     │  │
│                              └───────────────────────────────┘  │
│                                           │                     │
│                              ┌────────────▼──────────────────┐  │
│                              │    Resource Timeline           │  │
│                              │                               │  │
│                              │  Who changed it?              │  │
│                              │  What was the old config?     │  │
│                              │  When did it change?          │  │
│                              └───────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 AWS Services Used

| Service | Purpose |
|---|---|
| **AWS Config** | Records all resource configurations and changes |
| **Config Rules** | Evaluates whether resources comply with defined standards |
| **S3** | Stores Config snapshots and compliance history |
| **SNS** | Notifies when compliance changes occur |
| **IAM** | Config service role to read resource configurations |

---

## 🚀 Step-by-Step Build

### Step 1: Create S3 Bucket for Config Delivery

```bash
CONFIG_BUCKET="aws-config-delivery-$(date +%s)"

aws s3api create-bucket \
  --bucket $CONFIG_BUCKET \
  --region us-east-1

# Bucket policy: allow Config service to write to this bucket
aws s3api put-bucket-policy \
  --bucket $CONFIG_BUCKET \
  --policy file://configs/config-bucket-policy.json

echo "Config delivery bucket: $CONFIG_BUCKET"
```

### Step 2: Create IAM Role for AWS Config

```bash
# Config needs permission to read all resource configurations
aws iam create-role \
  --role-name "aws-config-role" \
  --assume-role-policy-document file://iam/config-trust-policy.json

aws iam attach-role-policy \
  --role-name "aws-config-role" \
  --policy-arn "arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"

CONFIG_ROLE_ARN=$(aws iam get-role \
  --role-name aws-config-role \
  --query 'Role.Arn' --output text)

echo "Config role ARN: $CONFIG_ROLE_ARN"
```

### Step 3: Enable Configuration Recorder

```bash
aws configservice put-configuration-recorder \
  --configuration-recorder \
    name=default,roleARN=$CONFIG_ROLE_ARN \
  --recording-group \
    allSupported=true,includeGlobalResourceTypes=true

echo "✅ Configuration recorder created"
```

### Step 4: Create Delivery Channel

```bash
aws configservice put-delivery-channel \
  --delivery-channel \
    name=default,s3BucketName=$CONFIG_BUCKET

echo "✅ Delivery channel set to: $CONFIG_BUCKET"
```

### Step 5: Start the Recorder

```bash
aws configservice start-configuration-recorder \
  --configuration-recorder-name default

echo "✅ AWS Config is now recording all resource changes"
```

### Step 6: Create an S3 Bucket to Evaluate

```bash
# Create a test bucket that Config will discover and evaluate
aws s3api create-bucket \
  --bucket "test-compliance-bucket-$(date +%s)" \
  --region us-east-1

echo "Test bucket created — Config will discover it within a few minutes"
```

### Step 7: Add AWS Managed Compliance Rules

```bash
# Rule 1: S3 buckets must not allow public read access
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
  }'

# Rule 2: S3 buckets must have versioning enabled
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-versioning-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_VERSIONING_ENABLED"
    }
  }'

# Rule 3: EC2 instances should not have public IPs
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "ec2-instance-no-public-ip",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "EC2_INSTANCE_NO_PUBLIC_IP"
    }
  }'

# Rule 4: Security groups should not allow unrestricted SSH
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "restricted-ssh",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "INCOMING_SSH_DISABLED"
    }
  }'

echo "✅ 4 managed compliance rules created"
```

### Step 8: Check Compliance Results

```bash
# View compliance status for all rules
aws configservice describe-compliance-by-config-rule \
  --query 'ComplianceByConfigRules[*].{Rule:ConfigRuleName,Compliance:Compliance.ComplianceType}' \
  --output table

# Get details on non-compliant resources for a specific rule
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name "s3-bucket-public-read-prohibited" \
  --compliance-types NON_COMPLIANT \
  --query 'EvaluationResults[*].{Resource:EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId,Status:ComplianceType}' \
  --output table
```

---

## 📄 IAM Trust Policy for Config

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "config.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## 📁 Repository Structure

```
08-aws-config-compliance/
├── README.md
├── iam/
│   └── config-trust-policy.json
├── configs/
│   └── config-bucket-policy.json   # S3 bucket policy for Config delivery
└── scripts/
    ├── 01-create-delivery-bucket.sh
    ├── 02-create-iam-role.sh
    ├── 03-enable-config-recorder.sh
    ├── 04-add-managed-rules.sh
    └── 05-check-compliance.sh
```

---

## 💡 Key Takeaways

1. **AWS Config ≠ AWS CloudTrail** — Config records *what your resources look like*; CloudTrail records *who made API calls*. Both are essential
2. **Managed rules are point-in-time evaluations** — Config runs the rule when a resource changes, plus on-demand; it doesn't evaluate in real time
3. **Config has a cost** — you're charged per configuration item recorded and per active rule evaluation. Filter to only what you need in production
4. **Non-compliant doesn't mean blocked** — Config evaluates and reports; use AWS Organizations SCPs or IAM policies to actually prevent non-compliant actions
5. **Resource timeline is invaluable for incident response** — Config can tell you exactly what changed and when before an outage occurred

---

## 🔗 Related Projects

- [04 - IAM Security: EC2 to S3](../04-iam-security-ec2-s3)
- [01 - Secure VPC Architecture](../01-secure-vpc-architecture)

---

*Built by Abdul Nazir Sadaat | AWS Cloud Engineer | Rancho Cordova, CA*
