# Guided Lab: Cost and Permission Governance with Organizations, SCPs, Budgets and Tags

## Objective

Organize accounts into OUs, write a Service Control Policy (SCP) that denies unapproved Regions, and set up cost governance with AWS Budgets, cost allocation tags and Cost Explorer. A single-account path covers the same ideas with IAM permission boundaries.

## Scenario

A startup wants developers confined to us-east-1 and us-west-2, wants an email when monthly spend nears a limit, and needs to see cost per project. You will build the guardrails and test that they work.

## Services Used

- AWS Organizations (OUs, SCPs)
- AWS Budgets
- Cost allocation tags and AWS Cost Explorer
- AWS IAM (permissions boundary) for the single-account path

## Prerequisites

- AWS CLI v2, Region `us-east-1` (Organizations, Budgets and Cost Explorer are global services with endpoints in us-east-1).
- Full path (Part A): a management account and at least one member account. A free organization works, but SCP testing needs a second account. You can invite or create a member account (each needs its own unique email). If you only have one account, skip to Part B.
- Single-account path (Part B): an admin IAM user in your one account.
- SCPs never affect the management account, so never test there.

## Expected Cost

Organizations, SCPs and IAM are free. The first two Budgets are free (additional action-enabled budgets can cost about 0.10 USD/day after the free ones). Cost Explorer through the console is free; API requests cost about 0.01 USD each, so prefer the console. Avoid launching resources in the denied Region to "see it work"; the test should fail before anything is created.

## Step by Step

### Step 1: Part A - Create OUs

Console: AWS Organizations > AWS accounts > select Root > Actions > Create new (OU) named `Sandbox`, then move the member account into it.

```bash
aws organizations describe-organization
aws organizations list-roots
aws organizations create-organizational-unit --parent-id <ROOT_ID> --name Sandbox
aws organizations move-account --account-id <MEMBER_ACCOUNT_ID> --source-parent-id <ROOT_ID> --destination-parent-id <OU_ID>
```

### Step 2: Part A - Enable SCPs and create a region-deny SCP

Console: Organizations > Policies > Service control policies > Enable, then Create policy.

```bash
aws organizations enable-policy-type --root-id <ROOT_ID> --policy-type SERVICE_CONTROL_POLICY
cat > deny-regions.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyOutsideApprovedRegions",
    "Effect": "Deny",
    "NotAction": ["iam:*","organizations:*","route53:*","cloudfront:*","support:*","budgets:*","ce:*","sts:*"],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": { "aws:RequestedRegion": ["us-east-1","us-west-2"] }
    }
  }]
}
EOF
aws organizations create-policy --name DenyOtherRegions --type SERVICE_CONTROL_POLICY \
  --description "Allow only us-east-1 and us-west-2" --content file://deny-regions.json
aws organizations attach-policy --policy-id <POLICY_ID> --target-id <OU_ID>
```

Global services are exempted through `NotAction` so you do not break IAM or billing. Keep the default `FullAWSAccess` SCP attached; SCPs only filter, they never grant permissions.

### Step 3: Part A - Test in the member account

Sign in (or assume `OrganizationAccountAccessRole`) in the member account as an admin.

```bash
aws ec2 describe-vpcs --region us-east-1   # works
aws ec2 describe-vpcs --region eu-west-1   # AccessDenied: explicit deny in a service control policy
```

Detach the SCP and retry; it now succeeds. Optional: the same `aws ec2 describe-vpcs --region eu-west-1` as the management account still works, proving SCPs do not apply there.

### Step 4: Part B (single account) - Permissions boundary as the guardrail

Console: IAM > Policies > Create policy (JSON below) named `DevBoundary`, then create user `dev-test` and under Permissions boundary select it, with the managed policy `AmazonEC2FullAccess`.

