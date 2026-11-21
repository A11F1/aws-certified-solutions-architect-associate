# Week 8 Main Lab: Resilient, Encrypted, and Governed Workload

## Objective

Take a small workload and make it recoverable, encrypted, private, and governed. You will encrypt everything with a customer managed KMS key, keep credentials in Secrets Manager, protect an EC2 instance with an AWS Backup plan that copies recovery points to a second Region, replicate an S3 bucket cross-Region, reach Secrets Manager privately through a VPC interface endpoint (AWS PrivateLink), and add governance guardrails (SCP and tag policy if you have an AWS Organization, or a permissions boundary plus a Budget if you do not). You will perform a real restore and measure your actual RTO and RPO, then place those numbers in a DR strategy comparison table.

Target time: 100 to 120 minutes (backup and copy jobs include waiting time). Region: `us-east-1` (primary), `us-west-2` (DR). Personal AWS account.

## Modules Covered

- [22 - Disaster Recovery and Continuity](../22-Recuperacao-de-Desastres-e-Continuidade/README.md): RTO/RPO, AWS Backup, cross-Region copy, S3 replication.
- [23 - AWS Organizations, Governance, and Cost](../23-AWS-Organizations-Governanca-e-Custos/README.md): SCPs, tag policies, Budgets.
- [24 - Advanced Networking and Hybrid Connectivity](../24-Redes-Avancadas-e-Conectividade-Hibrida/README.md): VPC interface endpoints (PrivateLink).
- [25 - KMS Encryption and Secrets Management](../25-Criptografia-KMS-e-Gestao-de-Segredos/README.md): customer managed keys, key policies, Secrets Manager.

## Scenario and Architecture

A small fintech runs an application server that stores files in S3 and reads its database credentials from Secrets Manager. Auditors require: encryption with a key the company controls, no credential retrieval over the public internet, a tested recovery plan with a documented RTO of under 60 minutes and an RPO of under 24 hours, a second Region copy of the data, and guardrails so nobody can delete the key or launch resources outside approved Regions.

```
                         us-east-1 (primary)                                     us-west-2 (DR)
 +---------------------------------------------------------------+      +-------------------------------+
 | VPC 10.80.0.0/16 (no internet gateway, no NAT)                |      |                               |
 |  private subnet A                                             |      |  KMS key (west)               |
 |   +------------------+     +-----------------------------+    |      |  Backup vault (west) <--------+--- copy job
 |   | EC2 t3.micro     |     | Lambda "secret-reader"      |    |      |  S3 replica bucket <----------+--- CRR
 |   | encrypted EBS    |     | (in VPC) --HTTPS--> Interface|   |      |  (SSE-KMS, west key)          |
 |   +--------+---------+     |  Endpoint (secretsmanager)   |   |      +-------------------------------+
 |            |               +--------------+--------------+    |
 |            | AWS Backup                    | PrivateLink       |
 |            v                               v                   |
 |   Backup vault (east) ---- plan: daily + copy to west   Secrets Manager secret (encrypted w/ CMK)
 |                                                                |
 |   S3 bucket (SSE-KMS, versioned) -- replication rule ---------------> us-west-2 bucket
 +---------------------------------------------------------------+
        KMS CMK alias/week8-key (rotation on)

 Governance:  Organization: SCP (deny key deletion + deny non-approved Regions) + tag policy
              Single account: IAM permissions boundary + AWS Budget
```

## Services Used

- AWS KMS (customer managed keys in two Regions)
- AWS Secrets Manager
- AWS Backup (vaults, plan, selection, copy job, restore job)
- Amazon EC2 and EBS, Amazon S3 (versioning, cross-Region replication)
- Amazon VPC (private subnet, interface endpoint, security groups) and AWS PrivateLink
- AWS Lambda (as a private test client)
- AWS Organizations (SCP, tag policy) or IAM permissions boundary plus AWS Budgets
- AWS IAM

## Prerequisites

- Personal AWS account, admin-level IAM identity, AWS CLI v2, region `us-east-1` for everything unless noted.
- Optional: an AWS Organization where you can create an OU with a throwaway member account. If you only have a single standalone account, use the fallback in Step 9. Never attach an experimental SCP to the organization root or to production accounts. SCPs do not affect the management account.
- A real email address for the Budget alert.
- Two to three hours is the safe window because restore jobs can take several minutes. Finish and clean up the same day.

```bash
export AWS_REGION=us-east-1
export DR_REGION=us-west-2
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SUFFIX=<your-initials>-$RANDOM
export SRC_BUCKET=w8-src-$SUFFIX
export DST_BUCKET=w8-dst-$SUFFIX
```

