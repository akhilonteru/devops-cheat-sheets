# AWS CLI Cheat Sheet

> AWS Command Line Interface reference — configuration, S3, EC2, IAM, EKS, Lambda, CloudFormation, and query/output patterns.

---

## Table of Contents

- [Setup & Configuration](#1-setup--configuration)
- [Global Options, Query & Output](#2-global-options-query--output)
- [STS — Identity](#3-sts--identity)
- [S3](#4-s3)
- [EC2](#5-ec2)
- [IAM](#6-iam)
- [EKS](#7-eks)
- [Lambda](#8-lambda)
- [CloudFormation](#9-cloudformation)
- [CloudWatch Logs](#10-cloudwatch-logs)
- [Useful One-Liners](#11-useful-one-liners)

---

## 1. Setup & Configuration

```
aws configure                        # Interactive: key, secret, region, output
aws configure --profile prod         # Named profile
aws configure list                   # Show current config
aws configure list-profiles

# ~/.aws/credentials
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
[prod]
aws_access_key_id = AKIA...

# ~/.aws/config
[default]
region = us-east-1
output = json
[profile prod]
region = eu-west-1
```

```
# Use a profile in any command
aws s3 ls --profile prod
export AWS_PROFILE=prod              # Or set env var
export AWS_DEFAULT_REGION=us-east-1
export AWS_REGION=us-east-1
aws sts get-caller-identity          # Verify who you are
```

## 2. Global Options, Query & Output

```
--region us-west-2                   # Override region
--profile prod                       # Named profile
--output json|table|text|yaml        # Output format
--query 'Reservations[*].Instances[*].InstanceId'   # JMESPath filter
--no-paginate                        # Disable auto-pagination
--max-items 100                      # Limit returned items
--cli-read-timeout 60
--cli-connect-timeout 10
--debug                              # Full debug logging
```

**JMESPath examples:**
```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' \
  --output table

aws s3api list-buckets --query 'Buckets[*].Name'
aws iam list-users --query 'Users[*].[UserName,CreateDate]' --output text
```

## 3. STS — Identity

```
aws sts get-caller-identity
# { "UserId": "...", "Account": "123456789012", "Arn": "arn:aws:iam::..." }

aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/DeployRole \
  --role-session-name ci-deploy \
  --duration-seconds 3600
# Returns temporary AccessKeyId/SecretAccessKey/SessionToken
```

## 4. S3

```bash
aws s3 ls                                    # List buckets
aws s3 ls s3://my-bucket
aws s3 ls s3://my-bucket --recursive --human-readable --summarize
aws s3 mb s3://my-new-bucket --region eu-west-1     # Make bucket

aws s3 cp file.txt s3://my-bucket/path/
aws s3 cp s3://my-bucket/file.txt ./
aws s3 cp ./dir s3://my-bucket/dir --recursive

aws s3 sync ./local s3://my-bucket/backup/          # Two-way aware sync
aws s3 sync s3://bucket-a/ s3://bucket-b/ --delete  # Mirror (incl. deletes!)
aws s3 sync ./dist s3://my-site/ --exclude "*.map" --include "assets/*"

aws s3 rm s3://my-bucket/file.txt
aws s3 rm s3://my-bucket/dir --recursive
aws s3 rb s3://my-bucket --force                    # Remove bucket

aws s3 presign s3://my-bucket/private.pdf --expires-in 3600
aws s3api head-object --bucket my-bucket --key file.txt
aws s3api create-bucket --bucket my-bucket --create-bucket-configuration LocationConstraint=eu-west-1
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled
```

## 5. EC2

```bash
# List instances
aws ec2 describe-instances --instance-ids i-0abcd1234
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" "Name=tag:Role,Values=web" \
  --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,InstanceType]' \
  --output table

# Launch
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --key-name mykey \
  --security-group-ids sg-01234 \
  --subnet-id subnet-56789 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-1}]' \
  --user-data file://user-data.sh
aws ec2 describe-instance-status --instance-ids i-0abcd1234

# Control
aws ec2 start-instances --instance-ids i-0abcd1234
aws ec2 stop-instances --instance-ids i-0abcd1234
aws ec2 reboot-instances --instance-ids i-0abcd1234
aws ec2 terminate-instances --instance-ids i-0abcd1234

# Key pairs & AMIs
aws ec2 create-key-pair --key-name deploy --query 'KeyMaterial' --output text > deploy.pem
chmod 400 deploy.pem
aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" \
  --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' --output text
aws ec2 create-image --instance-id i-0abcd1234 --name web-backup-$(date +%F)
```

## 6. IAM

```
aws iam list-users
aws iam get-user
aws iam list-roles
aws iam list-policies --scope AWS
aws iam list-attached-user-policies --user-name deploy
aws iam create-user --user-name ci-bot
aws iam create-access-key --user-name ci-bot          # Save the secret — shown once!
aws iam attach-user-policy --user-name ci-bot \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-access-key --user-name ci-bot --access-key-id AKIA...
aws iam update-access-key --user-name ci-bot --access-key-id AKIA... --status Inactive

# Assume-role trust (policy document files)
aws iam create-role --role-name AppRole --assume-role-policy-document file://trust-policy.json
aws iam put-role-policy --role-name AppRole --policy-name AppAccess --policy-document file://policy.json
```

## 7. EKS

```
aws eks list-clusters
aws eks describe-cluster --name prod --query 'cluster.status'
aws eks update-kubeconfig --name prod --region us-east-1 [--alias prod-eks]
aws eks update-kubeconfig --name prod --kubeconfig ~/.kube/config
kubectl get nodes
aws eks list-nodegroups --cluster-name prod
aws eks update-cluster-version --name prod --kubernetes-version 1.30
```

## 8. Lambda

```
aws lambda list-functions
aws lambda get-function --function-name my-func --query 'Configuration.{Runtime:Runtime,Role:Role}'
aws lambda invoke --function-name my-func out.json
aws lambda invoke --function-name my-func out.json \
  --payload '{"key":"value"}' --cli-binary-format raw-in-base64-out
cat out.json
aws lambda update-function-code --function-name my-func --zip-file fileb://function.zip
aws lambda update-function-configuration --function-name my-func --timeout 30 --memory-size 512
aws lambda publish-version --function-name my-func
aws lambda create-alias --function-name my-func --name prod --function-version 3
aws lambda list-layers
aws lambda delete-function --function-name my-func
```

## 9. CloudFormation

```bash
aws cloudformation validate-template --template-body file://template.yaml
aws cloudformation deploy \
  --template-file main.yaml \
  --stack-name my-app \
  --parameter-overrides Env=prod InstanceType=t3.small \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --tags Project=MyApp

aws cloudformation describe-stacks --stack-name my-app \
  --query 'Stacks[0].Outputs' --output table
aws cloudformation describe-stack-events --stack-name my-app   # Debug failures
aws cloudformation update-stack --stack-name my-app --use-previous-template \
  --parameters ParameterKey=Env,ParameterValue=staging
aws cloudformation delete-stack --stack-name my-app
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE
```

## 10. CloudWatch Logs

```
aws logs describe-log-groups
aws logs tail /aws/lambda/my-func --follow
aws logs tail /aws/eks/prod/cluster --since 1h
aws logs filter-log-events --log-group-name /aws/lambda/my-func \
  --filter-pattern "ERROR" --start-time $(date -d '1 hour ago' +%s000)
aws logs create-log-group --log-group-name /my/app
aws logs put-retention-policy --log-group-name /my/app --retention-in-days 14
```

## 11. Useful One-Liners

```bash
# All running instances with their Name tags
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[*].Instances[*].[Tags[?Key==`Name`]|[0].Value,InstanceId,PublicIpAddress]' \
  --output table

# Total size of a bucket
aws s3 ls s3://my-bucket --recursive --summarize | tail -1

# Find untagged resources (compliance check)
aws ec2 describe-instances --query \
  'Reservations[*].Instances[?!not_null(Tags[?Key==`Environment`])].[InstanceId]'

# Expiring certificates in ACM
aws acm list-certificates --query 'CertificateSummaryList[?Status==`EXPIRED`]'

# SS
M port forwarding (SSH through Systems Manager)
aws ssm start-session --target i-0abcd1234
```

---
