# Week 1 Main Lab: Secure Account Foundation with a Least-Privilege Application Role

## Objective
Build the baseline every real AWS account needs before any workload runs: protected root user, a budget alarm, a day-to-day admin identity, an encrypted and tamper-evident CloudTrail log archive, and an application role that can read exactly one S3 prefix and nothing else. You then prove the design works by running the policy simulator, making a real denied API call, and finding that denial in CloudTrail.

Estimated time: 75 to 100 minutes.

## Modules Covered
- [01 - Introduction to SAA-C03](../01-Introducao-SAA-C03/README.md)
- [02 - IAM and Security](../02-IAM-e-Seguranca/README.md)
- [03 - IAM Labs](../03-IAM-e-Seguranca-Labs/lab.md)

## Scenario and Architecture
You are the first cloud engineer at a small company that just opened a new AWS account. Before the first application ships, the security lead requires: (1) no daily use of the root user, (2) an alert before spend surprises anyone, (3) an immutable-ish audit trail of every API call, and (4) the reporting application may read only `reports/` objects from one bucket, even if someone later attaches a broad policy to its role.

```
                         AWS Account (us-east-1 home Region)
  +----------------------------------------------------------------------+
  |  Root user (MFA, no access keys)          AWS Budgets ---> e-mail    |
  |                                                                      |
  |  IAM admin user / Identity Center  --assume-->  lab-app-role         |
  |        (MFA, daily work)                         |                    |
  |                                                  | permissions =      |
  |                                                  | identity policy    |
  |                                                  |   INTERSECT        |
  |                                                  | permissions        |
  |                                                  |   boundary         |
  |                                                  v                    |
  |                         +-------------------------------+            |
  |                         | S3: app-data-bucket           |            |
  |                         |   reports/*   <- ALLOWED read |            |
  |                         |   secret/*    <- DENIED       |            |
  |                         +-------------------------------+            |
  |                                                                      |
  |  CloudTrail (multi-Region trail, log file validation)                |
  |        |                                                             |
  |        v                                                             |
  |  S3: audit-log-bucket (SSE, Block Public Access, bucket policy       |
  |       allowing only cloudtrail.amazonaws.com to write)               |
  |                                                                      |
  |  Event history / lookup-events  <-- shows AccessDenied for the role  |
  +----------------------------------------------------------------------+
```

## Services Used
- AWS IAM (users, roles, managed policies, permissions boundary, policy simulator)
- AWS STS (`AssumeRole`)
- AWS CloudTrail (trail and event history)
- Amazon S3 (log bucket and application data bucket, default encryption, Block Public Access)
- AWS Budgets
- AWS IAM Identity Center (discussed; optional to enable)

## Prerequisites
- A personal AWS account you own. Do not run this in a shared or employer account.
- AWS CLI v2 configured with an admin-capable IAM user (not root, not long-lived root keys). Check with `aws sts get-caller-identity`.
- Region `us-east-1`. IAM is global, but S3 buckets and the trail home Region in this lab are `us-east-1`.
- An e-mail address for the budget alert and an authenticator app or security key for MFA.
- `jq` is useful but optional.

Set shell variables used throughout:

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SUFFIX=$(date +%s)   # keeps bucket names globally unique
export LOG_BUCKET=lab-w1-audit-logs-${ACCOUNT_ID}-${SUFFIX}
export DATA_BUCKET=lab-w1-app-data-${ACCOUNT_ID}-${SUFFIX}
export ALERT_EMAIL=<YOUR_EMAIL>
```

## Expected Cost
Approximately a few cents or less if you clean up the same day. IAM, STS, and Budgets (the first budgets) are free. CloudTrail keeps your first copy of management events free in event history, and the first management-event trail delivers to S3 at no CloudTrail charge; you pay only for S3 storage of a few small log files. Do not enable data events or CloudTrail Lake in this lab, since they are billed per event or per GB. Prices change, so confirm on the AWS pricing pages for CloudTrail, S3, and Budgets before you start.

## Step by Step

### Step 1: Secure the root user
Console navigation:
1. Sign in as the root user (this is one of the few justified uses of root).
2. Account menu (top right) > **Security credentials**.
3. **Multi-factor authentication (MFA)** > **Assign MFA device**. Choose a passkey/security key or an authenticator app, and register it.
4. Confirm that **Access keys** shows none. If root access keys exist, delete them.
5. Sign out. Store the root password and MFA device somewhere safe.

Root-only tasks to remember for the exam: changing account settings or the root e-mail, closing the account, restoring IAM permissions when locked out, and a few specific S3/SQS policy repairs.

### Step 2: Create a budget alarm
Console: **Billing and Cost Management > Budgets > Create budget > Use a template > Monthly cost budget**, set an amount such as 10 USD, and enter your e-mail. Or with the CLI:

```bash
cat > budget.json <<JSON
{
  "BudgetName": "lab-monthly-budget",
  "BudgetLimit": {"Amount": "10", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
JSON

cat > budget-notifications.json <<JSON
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "${ALERT_EMAIL}"}]
  }
]
JSON