## Expected Cost

Approximate us-east-1 / us-west-2 figures; confirm on the AWS pricing pages (KMS, Secrets Manager, AWS Backup, VPC/PrivateLink, EC2, S3, Budgets).

- WARNING, hourly-billed: one VPC interface endpoint is roughly $0.01 per hour per AZ plus data processing (about $7 to $8 per month if left running, per AZ). One `t3.micro` is roughly $0.0104 per hour. Delete both the same day. Do NOT add a NAT gateway (roughly $0.045 per hour plus data); this design intentionally has none.
- KMS: about $1 per customer managed key per month (prorated hourly), plus a small per-request charge. Two keys, deleted at the end, cost a few cents. Keys scheduled for deletion are not billed after they enter the pending deletion state, but verify.
- Secrets Manager: about $0.40 per secret per month (prorated) plus per-10,000-call API charges. A 30-day recovery window applies unless you force-delete.
- AWS Backup: charged per GB-month of backup storage (EBS snapshots are billed at snapshot rates), plus cross-Region data transfer. With an 8 GiB volume this is cents. Restored resources (a second EC2 volume or instance) bill as normal.
- S3 replication: normal request and storage charges plus inter-Region transfer for a handful of tiny objects (negligible). AWS KMS requests for replication add small charges.
- Budgets: the first two action-free budgets per account are free (confirm).

A well-run session costs roughly $1 or less. Left running for a month the interface endpoint, instance, and keys would cost tens of dollars.

## Step by Step

### Step 1: Create customer managed KMS keys in both Regions

```bash
KEY_E=$(aws kms create-key --region $AWS_REGION \
  --description "week8 primary key" --query KeyMetadata.KeyId --output text)
aws kms create-alias --region $AWS_REGION --alias-name alias/week8-key --target-key-id $KEY_E
aws kms enable-key-rotation --region $AWS_REGION --key-id $KEY_E

KEY_W=$(aws kms create-key --region $DR_REGION \
  --description "week8 DR key" --query KeyMetadata.KeyId --output text)
aws kms create-alias --region $DR_REGION --alias-name alias/week8-key-dr --target-key-id $KEY_W
aws kms enable-key-rotation --region $DR_REGION --key-id $KEY_W

KEY_E_ARN=arn:aws:kms:$AWS_REGION:$ACCOUNT_ID:key/$KEY_E
KEY_W_ARN=arn:aws:kms:$DR_REGION:$ACCOUNT_ID:key/$KEY_W
aws kms get-key-rotation-status --key-id $KEY_E --query KeyRotationEnabled
```

Console: KMS > Customer managed keys. KMS keys are Regional, which is why a DR copy needs its own key in the destination Region (or a multi-Region key; see Stretch Goals). The default key policy created by the CLI lets IAM policies in your account grant access to the key. In production, write an explicit key policy that separates key administrators from key users.

### Step 2: Network: a private VPC with no internet path

```bash
VPC=$(aws ec2 create-vpc --cidr-block 10.80.0.0/16 --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames
aws ec2 create-tags --resources $VPC --tags Key=Name,Value=week8-vpc Key=Project,Value=week8

SUBNET_A=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.80.1.0/24 \
  --availability-zone us-east-1a --query Subnet.SubnetId --output text)

# Security group for the endpoint (443 from inside the VPC)
SG_EP=$(aws ec2 create-security-group --group-name w8-endpoint-sg \
  --description "Allow HTTPS from VPC to endpoint" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_EP --protocol tcp --port 443 --cidr 10.80.0.0/16

# Security group for the Lambda client (no inbound; default egress)
SG_FN=$(aws ec2 create-security-group --group-name w8-lambda-sg \
  --description "Lambda test client" --vpc-id $VPC --query GroupId --output text)
```

Private DNS on the interface endpoint requires both DNS support and DNS hostnames enabled on the VPC, which the two `modify-vpc-attribute` calls do. The subnet uses the VPC main route table, which only has the local route, so there is no internet path.

### Step 3: Secrets Manager secret encrypted with your CMK

```bash
PW=$(aws secretsmanager get-random-password --password-length 24 \
  --exclude-punctuation --query RandomPassword --output text)

aws secretsmanager create-secret --name week8/app/db \
  --description "Demo DB credentials" \
  --kms-key-id alias/week8-key \
  --secret-string "{\"username\":\"appuser\",\"password\":\"$PW\"}" \
  --tags Key=Project,Value=week8

unset PW
aws secretsmanager describe-secret --secret-id week8/app/db --query '[Name,KmsKeyId]'
```