```bash
cat > boundary.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["ec2:*","s3:*"], "Resource": "*",
      "Condition": { "StringEquals": { "aws:RequestedRegion": ["us-east-1","us-west-2"] } } }
  ]
}
EOF
aws iam create-policy --policy-name DevBoundary --policy-document file://boundary.json
aws iam create-user --user-name dev-test --permissions-boundary arn:aws:iam::<ACCOUNT_ID>:policy/DevBoundary
aws iam attach-user-policy --user-name dev-test --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam create-access-key --user-name dev-test
```

Use the new key in a separate profile (never commit it). Effective permissions are the intersection of the identity policy and the boundary.

```bash
aws ec2 describe-vpcs --region us-east-1 --profile dev-test   # works
aws ec2 describe-vpcs --region eu-west-1 --profile dev-test   # UnauthorizedOperation / denied
aws iam list-users --profile dev-test                           # denied: not in the boundary
```

### Step 5: Create a budget with an email alert

Console: Billing and Cost Management > Budgets > Create budget > Customize (advanced) > Cost budget > Monthly, amount 10 USD, alert at 80% of actual spend with your email.

```bash
cat > budget.json <<'EOF'
{ "BudgetName": "lab-monthly-10usd", "BudgetType": "COST", "TimeUnit": "MONTHLY",
  "BudgetLimit": { "Amount": "10", "Unit": "USD" } }
EOF
cat > notifications.json <<'EOF'
[{ "Notification": { "NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN",
   "Threshold": 80, "ThresholdType": "PERCENTAGE" },
   "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "<YOUR_EMAIL>" }] }]
EOF
aws budgets create-budget --account-id <ACCOUNT_ID> --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

Budgets alert; they do not stop spending (only budget actions can attach a deny policy).

### Step 6: Tag resources and activate cost allocation tags

```bash
aws ec2 create-tags --resources <INSTANCE_OR_VOLUME_ID> --tags Key=Project,Value=orders-app
aws ce update-cost-allocation-tags-status --cost-allocation-tags-status TagKey=Project,Status=Active
```

Console equivalent: Billing > Cost allocation tags > select `Project` > Activate. Tags appear in reports only after activation (up to 24 hours), and are not retroactive. In an organization, only the management account can activate them.

### Step 7: Explore Cost Explorer

Console: Billing and Cost Management > Cost Explorer > Launch (first use takes up to 24 hours to populate). Group by Service, then by Tag `Project`. Check the Free Tier page too.

## Validation

- Part A: `describe-vpcs` in `eu-west-1` is denied in the member account, allowed in us-east-1, and allowed again after detaching the SCP.
- Part B: `dev-test` is denied in `eu-west-1` and for IAM actions even though `AmazonEC2FullAccess` is attached.
- `aws budgets describe-budgets --account-id <ACCOUNT_ID>` lists `lab-monthly-10usd`.
- `Project` shows as Active in Cost allocation tags.

## Cleanup

```bash
aws organizations detach-policy --policy-id <POLICY_ID> --target-id <OU_ID>
aws organizations delete-policy --policy-id <POLICY_ID>
aws organizations move-account --account-id <MEMBER_ACCOUNT_ID> --source-parent-id <OU_ID> --destination-parent-id <ROOT_ID>
aws organizations delete-organizational-unit --organizational-unit-id <OU_ID>
aws budgets delete-budget --account-id <ACCOUNT_ID> --budget-name lab-monthly-10usd
aws iam list-access-keys --user-name dev-test    # note the key ID
aws iam delete-access-key --user-name dev-test --access-key-id <ACCESS_KEY_ID>
aws iam detach-user-policy --user-name dev-test --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam delete-user --user-name dev-test
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/DevBoundary
```

Removing a member account fully requires closing it (a 90-day post-closure period applies); only do so if it was created just for this lab.

## Exam Takeaway

SCPs are guardrails that set the maximum permissions for accounts in an OU; they grant nothing and never restrict the management account. An effective permission needs an Allow in both the SCP and the identity policy. Permissions boundaries do the same job for a single IAM principal. For cost questions: Budgets alert (and can act), Cost Explorer analyzes, and cost allocation tags must be activated before they appear in billing data.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