aws budgets create-budget \
  --account-id $ACCOUNT_ID \
  --budget file://budget.json \
  --notifications-with-subscribers file://budget-notifications.json
```

If the call fails because billing data access is not enabled for IAM users, enable "IAM user and role access to Billing information" in the root account settings (account menu > Account), then retry.

### Step 3: Day-to-day admin identity
Pick one of two paths.

Path A (simple, used in this lab): an IAM user with MFA.
```bash
aws iam create-user --user-name lab-admin
aws iam attach-user-policy --user-name lab-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```
Then in the console: **IAM > Users > lab-admin > Security credentials > Assign MFA device**. Enable console access with a generated password only if you need it. If you already use an admin user, reuse it rather than creating another.

Path B (recommended by AWS for people): **IAM Identity Center**. Console: **IAM Identity Center > Enable** (this creates an organization if you have none), create a user, create a permission set (for example `AdministratorAccess`), and assign it to your account. You then sign in through the access portal with short-lived credentials and no long-lived keys. This is a note for the lab, not a requirement. The rest of the lab works with either path.

For the remaining steps, act as the admin identity, not root.

### Step 4: Create the encrypted log bucket and the CloudTrail trail
```bash
aws s3api create-bucket --bucket $LOG_BUCKET --region us-east-1

aws s3api put-public-access-block --bucket $LOG_BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-encryption --bucket $LOG_BUCKET \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

Bucket policy that lets only the CloudTrail service write, scoped to your trail with `aws:SourceArn`:

```bash
cat > trail-bucket-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${LOG_BUCKET}",
      "Condition": {"StringEquals": {"aws:SourceArn": "arn:aws:cloudtrail:us-east-1:${ACCOUNT_ID}:trail/lab-trail"}}
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${LOG_BUCKET}/AWSLogs/${ACCOUNT_ID}/*",
      "Condition": {"StringEquals": {
        "s3:x-amz-acl": "bucket-owner-full-control",
        "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:${ACCOUNT_ID}:trail/lab-trail"}}
    }
  ]
}
JSON

aws s3api put-bucket-policy --bucket $LOG_BUCKET --policy file://trail-bucket-policy.json
```

Create and start the trail (multi-Region, with log file validation so tampering can be detected):

```bash
aws cloudtrail create-trail --name lab-trail \
  --s3-bucket-name $LOG_BUCKET \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name lab-trail
aws cloudtrail get-trail-status --name lab-trail --query IsLogging
```

Console equivalent: **CloudTrail > Trails > Create trail**. Tip: the console wizard can create the bucket and policy for you; the CLI path above shows what it builds. Encryption note: this lab uses SSE-S3 for simplicity. CloudTrail can also encrypt log files with an SSE-KMS key (`--kms-key-id`), which adds a key policy to manage and KMS request charges.

### Step 5: Create the application data bucket and sample objects
```bash
aws s3api create-bucket --bucket $DATA_BUCKET --region us-east-1
aws s3api put-public-access-block --bucket $DATA_BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

echo "Q3 sales summary" > q3.txt
echo "database password placeholder - not real" > creds.txt
aws s3 cp q3.txt s3://$DATA_BUCKET/reports/q3.txt
aws s3 cp creds.txt s3://$DATA_BUCKET/secret/creds.txt
```

### Step 6: Write the permissions boundary
The boundary defines the maximum the role may ever do: read objects under `reports/` and list only that prefix.

```bash
cat > boundary.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadReportsObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${DATA_BUCKET}/reports/*"
    },
    {
      "Sid": "ListReportsPrefixOnly",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::${DATA_BUCKET}",
      "Condition": {"StringLike": {"s3:prefix": ["reports/*", "reports"]}}
    }
  ]
}
JSON

aws iam create-policy --policy-name lab-app-boundary \
  --policy-document file://boundary.json
```