The password is generated locally and never written to a file. Never put real credentials into lab commands or shell history. Secrets Manager adds automatic rotation through Lambda for supported databases; this lab uses a static secret for simplicity.

### Step 4: PrivateLink: interface endpoint for Secrets Manager

```bash
EP=$(aws ec2 create-vpc-endpoint --vpc-id $VPC --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids $SUBNET_A --security-group-ids $SG_EP \
  --private-dns-enabled \
  --query VpcEndpoint.VpcEndpointId --output text)

aws ec2 describe-vpc-endpoints --vpc-endpoint-ids $EP --query 'VpcEndpoints[0].State'
```

Wait for `available`. This uses one subnet to save money; production uses at least two AZs. Console: VPC > Endpoints.

### Step 5: A private client: Lambda inside the VPC reads the secret

```bash
cat > trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name w8-secret-reader --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name w8-secret-reader \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole

SECRET_ARN=$(aws secretsmanager describe-secret --secret-id week8/app/db --query ARN --output text)
cat > reader-policy.json <<EOF
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":"secretsmanager:GetSecretValue","Resource":"$SECRET_ARN"},
 {"Effect":"Allow","Action":"kms:Decrypt","Resource":"$KEY_E_ARN"}]}
EOF
aws iam put-role-policy --role-name w8-secret-reader --policy-name read-secret \
  --policy-document file://reader-policy.json
```

Function code (`reader.py`). It does not print the secret value, only whether retrieval worked:

```python
import boto3
from botocore.config import Config

client = boto3.client("secretsmanager",
    config=Config(connect_timeout=3, read_timeout=5, retries={"max_attempts": 0}))

def handler(event, context):
    try:
        v = client.get_secret_value(SecretId="week8/app/db")
        keys = list(__import__("json").loads(v["SecretString"]).keys())
        return {"ok": True, "keys": keys}
    except Exception as e:
        return {"ok": False, "error": type(e).__name__}
```

```bash
zip reader.zip reader.py
sleep 10
aws lambda create-function --function-name w8-secret-reader \
  --runtime python3.12 --handler reader.handler \
  --role arn:aws:iam::$ACCOUNT_ID:role/w8-secret-reader \
  --zip-file fileb://reader.zip --timeout 30 \
  --vpc-config SubnetIds=$SUBNET_A,SecurityGroupIds=$SG_FN

aws lambda wait function-active-v2 --function-name w8-secret-reader
aws lambda invoke --function-name w8-secret-reader out.json >/dev/null && cat out.json
```

Expected: `{"ok": true, "keys": ["username", "password"]}`. The VPC has no internet gateway or NAT, so the only way this works is through the endpoint with private DNS (the public Secrets Manager hostname resolves to the endpoint's private IP).

### Step 6: Workload instance with an encrypted volume

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)

INSTANCE=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --subnet-id $SUBNET_A --security-group-ids $SG_FN \
  --block-device-mappings "DeviceName=/dev/xvda,Ebs={VolumeSize=8,VolumeType=gp3,Encrypted=true,KmsKeyId=$KEY_E,DeleteOnTermination=true}" \
  --user-data '#!/bin/bash
echo "week8 marker $(date -u +%FT%TZ)" > /var/tmp/marker.txt' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Project,Value=week8},{Key=Name,Value=week8-app}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INSTANCE
```

The instance has no public IP and no SSH access; it is the "protected resource", and a recovery point of its volume proves the DR workflow. If an SCP or account setting forces IMDSv2, add `--metadata-options HttpTokens=required`.

### Step 7: S3 cross-Region replication with SSE-KMS

```bash
aws s3api create-bucket --bucket $SRC_BUCKET --region $AWS_REGION
aws s3api create-bucket --bucket $DST_BUCKET --region $DR_REGION \
  --create-bucket-configuration LocationConstraint=$DR_REGION

aws s3api put-bucket-versioning --bucket $SRC_BUCKET --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket $DST_BUCKET --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption --bucket $SRC_BUCKET --server-side-encryption-configuration \
  "{\"Rules\":[{\"ApplyServerSideEncryptionByDefault\":{\"SSEAlgorithm\":\"aws:kms\",\"KMSMasterKeyID\":\"$KEY_E_ARN\"},\"BucketKeyEnabled\":true}]}"
aws s3api put-bucket-encryption --bucket $DST_BUCKET --server-side-encryption-configuration \
  "{\"Rules\":[{\"ApplyServerSideEncryptionByDefault\":{\"SSEAlgorithm\":\"aws:kms\",\"KMSMasterKeyID\":\"$KEY_W_ARN\"},\"BucketKeyEnabled\":true}]}"
