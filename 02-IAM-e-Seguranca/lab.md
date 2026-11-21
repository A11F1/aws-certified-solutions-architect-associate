# Guided Lab: Least-Privilege S3 Access with an IAM Role and Policy Conditions

## Objective
Build an IAM policy that allows access to one S3 prefix only, attach it to a role assumed through STS, and prove that everything outside the policy is denied.

## Scenario
A reporting application must read files under `reports/` in a bucket but must never see `finance/` or delete anything. You give it a role with a scoped policy and test it.

## Services Used
- AWS IAM (policies, roles, trust policies)
- AWS STS (`AssumeRole`)
- Amazon S3
- IAM Policy Simulator, CloudTrail

## Prerequisites
- Admin identity and CLI from Module 01 (`us-east-1`).
- A globally unique bucket name.

## Expected Cost
About $0. A few tiny S3 objects and IAM/STS calls are free or fractions of a cent. Do not leave the bucket behind.

## Step by Step

### Step 1: Create a bucket with test data
Console: **S3** > **Create bucket** > name `<YOUR_BUCKET_NAME>`, region `us-east-1`, keep Block Public Access on.
```bash
aws s3 mb s3://<YOUR_BUCKET_NAME> --region us-east-1
echo "q1 report" > r.txt; echo "salaries" > f.txt
aws s3 cp r.txt s3://<YOUR_BUCKET_NAME>/reports/r.txt
aws s3 cp f.txt s3://<YOUR_BUCKET_NAME>/finance/f.txt
```

### Step 2: Write the least-privilege policy
```bash
cat > reports-read.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyReportsPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>",
      "Condition": { "StringLike": { "s3:prefix": ["reports/*"] } }
    },
    {
      "Sid": "ReadReportObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/reports/*"
    }
  ]
}
JSON
aws iam create-policy --policy-name ReportsReadOnly \
  --policy-document file://reports-read.json
```
Note the split: `s3:ListBucket` applies to the bucket ARN, `s3:GetObject` to the object ARN.

### Step 3: Create a role your own identity can assume
```bash
cat > trust.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:root" },
    "Action": "sts:AssumeRole"
  }]
}
JSON
aws iam create-role --role-name ReportsReader \
  --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name ReportsReader \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ReportsReadOnly
```
Console path: **IAM** > **Roles** > **Create role** > AWS account > This account. Principal `...:root` in a trust policy means "any principal in this account that is also allowed `sts:AssumeRole` by its own policy". Your admin identity qualifies.

### Step 4: Assume the role and test
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/ReportsReader \
  --role-session-name lab
# export AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN from the output
aws s3 cp s3://<YOUR_BUCKET_NAME>/reports/r.txt -        # works
aws s3 cp s3://<YOUR_BUCKET_NAME>/finance/f.txt -        # AccessDenied
aws s3 rm s3://<YOUR_BUCKET_NAME>/reports/r.txt          # AccessDenied
aws s3 ls s3://<YOUR_BUCKET_NAME>/finance/               # AccessDenied
```
Unset the three variables afterward to return to your admin identity.

### Step 5: Use the Policy Simulator and an explicit Deny
Console: open the **IAM Policy Simulator** (policysim.aws.amazon.com), select role `ReportsReader`, service S3, actions `GetObject` and `DeleteObject`, and run. Then add this statement to the policy to see precedence (an explicit Deny overrides any Allow):
```json
{ "Sid": "NoDeleteEver", "Effect": "Deny", "Action": "s3:DeleteObject", "Resource": "*" }
```

### Step 6: Look at the activity in CloudTrail
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 3
```
Event history shows management events such as `AssumeRole`. S3 object-level calls (`GetObject`) are data events and are not recorded unless you enable data events on a trail. This is an exam-relevant detail.

## Validation
- Reading `reports/r.txt` succeeds; reading `finance/`, listing it, and deleting objects all fail with `AccessDenied`.
- `aws sts get-caller-identity` with the temporary keys shows an `assumed-role/ReportsReader/lab` ARN.

## Cleanup
```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws iam detach-role-policy --role-name ReportsReader \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ReportsReadOnly
aws iam delete-role --role-name ReportsReader
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ReportsReadOnly
aws s3 rb s3://<YOUR_BUCKET_NAME> --force
```

## Exam Takeaway
Roles with temporary STS credentials beat long-lived user keys. Policy evaluation: everything is denied by default, an explicit Deny always wins, and an Allow is needed from every applicable layer (identity policy, plus resource policy, SCP, or permissions boundary when present). Remember the bucket-ARN versus object-ARN split for S3 actions. Identity-policy versus resource-policy and same-account versus cross-account questions build on this exact mechanic.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