### Step 7: Create the application role with the boundary
Trust policy: only your admin identity in this account may assume the role (a real application would trust `ec2.amazonaws.com` or `lambda.amazonaws.com` instead). Using the account root principal as shown allows any principal in the account that has `sts:AssumeRole` permission, which is fine for a lab; tighten it to a specific user ARN if you prefer.

```bash
cat > app-trust.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::${ACCOUNT_ID}:root"},
    "Action": "sts:AssumeRole"
  }]
}
JSON

aws iam create-role --role-name lab-app-role \
  --assume-role-policy-document file://app-trust.json \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/lab-app-boundary
```

To make the boundary matter, deliberately attach a policy that is too broad:

```bash
aws iam attach-role-policy --role-name lab-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

The identity policy alone would allow reading everything in S3, but effective permissions are the intersection of the identity policy and the boundary. Console check: **IAM > Roles > lab-app-role > Permissions** shows the "Permissions boundary" section.

### Step 8: Prove it with the IAM policy simulator
Allowed case, `reports/`:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::${ACCOUNT_ID}:role/lab-app-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::${DATA_BUCKET}/reports/q3.txt \
  --query 'EvaluationResults[].[EvalActionName,EvalDecision]' --output table
```

Denied case, `secret/`, and a different action:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::${ACCOUNT_ID}:role/lab-app-role \
  --action-names s3:GetObject s3:PutObject \
  --resource-arns arn:aws:s3:::${DATA_BUCKET}/secret/creds.txt \
  --query 'EvaluationResults[].[EvalActionName,EvalDecision]' --output table
```

Expected: `allowed` for the first, `implicitDeny` for the others. Console version: **IAM policy simulator** (`policysim.aws.amazon.com`) > Mode: Existing policies > Roles > lab-app-role > choose S3 actions and run. Note that the simulator does not make real calls; it evaluates policies only.

### Step 9: Make real calls as the role, including a denied one
```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/lab-app-role \
  --role-session-name w1-test \
  --query Credentials --output json)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r .AccessKeyId)
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r .SecretAccessKey)
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r .SessionToken)

aws sts get-caller-identity          # shows assumed-role/lab-app-role/w1-test
aws s3 cp s3://$DATA_BUCKET/reports/q3.txt -        # works
aws s3 cp s3://$DATA_BUCKET/secret/creds.txt -      # AccessDenied
aws s3api list-buckets                              # AccessDenied (ListBuckets is not in the boundary)
```

If you do not have `jq`, copy the three values from the JSON by hand. After testing, return to your admin identity:

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws sts get-caller-identity          # back to the admin identity
```

### Step 10: Find the denial in CloudTrail
Management events usually show up in event history within a few minutes (it can take up to roughly 15 minutes). `ListBuckets` is a management event, so the denial is searchable. Object-level calls such as `GetObject` are data events and do not appear in event history unless you enable data event logging on a trail.

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ListBuckets \
  --max-results 5 \
  --query 'Events[].CloudTrailEvent' --output text | jq '{eventName, errorCode, errorMessage, arn: .userIdentity.arn}'
```

You should see `errorCode: AccessDenied` and the assumed-role ARN `.../lab-app-role/w1-test`. Also look up `AssumeRole`:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 3
```

Console: **CloudTrail > Event history**, filter by Event name. Then, after about 5 to 15 minutes, confirm log files reached the bucket:

```bash
aws s3 ls s3://$LOG_BUCKET/AWSLogs/${ACCOUNT_ID}/CloudTrail/ --recursive | head
```

### Step 11: Failure experiment, break the boundary on purpose
Question: does adding a broad Allow to the role override the boundary? Attach `AdministratorAccess` to `lab-app-role`, re-assume the role, and retry the `secret/` read.

```bash
aws iam attach-role-policy --role-name lab-app-role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::${ACCOUNT_ID}:role/lab-app-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::${DATA_BUCKET}/secret/creds.txt \
  --query 'EvaluationResults[].EvalDecision'
```

Expected: still `implicitDeny`, because the boundary never allowed that action. A boundary caps permissions; it does not grant them. Detach `AdministratorAccess` again afterward:

```bash
aws iam detach-role-policy --role-name lab-app-role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

## Validation
- Root has MFA and no access keys (**Security credentials** page).
- `aws budgets describe-budgets --account-id $ACCOUNT_ID` lists `lab-monthly-budget`.
- `aws cloudtrail get-trail-status --name lab-trail` reports `IsLogging: true`, and `aws cloudtrail describe-trails --query 'trailList[].[Name,IsMultiRegionTrail,LogFileValidationEnabled]'` shows `true` for both flags.
- Log bucket shows default encryption and all four Block Public Access settings on.
- Simulator: `reports/` allowed, `secret/` denied, even with `AdministratorAccess` attached.
- Real calls: `reports/q3.txt` returns content, `secret/creds.txt` and `list-buckets` return AccessDenied.
- CloudTrail lookup shows the denied `ListBuckets` event with the role session ARN.

## Cleanup
Run as the admin identity (unset the temporary role credentials first). Order matters: dependents before dependencies.

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

# 1. Stop and delete the trail (log bucket cannot be emptied meaningfully while it keeps writing)
aws cloudtrail stop-logging --name lab-trail
aws cloudtrail delete-trail --name lab-trail

# 2. Empty and delete the buckets (no versioning is enabled, so a plain recursive delete is enough)
aws s3 rm s3://$DATA_BUCKET --recursive
aws s3api delete-bucket --bucket $DATA_BUCKET
aws s3 rm s3://$LOG_BUCKET --recursive
aws s3api delete-bucket --bucket $LOG_BUCKET

# 3. Role: detach policies, then delete the role, then the boundary policy
aws iam detach-role-policy --role-name lab-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name lab-app-role
aws iam delete-policy --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/lab-app-boundary

# 4. Budget (optional: keeping a budget is free and good practice)
aws budgets delete-budget --account-id $ACCOUNT_ID --budget-name lab-monthly-budget
```

Keep root MFA, the admin identity with MFA, and ideally the budget; these are real baseline controls, not lab clutter. If you created `lab-admin` only for this lab, remove its MFA device, delete its login profile and access keys, detach `AdministratorAccess`, then `aws iam delete-user`. If you enabled IAM Identity Center only for the lab, delete the assignments and users before disabling it.

### What Would You Change for Production
- Use AWS Organizations with a dedicated log-archive account, an organization trail, and Service Control Policies as guardrails above account-level IAM.
- Use IAM Identity Center with an external IdP instead of IAM users; no human long-lived access keys.
- Encrypt logs with SSE-KMS, enable S3 Object Lock or MFA delete on the log bucket, and send events to CloudWatch Logs with alarms for root login, IAM changes, and `AccessDenied` spikes.
- Enable data events only for sensitive buckets, and turn on IAM Access Analyzer, GuardDuty, and AWS Config.
- Trust a specific service principal (for example EC2 or Lambda) in the role, not the account root, and add `aws:PrincipalTag` or `ExternalId` conditions where relevant.

## Exam Takeaway
- Domain 1, Design Secure Architectures (the heaviest domain in SAA-C03): root MFA and no daily root use, least privilege, IAM roles instead of long-lived keys, Identity Center for workforce access, and the evaluation logic: explicit Deny beats Allow, default is implicit deny, and permissions boundaries and SCPs limit but never grant.
- Effective permissions of a role with a boundary = identity policy AND boundary (intersection). Resource policies in the same account can still grant independently of the boundary for principals named explicitly, a subtlety that appears in "why is access allowed" questions.
- CloudTrail records management events by default in event history for 90 days; a trail is needed for long retention, multi-Region coverage, and data events. Log file validation proves logs were not altered. S3 plus encryption plus a restrictive bucket policy is the standard audit archive.
- Domain 4, Design Cost-Optimized Architectures: Budgets for alerts, and cost awareness as a baseline control.
- Domain 2/3 hook: this baseline is what makes multi-account governance and monitoring possible later (AWS Organizations, CloudWatch alarms on CloudTrail events).

## Stretch Goals
1. Add a CloudWatch Logs delivery to the trail with a metric filter and alarm for `ConsoleLogin` without MFA or for root usage.
2. Replace the account-root trust with a specific trust to `ec2.amazonaws.com`, create an instance profile, and read `reports/` from an EC2 instance with no keys on disk.
3. Add an explicit `Deny` bucket policy on the data bucket for requests without TLS (`aws:SecureTransport` is `false`) and test it.
4. Run IAM Access Analyzer (**IAM > Access Analyzer**) and review any findings for the two buckets and the role.


---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