for B in $SRC_BUCKET $DST_BUCKET; do
  aws s3api put-public-access-block --bucket $B --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
done
```

Replication role and rule:

```bash
cat > s3-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"s3.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name w8-s3-replication --assume-role-policy-document file://s3-trust.json

cat > repl-policy.json <<EOF
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":["s3:GetReplicationConfiguration","s3:ListBucket"],
  "Resource":"arn:aws:s3:::$SRC_BUCKET"},
 {"Effect":"Allow","Action":["s3:GetObjectVersionForReplication","s3:GetObjectVersionAcl","s3:GetObjectVersionTagging"],
  "Resource":"arn:aws:s3:::$SRC_BUCKET/*"},
 {"Effect":"Allow","Action":["s3:ReplicateObject","s3:ReplicateDelete","s3:ReplicateTags"],
  "Resource":"arn:aws:s3:::$DST_BUCKET/*"},
 {"Effect":"Allow","Action":"kms:Decrypt","Resource":"$KEY_E_ARN",
  "Condition":{"StringLike":{"kms:ViaService":"s3.us-east-1.amazonaws.com"}}},
 {"Effect":"Allow","Action":"kms:Encrypt","Resource":"$KEY_W_ARN",
  "Condition":{"StringLike":{"kms:ViaService":"s3.us-west-2.amazonaws.com"}}}]}
EOF
aws iam put-role-policy --role-name w8-s3-replication --policy-name repl --policy-document file://repl-policy.json

cat > replication.json <<EOF
{"Role":"arn:aws:iam::$ACCOUNT_ID:role/w8-s3-replication",
 "Rules":[{"ID":"all-to-dr","Status":"Enabled","Priority":1,"Filter":{},
  "DeleteMarkerReplication":{"Status":"Disabled"},
  "SourceSelectionCriteria":{"SseKmsEncryptedObjects":{"Status":"Enabled"}},
  "Destination":{"Bucket":"arn:aws:s3:::$DST_BUCKET",
    "EncryptionConfiguration":{"ReplicaKmsKeyID":"$KEY_W_ARN"}}}]}
EOF
sleep 10
aws s3api put-bucket-replication --bucket $SRC_BUCKET --replication-configuration file://replication.json
```

Measure replication lag, which is your S3 RPO evidence. Replication applies only to objects written after the rule exists. Without S3 Replication Time Control (an extra-cost option with a 15-minute SLA), there is no time guarantee.

```bash
date -u +%T
aws s3api put-object --bucket $SRC_BUCKET --key docs/contract-001.txt --body reader.py
until aws s3api head-object --bucket $DST_BUCKET --key docs/contract-001.txt --region $DR_REGION >/dev/null 2>&1; do sleep 5; done
date -u +%T
aws s3api head-object --bucket $SRC_BUCKET --key docs/contract-001.txt --query ReplicationStatus
```

Record the elapsed time as your measured S3 RPO.

### Step 8: AWS Backup: vaults, plan with cross-Region copy, and a recovery point

Create vaults, each encrypted with the key of its own Region:

```bash
aws backup create-backup-vault --region $AWS_REGION --backup-vault-name w8-vault \
  --encryption-key-arn $KEY_E_ARN
aws backup create-backup-vault --region $DR_REGION --backup-vault-name w8-vault-dr \
  --encryption-key-arn $KEY_W_ARN
```

Create the service role AWS Backup uses (the console creates `AWSBackupDefaultServiceRole` automatically on first use; the CLI does not):

```bash
cat > backup-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"backup.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name w8-backup-role --assume-role-policy-document file://backup-trust.json
aws iam attach-role-policy --role-name w8-backup-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup
aws iam attach-role-policy --role-name w8-backup-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores
```

The vault encrypted with a CMK needs the role to use the key. Grant it:

```bash
cat > backup-kms.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Action":["kms:Decrypt","kms:Encrypt","kms:GenerateDataKey*","kms:ReEncrypt*","kms:DescribeKey","kms:CreateGrant"],
 "Resource":["$KEY_E_ARN","$KEY_W_ARN"]}]}
EOF
aws iam put-role-policy --role-name w8-backup-role --policy-name kms-use --policy-document file://backup-kms.json
```

Now the plan (daily rule, copy to the DR vault) and a tag-based selection:

```bash
cat > plan.json <<EOF
{"BackupPlanName":"w8-plan","Rules":[{
  "RuleName":"daily-with-dr-copy",
  "TargetBackupVaultName":"w8-vault",
  "ScheduleExpression":"cron(0 5 * * ? *)",
  "StartWindowMinutes":60,
  "CompletionWindowMinutes":180,
  "Lifecycle":{"DeleteAfterDays":7},
  "CopyActions":[{"DestinationBackupVaultArn":"arn:aws:backup:$DR_REGION:$ACCOUNT_ID:backup-vault:w8-vault-dr",
                  "Lifecycle":{"DeleteAfterDays":7}}]}]}
