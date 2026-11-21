# Guided Lab: Set Up a Safe, Cost-Controlled AWS Account for SAA-C03 Prep

## Objective
Prepare a personal AWS account so you can practice every later module safely: root user locked down with MFA, a daily-use admin identity, a budget and billing alarm, CloudTrail confirmed, and the AWS CLI configured for `us-east-1`.

## Scenario
You just created a new AWS account to study for SAA-C03. Before launching anything, you want to make sure a leaked key or a forgotten resource cannot cost you hundreds of dollars, and that you never use the root user for daily work.

## Services Used
- AWS Billing and Cost Management (Budgets, Free Tier alerts)
- Amazon CloudWatch (billing alarm) and Amazon SNS
- AWS IAM / IAM Identity Center
- AWS CloudTrail
- AWS CLI v2

## Prerequisites
- A new or personal AWS account (not an employer account) and access to the root user email.
- An MFA app (authenticator) or a FIDO2 security key.
- AWS CLI v2 installed (`aws --version`).

## Expected Cost
Effectively $0. AWS Budgets has a free allowance (the first two action-free budgets per account), CloudTrail event history is free, and a few CloudWatch alarms fall under the Free Tier. Avoid creating extra trails (a second copy of management events is billed) and never create access keys for the root user.

## Step by Step

### Step 1: Enable MFA on the root user
Console: sign in as root, click your account name (top right) > **Security credentials** > **Multi-factor authentication (MFA)** > **Assign MFA device**. Choose an authenticator app (or passkey/security key), scan the QR code, and enter two consecutive codes.

On the same page confirm that **Access keys** is empty. Root access keys should not exist.

### Step 2: Create a daily-use admin identity
Preferred: IAM Identity Center. Console: **IAM Identity Center** > **Enable** (use `us-east-1`) > **Users** > **Add user** > **Permission sets** > create one from the `AdministratorAccess` template > **AWS accounts** > assign your user and permission set to your account. Sign in through the access portal URL.

Simpler alternative with an IAM user (create it in the console as root the first time, since you have no other credentials yet; the CLI form is shown for reference):
```bash
aws iam create-user --user-name study-admin
aws iam attach-user-policy --user-name study-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam create-login-profile --user-name study-admin \
  --password '<TEMP_PASSWORD>' --password-reset-required
```
Add an MFA device to this user as well (IAM > Users > study-admin > Security credentials > Assign MFA device). Sign out of root and use this identity from now on.

### Step 3: Turn on billing visibility and Free Tier alerts
Console (as root, once): account menu > **Account** > **IAM user and role access to Billing information** > **Edit** > activate. Then **Billing and Cost Management** > **Billing preferences** > enable **Receive Free Tier usage alerts** and **Receive CloudWatch billing alerts**, enter your email, and save.

### Step 4: Create a monthly cost budget
Console: **Billing and Cost Management** > **Budgets** > **Create budget** > **Use a template (simplified)** > **Monthly cost budget**, amount `10` USD, add your email.

CLI equivalent:
```bash
cat > budget.json <<'JSON'
{
  "BudgetName": "study-monthly-10usd",
  "BudgetLimit": { "Amount": "10", "Unit": "USD" },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
JSON
cat > notif.json <<'JSON'
[{
  "Notification": {
    "NotificationType": "ACTUAL",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 80,
    "ThresholdType": "PERCENTAGE"
  },
  "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "<YOUR_EMAIL>" }]
}]
JSON
aws budgets create-budget --account-id <ACCOUNT_ID> \
  --budget file://budget.json --notifications-with-subscribers file://notif.json
```

### Step 5: Create a CloudWatch billing alarm
Billing metrics are published only in `us-east-1`, which is another reason to work there.
```bash
aws sns create-topic --name billing-alerts --region us-east-1
aws sns subscribe --region us-east-1 \
  --topic-arn arn:aws:sns:us-east-1:<ACCOUNT_ID>:billing-alerts \
  --protocol email --notification-endpoint <YOUR_EMAIL>
# confirm the subscription from the email, then:
aws cloudwatch put-metric-alarm --region us-east-1 \
  --alarm-name billing-over-10usd \
  --namespace AWS/Billing --metric-name EstimatedCharges \
  --dimensions Name=Currency,Value=USD \
  --statistic Maximum --period 21600 --evaluation-periods 1 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:<ACCOUNT_ID>:billing-alerts
```

### Step 6: Configure the CLI for us-east-1
With Identity Center:
```bash
aws configure sso          # SSO region us-east-1, pick account and role, profile name "study"
aws sso login --profile study
```
With an IAM user, create an access key under Security credentials and run `aws configure` (default region `us-east-1`, output `json`). Never commit keys to git.
```bash
export AWS_PROFILE=study AWS_REGION=us-east-1
aws sts get-caller-identity
```
The returned ARN must not end in `:root`.

### Step 7: Verify CloudTrail
Console: **CloudTrail** > **Event history**. You should already see your recent `ConsoleLogin` and API calls (management events, last 90 days, free).
```bash
aws cloudtrail lookup-events --max-results 5 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ConsoleLogin
```
Optional, for long-term retention: **Create trail** with a new S3 bucket, all regions, management events only. The first copy of management events per region is free; S3 storage is billed.

### Step 8: Failure experiment, prove a user with no permissions is denied
```bash
aws iam create-user --user-name no-perms-test
aws iam create-access-key --user-name no-perms-test   # put the key in a throwaway profile "noperms"
aws s3 ls --profile noperms                           # expect AccessDenied
```
Then find the `CreateUser` event in CloudTrail event history (it can take up to about 15 minutes to appear).

## Validation
- `aws sts get-caller-identity` shows your admin identity, not root.
- Root has MFA and no access keys (the IAM dashboard shows no security recommendations about root).
- `aws budgets describe-budgets --account-id <ACCOUNT_ID>` lists your budget.
- `aws cloudwatch describe-alarms --alarm-names billing-over-10usd --region us-east-1` shows state `OK` or `INSUFFICIENT_DATA`.
- CloudTrail event history shows your actions.

## Cleanup
Keep the budget, alarm, MFA, and admin identity: they protect later labs. Remove only the test user.
```bash
aws iam delete-access-key --user-name no-perms-test --access-key-id <ACCESS_KEY_ID>
aws iam delete-user --user-name no-perms-test
```
If you created an optional trail, delete it and then the bucket:
```bash
aws cloudtrail delete-trail --name <YOUR_TRAIL_NAME>
aws s3 rb s3://<YOUR_BUCKET_NAME> --force
```

## Exam Takeaway
This lab maps to the Security and Cost Optimization pillars: protect the root user (MFA, no access keys, use it only for the few tasks that require it), give people day-to-day access through least-privilege identities (IAM Identity Center is the recommended approach for workforce access), and use detective controls (CloudTrail for who did what, Budgets and billing alarms for cost). Questions about the "most secure way to give a person access" or "auditing API activity" point here. Billing metrics live only in `us-east-1`.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
