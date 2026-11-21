# Guided Lab: Guardrails with Permission Boundaries, Tag-Based Access, and Credential Reports

## Objective
Practice three advanced IAM controls: a permissions boundary that caps what a delegated user can grant, attribute-based access control (ABAC) with tags, and auditing with the credential report and IAM Access Analyzer.

## Scenario
A junior engineer may create IAM roles for their app but those roles must never exceed S3 read access, and the engineer may only start/stop EC2 instances tagged for their own team.

## Services Used
- IAM (permissions boundary, tags, credential report)
- IAM Access Analyzer
- Amazon EC2 (one tagged instance, optional)

## Prerequisites
- Admin identity and CLI from Module 01.
- Optional: the default VPC (exists by default) for the EC2 test.

## Expected Cost
$0 for the IAM features. The optional EC2 test uses one `t3.micro` for a few minutes (Free Tier eligible, otherwise a few cents). Terminate it afterward.

## Step by Step

### Step 1: Create the boundary policy
```bash
cat > boundary.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:Get*", "s3:List*"],
    "Resource": "*"
  }]
}
JSON
aws iam create-policy --policy-name S3ReadBoundary --policy-document file://boundary.json
```

### Step 2: Create the delegated user
The user may create roles named `app-*` only if the boundary is attached, may attach policies to those roles, and may start/stop only EC2 instances whose `Team` tag matches the user's own `Team` tag.
```bash
aws iam create-user --user-name junior-dev
cat > delegate.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:CreateRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/app-*",
      "Condition": { "StringEquals": {
        "iam:PermissionsBoundary": "arn:aws:iam::<ACCOUNT_ID>:policy/S3ReadBoundary" } }
    },
    {
      "Effect": "Allow",
      "Action": "iam:AttachRolePolicy",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/app-*"
    },
    {
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "arn:aws:ec2:us-east-1:<ACCOUNT_ID>:instance/*",
      "Condition": { "StringEquals": {
        "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}" } }
    }
  ]
}
JSON
aws iam put-user-policy --user-name junior-dev \
  --policy-name delegate --policy-document file://delegate.json
aws iam tag-user --user-name junior-dev --tags Key=Team,Value=blue
aws iam create-access-key --user-name junior-dev   # store in a CLI profile named "junior"
```

### Step 3: Test the boundary
```bash
cat > ec2-trust.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
JSON
# Without the boundary: denied
aws iam create-role --profile junior --role-name app-bad \
  --assume-role-policy-document file://ec2-trust.json
# With the boundary: allowed
aws iam create-role --profile junior --role-name app-ok \
  --assume-role-policy-document file://ec2-trust.json \
  --permissions-boundary arn:aws:iam::<ACCOUNT_ID>:policy/S3ReadBoundary
# Attaching a powerful policy succeeds, but effective permissions stay capped
aws iam attach-role-policy --profile junior --role-name app-ok \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```
Now open the **IAM Policy Simulator**, select role `app-ok`, and test `s3:GetObject` (allowed) and `ec2:RunInstances` (implicitly denied) even though the role holds AdministratorAccess. Effective permissions are the intersection of the identity policy and the boundary.

### Step 4: Test tag-based (ABAC) access (optional EC2)
Find a current Amazon Linux 2023 AMI:
```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)
aws ec2 run-instances --image-id $AMI --instance-type t3.micro --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Team,Value=blue}]'
aws ec2 stop-instances --profile junior --instance-ids <INSTANCE_ID>    # allowed
aws ec2 create-tags --resources <INSTANCE_ID> --tags Key=Team,Value=red  # run as admin
aws ec2 start-instances --profile junior --instance-ids <INSTANCE_ID>   # UnauthorizedOperation
```

### Step 5: Audit with the credential report and Access Analyzer
```bash
aws iam generate-credential-report
aws iam get-credential-report --query Content --output text | base64 --decode > report.csv
```
Open `report.csv` and check `mfa_active`, `password_last_used`, and `access_key_1_last_rotated`.

Console: **IAM** > **Access Analyzer** > **Create analyzer** (external access, zone of trust = your account). Findings list resources shared outside your account (S3 buckets, roles, KMS keys, and so on). External-access findings are free.

## Validation
- `app-bad` creation returns `AccessDenied`; `app-ok` succeeds.
- Policy Simulator shows only S3 read actions allowed for `app-ok` despite AdministratorAccess.
- The junior user can stop/start the instance only while its `Team` tag equals `blue`.
- The credential report lists `junior-dev` with its access key age.

## Cleanup
```bash
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
aws iam detach-role-policy --role-name app-ok --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam delete-role --role-name app-ok
aws iam delete-user-policy --user-name junior-dev --policy-name delegate
aws iam delete-access-key --user-name junior-dev --access-key-id <ACCESS_KEY_ID>
aws iam delete-user --user-name junior-dev
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/S3ReadBoundary
```
Delete the Access Analyzer in the console if you do not want to keep it.

## Exam Takeaway
A permissions boundary never grants anything; effective permissions are the intersection of the identity policy and the boundary, which makes it the answer for safely delegating IAM administration. Compare it with SCPs (guardrails across accounts in an Organization) and ABAC (scale access with tags instead of per-resource policies). The credential report and Access Analyzer are the "find stale credentials / externally shared resources" answers.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