EOF
PLAN_ID=$(aws backup create-backup-plan --backup-plan file://plan.json --query BackupPlanId --output text)

cat > selection.json <<EOF
{"SelectionName":"w8-tagged","IamRoleArn":"arn:aws:iam::$ACCOUNT_ID:role/w8-backup-role",
 "ListOfTags":[{"ConditionType":"STRINGEQUALS","ConditionKey":"Project","ConditionValue":"week8"}]}
EOF
aws backup create-backup-selection --backup-plan-id $PLAN_ID --backup-selection file://selection.json
```

The plan runs at 05:00 UTC daily, which is your worst-case RPO of roughly 24 hours for this tier. To avoid waiting, start an on-demand backup now and copy it to the DR Region:

```bash
INSTANCE_ARN=arn:aws:ec2:$AWS_REGION:$ACCOUNT_ID:instance/$INSTANCE
JOB=$(aws backup start-backup-job --backup-vault-name w8-vault \
  --resource-arn $INSTANCE_ARN \
  --iam-role-arn arn:aws:iam::$ACCOUNT_ID:role/w8-backup-role \
  --lifecycle DeleteAfterDays=7 --query BackupJobId --output text)

aws backup describe-backup-job --backup-job-id $JOB --query '[State,RecoveryPointArn]'
```

Repeat until `COMPLETED` (typically a few minutes for an 8 GiB volume). Then copy:

```bash
RP=$(aws backup describe-backup-job --backup-job-id $JOB --query RecoveryPointArn --output text)
aws backup start-copy-job --recovery-point-arn $RP \
  --source-backup-vault-name w8-vault \
  --destination-backup-vault-arn arn:aws:backup:$DR_REGION:$ACCOUNT_ID:backup-vault:w8-vault-dr \
  --iam-role-arn arn:aws:iam::$ACCOUNT_ID:role/w8-backup-role \
  --lifecycle DeleteAfterDays=7

aws backup list-copy-jobs --query 'CopyJobs[].[CopyJobId,State]'
aws backup list-recovery-points-by-backup-vault --region $DR_REGION --backup-vault-name w8-vault-dr \
  --query 'RecoveryPoints[].[RecoveryPointArn,Status,CreationDate]'
```

The scheduled plan would copy automatically; the manual copy job just avoids waiting until tomorrow. Stop the instance only after you confirm the recovery point is `COMPLETED`.

### Step 9: Governance guardrails

Choose the path that matches your account.

#### Path A: you have an AWS Organization (use a test OU and member account)

Tag policy (`tag-policy.json`). It standardizes the allowed values of the `Project` tag; to block non-compliant creation, `enforced_for` is needed and supported only for specific resource types, so check the current docs:

```json
{
  "tags": {
    "Project": {
      "tag_key": {"@@assign": "Project"},
      "tag_value": {"@@assign": ["week8", "sandbox"]},
      "enforced_for": {"@@assign": ["ec2:instance"]}
    }
  }
}
```

SCP (`scp.json`). It protects the key and limits Regions. The `NotAction` list exempts global services:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyKeyDeletion",
      "Effect": "Deny",
      "Action": ["kms:ScheduleKeyDeletion", "kms:DisableKey", "kms:DisableKeyRotation"],
      "Resource": "*"
    },
    {
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": ["iam:*", "organizations:*", "sts:*", "support:*", "budgets:*", "cloudfront:*", "route53:*", "route53domains:*", "waf:*", "ce:*", "health:*"],
      "Resource": "*",
      "Condition": {"StringNotEquals": {"aws:RequestedRegion": ["us-east-1", "us-west-2"]}}
    }
  ]
}
```

```bash
aws organizations enable-policy-type --root-id <ROOT_ID> --policy-type TAG_POLICY   # one time
aws organizations create-policy --name w8-tag-policy --type TAG_POLICY \
  --description "Standard Project tag" --content file://tag-policy.json
aws organizations create-policy --name w8-guardrails --type SERVICE_CONTROL_POLICY \
  --description "Protect KMS keys, approved Regions" --content file://scp.json

# Attach only to a TEST OU, never the root
aws organizations attach-policy --policy-id <SCP_ID> --target-id <TEST_OU_ID>
aws organizations attach-policy --policy-id <TAG_POLICY_ID> --target-id <TEST_OU_ID>
```

Prove it from a principal in the test account: launching an instance in `eu-west-1` or running `kms schedule-key-deletion` must return an explicit deny from a service control policy. Remember that SCPs filter permissions but never grant them, and they do not apply to the management account. Because the SCP above blocks key disable and key deletion, run the failure experiment in Step 11 and the cleanup in the management account or detach the SCP first.

#### Path B: single standalone account (fallback)

Use an IAM permissions boundary to simulate the guardrail, plus a Budget for cost control.

```bash
cat > boundary.json <<'EOF'
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":"*","Resource":"*"},
 {"Sid":"DenyKeyDeletion","Effect":"Deny",
  "Action":["kms:ScheduleKeyDeletion","kms:DisableKey","kms:DisableKeyRotation"],"Resource":"*"},
 {"Sid":"DenyOutsideRegions","Effect":"Deny","NotAction":["iam:*","sts:*","organizations:*","budgets:*","support:*","cloudfront:*","route53:*"],
  "Resource":"*","Condition":{"StringNotEquals":{"aws:RequestedRegion":["us-east-1","us-west-2"]}}}]}
EOF
BOUNDARY_ARN=$(aws iam create-policy --policy-name w8-boundary --policy-document file://boundary.json \
  --query Policy.Arn --output text)

# A role that has AdministratorAccess but is capped by the boundary.
# Trust: your own account root principal (any IAM identity here that is allowed to assume it).
cat > dev-trust.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::$ACCOUNT_ID:root"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name w8-dev --assume-role-policy-document file://dev-trust.json \
  --permissions-boundary $BOUNDARY_ARN
aws iam attach-role-policy --role-name w8-dev --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

CREDS=$(aws sts assume-role --role-arn arn:aws:iam::$ACCOUNT_ID:role/w8-dev \
  --role-session-name boundary-test --query Credentials --output json)
AWS_ACCESS_KEY_ID=$(echo $CREDS | python3 -c 'import sys,json;print(json.load(sys.stdin)["AccessKeyId"])') \
AWS_SECRET_ACCESS_KEY=$(echo $CREDS | python3 -c 'import sys,json;print(json.load(sys.stdin)["SecretAccessKey"])') \
AWS_SESSION_TOKEN=$(echo $CREDS | python3 -c 'import sys,json;print(json.load(sys.stdin)["SessionToken"])') \
  aws kms schedule-key-deletion --key-id $KEY_E --pending-window-in-days 7
```

Expected: `AccessDeniedException ... no permissions boundary allows the kms:ScheduleKeyDeletion action`, even though the role has AdministratorAccess. The effective permission is the intersection of the identity policy and the boundary.

Budget with an email alert:

```bash
cat > budget.json <<'EOF'
{"BudgetName":"w8-monthly","BudgetLimit":{"Amount":"10","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}
EOF
cat > notif.json <<'EOF'
[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},
  "Subscribers":[{"SubscriptionType":"EMAIL","Address":"<you@example.com>"}]}]
EOF
aws budgets create-budget --account-id $ACCOUNT_ID --budget file://budget.json \
  --notifications-with-subscribers file://notif.json
```

### Step 10: The restore test and your measured RTO

Restoring is the only proof that a backup works. Start the clock when you decide to recover.

```bash
date -u +%FT%TZ    # T0: "disaster declared"
aws ec2 terminate-instances --instance-ids $INSTANCE   # simulate loss of the server
```

Restore in the console (it handles the instance metadata for you): AWS Backup > Protected resources > select the EC2 instance > select the recovery point in `w8-vault` > Restore. Keep the original network settings or select your private subnet and `w8-lambda-sg`; use `t3.micro`; choose the `w8-backup-role` IAM role. Then follow the restore job:

```bash
aws backup list-restore-jobs --by-resource-type EC2 \
  --query 'RestoreJobs[].[RestoreJobId,Status,CreationDate,CompletionDate]' --output table
date -u +%FT%TZ    # T1: restored instance is running
```

Record three numbers:

- Restore job duration: `CompletionDate - CreationDate`.
- Measured RTO: T1 minus T0, including the time you spent clicking and waiting. This is the honest figure for a backup-and-restore strategy.
- Measured RPO: T0 minus the `CreationDate` of the recovery point you restored (here only minutes, because you took it on demand). The scheduled plan's real worst-case RPO is the backup interval, 24 hours.

Optionally, repeat from the DR Region by restoring from `w8-vault-dr` in `us-west-2` (you would need a subnet and security group there, so treat this as a stretch).

### Step 11: Failure experiments

Experiment A: remove the private path. Delete the endpoint and re-invoke:

```bash
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $EP
sleep 60
aws lambda invoke --function-name w8-secret-reader out.json >/dev/null && cat out.json
```

Expected: `{"ok": false, "error": "ConnectTimeoutError"}` (or a similar endpoint connection error). With no internet gateway or NAT, the secret cannot be reached, which proves the earlier success came through PrivateLink. Recreate the endpoint (Step 4) if you want to test again.

Experiment B: disable the key. If you are under the Path A SCP, do this from the management account or detach the SCP first.

```bash
aws kms disable-key --key-id $KEY_E
aws secretsmanager get-secret-value --secret-id week8/app/db      # fails: KMSInvalidStateException / DisabledException
aws s3api get-object --bucket $SRC_BUCKET --key docs/contract-001.txt /dev/null  # fails: KMS.DisabledException
aws kms enable-key --key-id $KEY_E
aws secretsmanager get-secret-value --secret-id week8/app/db --query Name   # works again
```

Disabling a CMK instantly blocks every service that depends on it; this is the "crypto-shredding" lever, and also why key deletion is guarded by a mandatory 7 to 30 day waiting period.

### Step 12: Fill in the DR strategy comparison

Copy this table into your notes and fill the last column with what you measured. The "typical" figures are rules of thumb, not guarantees.

| Strategy | Typical RTO | Typical RPO | Relative cost | What you would deploy for this workload | Your measured result |
|---|---|---|---|---|---|
| Backup and restore | hours | hours (backup interval) | lowest | This lab: Backup plan plus cross-Region copy | RTO: ___ min, RPO: ___ min (on demand) or 24 h (daily plan) |
| Pilot light | tens of minutes to hours | minutes | low | Replicated database and S3 running in DR, app servers off (AMIs ready) | S3 replication lag: ___ s |
| Warm standby | minutes | seconds to minutes | medium | Scaled-down full stack always running in DR, scale up on failover | Estimate only: ___ |
| Multi-site active/active | near zero | near zero | highest | Full stacks in both Regions with Route 53 routing and Aurora Global Database or DynamoDB global tables | Estimate only: ___ |

Decision rule: pick the cheapest strategy that still meets the stated RTO and RPO. Your scenario asked for RTO under 60 minutes and RPO under 24 hours: did the measured backup-and-restore RTO meet it? If not, which row would you move to?

## Validation

- `aws kms get-key-rotation-status` is `true` for the primary key; the secret, the S3 buckets, the Backup vaults, and the instance volume all reference your keys (`describe-secret`, `get-bucket-encryption`, `describe-backup-vault`, `describe-volumes`).
- The Lambda in the no-internet VPC returned the secret key names through the interface endpoint, and failed after the endpoint was deleted.
- Object `docs/contract-001.txt` exists in the `us-west-2` bucket with `ReplicationStatus` `COMPLETED` on the source, and your lag is recorded.
- A recovery point exists in both `w8-vault` and `w8-vault-dr`, and the restore job finished with a recorded RTO.
- The guardrail (SCP or permissions boundary) returned an explicit deny for key deletion, and the Budget exists.

## Cleanup

Do it in this order and the same day. Hourly-billed items first.

```bash
# 1. Compute and networking (hourly billed)
aws ec2 terminate-instances --instance-ids $INSTANCE      # and the restored instance from Step 10
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $EP 2>/dev/null
aws lambda delete-function --function-name w8-secret-reader

# 2. Backup: delete recovery points before vaults (use the console if the list is long)
for R in $AWS_REGION $DR_REGION; do
  V=w8-vault; [ $R = $DR_REGION ] && V=w8-vault-dr
  for ARN in $(aws backup list-recovery-points-by-backup-vault --region $R --backup-vault-name $V \
      --query 'RecoveryPoints[].RecoveryPointArn' --output text); do
    aws backup delete-recovery-point --region $R --backup-vault-name $V --recovery-point-arn $ARN
  done
done
aws backup delete-backup-selection --backup-plan-id $PLAN_ID --selection-id <SELECTION_ID>
aws backup delete-backup-plan --backup-plan-id $PLAN_ID
aws backup delete-backup-vault --region $AWS_REGION --backup-vault-name w8-vault
aws backup delete-backup-vault --region $DR_REGION --backup-vault-name w8-vault-dr

# 3. S3 (versioned buckets: delete all versions and delete markers)
aws s3api delete-bucket-replication --bucket $SRC_BUCKET
for B in $SRC_BUCKET $DST_BUCKET; do
  R=$AWS_REGION; [ $B = $DST_BUCKET ] && R=$DR_REGION
  aws s3api list-object-versions --bucket $B --region $R \
    --query '{Objects: [Versions[].{Key:Key,VersionId:VersionId}, DeleteMarkers[].{Key:Key,VersionId:VersionId}][]}' \
    --output json > del-$B.json
  # if the list is empty, skip delete-objects; otherwise:
  aws s3api delete-objects --bucket $B --region $R --delete file://del-$B.json
  aws s3api delete-bucket --bucket $B --region $R
done

# 4. Secret (force delete only because this is a lab)
aws secretsmanager delete-secret --secret-id week8/app/db --force-delete-without-recovery

# 5. Network (after endpoint and Lambda ENIs are gone; Lambda ENIs can take several minutes to release)
aws ec2 delete-security-group --group-id $SG_FN
aws ec2 delete-security-group --group-id $SG_EP
aws ec2 delete-subnet --subnet-id $SUBNET_A
aws ec2 delete-vpc --vpc-id $VPC

# 6. IAM
for R in w8-secret-reader w8-s3-replication w8-backup-role; do
  for P in $(aws iam list-role-policies --role-name $R --query PolicyNames --output text); do
    aws iam delete-role-policy --role-name $R --policy-name $P; done
  for A in $(aws iam list-attached-role-policies --role-name $R --query 'AttachedPolicies[].PolicyArn' --output text); do
    aws iam detach-role-policy --role-name $R --policy-arn $A; done
  aws iam delete-role --role-name $R
done
# Path B only:
aws iam detach-role-policy --role-name w8-dev --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam delete-role --role-name w8-dev
aws iam delete-policy --policy-arn $BOUNDARY_ARN
aws budgets delete-budget --account-id $ACCOUNT_ID --budget-name w8-monthly
# Path A only: detach and delete the SCP and tag policy (aws organizations detach-policy / delete-policy)

# 7. KMS last (deletion is scheduled, minimum waiting period 7 days)
aws kms schedule-key-deletion --region $AWS_REGION --key-id $KEY_E --pending-window-in-days 7
aws kms schedule-key-deletion --region $DR_REGION --key-id $KEY_W --pending-window-in-days 7
aws kms delete-alias --region $AWS_REGION --alias-name alias/week8-key
aws kms delete-alias --region $DR_REGION --alias-name alias/week8-key-dr
```

Check Console > EC2 for restored instances, volumes, and AMIs created by the restore, and Console > EC2 > Snapshots for snapshots left behind by backups (deleting recovery points normally removes them). The default VPC route tables are untouched.

## Exam Takeaway

- RTO is how long you can be down; RPO is how much data you can lose. They determine the DR strategy: backup and restore (cheapest, slowest), pilot light, warm standby, multi-site active/active (costliest, near-zero). AWS Backup centralizes backup policy, cross-Region and cross-account copy, and vault lock; S3 CRR needs versioning on both buckets; replication is asynchronous and not retroactive.
- KMS: keys are Regional (use multi-Region keys when the same key material is needed in another Region), customer managed keys give you rotation control, key policies and grants, and CloudTrail audit of every use. Disabling a key blocks all dependent data; deletion has a 7 to 30 day waiting period. Envelope encryption underlies SSE-KMS. For credentials use Secrets Manager (automatic rotation, cross-Region replication) rather than hard-coded values; SSM Parameter Store is the cheaper option when you do not need built-in rotation.
- Interface endpoints (PrivateLink) keep traffic to AWS services off the internet and need no NAT; gateway endpoints exist only for S3 and DynamoDB and are free. Interface endpoints are billed per hour per AZ.
- Organizations: SCPs set the maximum permissions for member accounts (guardrails, never grants, not applied to the management account); tag policies standardize tags; consolidated billing pools discounts; Budgets alert on spend. Permissions boundaries cap IAM entities inside one account.
- Domain mapping: Design Secure Architectures (KMS, Secrets Manager, SCPs, PrivateLink); Design Resilient Architectures (backup, replication, DR strategies, RTO/RPO); Design Cost-Optimized Architectures (strategy selection by cost, Budgets, tag policy); Design High-Performing Architectures (private connectivity design).

## Stretch Goals

1. Replace the second Region key with a KMS multi-Region key (`aws kms create-key --multi-region` then `replicate-key`) and re-encrypt the replication design around it.
2. Replicate the secret to `us-west-2` with `aws secretsmanager replicate-secret-to-regions` and read it from a DR Lambda.
3. Add S3 Replication Time Control and compare its CloudWatch replication latency metrics with your measured lag, then check the pricing premium.
4. Restore the recovery point from `w8-vault-dr` in `us-west-2`, build a short runbook with exact steps, and re-measure the RTO as a true cross-Region recovery.


---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
