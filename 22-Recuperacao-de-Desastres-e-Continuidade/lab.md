# Guided Lab: Backup, Cross-Region Copy and Restore Test with AWS Backup

## Objective

Protect a small EBS-backed EC2 instance with an AWS Backup plan, copy recovery points to a second Region, restore from the copy, and verify S3 Cross-Region Replication (CRR). Then map what you built to the four DR strategies and to RTO/RPO.

## Scenario

A small web server stores a `orders.txt` file on its EBS volume. The business accepts losing up to 24 hours of data (RPO) and up to a few hours of downtime (RTO). You must prove that a backup exists in another Region and that it can really be restored.

## Services Used

- AWS Backup (backup vault, backup plan, copy job, restore job)
- Amazon EC2 / EBS
- Amazon S3 with Cross-Region Replication and versioning
- AWS IAM (service role `AWSBackupDefaultServiceRole`)

## Prerequisites

- Personal AWS account, AWS CLI v2 configured, default Region `us-east-1`. Destination Region: `us-west-2`.
- Permission to use AWS Backup, EC2, S3 and IAM.
- Region `us-east-1` is the source; `us-west-2` is the DR Region.

## Expected Cost

Roughly under 1 USD if you clean up the same day: a `t3.micro` (Free Tier eligible for new accounts) with an 8 GiB gp3 volume, snapshot storage of a few GiB-days (about 0.05 USD per GB-month), and tiny S3 objects. Cross-Region data transfer for 8 GiB is a few cents. Avoid leaving recovery points, restored volumes or the instance running; snapshots are billed until deleted. Do not enable warm-standby resources (RDS replicas, NAT Gateways) for this lab.

## Step by Step

### Step 1: Create the source instance and write test data

Console: EC2 > Launch instance > Amazon Linux 2023, `t3.micro`, 8 GiB gp3 root volume, add tag `backup=daily`. Connect with EC2 Instance Connect or Session Manager.

```bash
echo "order-1001,paid" | sudo tee /home/ec2-user/orders.txt
```

Note the volume ID from the instance's Storage tab.

```bash
aws ec2 describe-instances --region us-east-1 \
  --filters "Name=tag:backup,Values=daily" \
  --query "Reservations[].Instances[].[InstanceId,BlockDeviceMappings[0].Ebs.VolumeId]" --output text
```

### Step 2: Create vaults in both Regions

Console: AWS Backup > Backup vaults > Create backup vault (name `lab-vault`) in `us-east-1`, then switch Region to `us-west-2` and create `lab-vault-dr`.

```bash
aws backup create-backup-vault --backup-vault-name lab-vault --region us-east-1
aws backup create-backup-vault --backup-vault-name lab-vault-dr --region us-west-2
```

### Step 3: Create a backup plan with a cross-Region copy rule

Console: AWS Backup > Backup plans > Create plan > Build a new plan. Rule: daily, retention 7 days, vault `lab-vault`, and under "Copy to destination Region" choose `us-west-2` / `lab-vault-dr` with retention 7 days.

```bash
cat > plan.json <<'EOF'
{
  "BackupPlanName": "lab-daily-with-dr-copy",
  "Rules": [{
    "RuleName": "daily-0500utc",
    "TargetBackupVaultName": "lab-vault",
    "ScheduleExpression": "cron(0 5 ? * * *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 180,
    "Lifecycle": { "DeleteAfterDays": 7 },
    "CopyActions": [{
      "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:<ACCOUNT_ID>:backup-vault:lab-vault-dr",
      "Lifecycle": { "DeleteAfterDays": 7 }
    }]
  }]
}
EOF
aws backup create-backup-plan --backup-plan file://plan.json --region us-east-1
```

Assign resources by tag (console: Plan > Assign resources, IAM role Default role, tag `backup` = `daily`):

```bash
aws backup create-backup-selection --backup-plan-id <BACKUP_PLAN_ID> --region us-east-1 \
  --backup-selection '{"SelectionName":"tagged-daily","IamRoleArn":"arn:aws:iam::<ACCOUNT_ID>:role/service-role/AWSBackupDefaultServiceRole","ListOfTags":[{"ConditionType":"STRINGEQUALS","ConditionKey":"backup","ConditionValue":"daily"}]}'
```

### Step 4: Run an on-demand backup (do not wait for the schedule)

Console: AWS Backup > Protected resources > select the instance > Create on-demand backup.

```bash
aws backup start-backup-job --backup-vault-name lab-vault \
  --resource-arn arn:aws:ec2:us-east-1:<ACCOUNT_ID>:instance/<INSTANCE_ID> \
  --iam-role-arn arn:aws:iam::<ACCOUNT_ID>:role/service-role/AWSBackupDefaultServiceRole \
  --region us-east-1
aws backup list-backup-jobs --by-state COMPLETED --region us-east-1
```

On-demand jobs do not trigger the plan's copy rule. Copy it manually:

```bash
aws backup start-copy-job --recovery-point-arn <RECOVERY_POINT_ARN> \
  --source-backup-vault-name lab-vault \
  --destination-backup-vault-arn arn:aws:backup:us-west-2:<ACCOUNT_ID>:backup-vault:lab-vault-dr \
  --iam-role-arn arn:aws:iam::<ACCOUNT_ID>:role/service-role/AWSBackupDefaultServiceRole \
  --region us-east-1
aws backup list-copy-jobs --region us-east-1
```

### Step 5: Restore test in the DR Region

Console: switch to `us-west-2` > AWS Backup > Backup vaults > `lab-vault-dr` > select the recovery point > Restore. Pick instance type `t3.micro` and a default subnet.

```bash
aws backup list-recovery-points-by-backup-vault --backup-vault-name lab-vault-dr --region us-west-2
aws backup start-restore-job --recovery-point-arn <DR_RECOVERY_POINT_ARN> \
  --iam-role-arn arn:aws:iam::<ACCOUNT_ID>:role/service-role/AWSBackupDefaultServiceRole \
  --metadata file://restore-metadata.json --region us-west-2
```

Get valid metadata with `aws backup get-recovery-point-restore-metadata --backup-vault-name lab-vault-dr --recovery-point-arn <DR_RECOVERY_POINT_ARN> --region us-west-2`; the console builds it for you and is easier. Time the restore from start to a running instance: that is your measured RTO. Connect and `cat /home/ec2-user/orders.txt`.

### Step 6: Check S3 Cross-Region Replication

Console: create `<YOUR_BUCKET_NAME>-src` (us-east-1) and `<YOUR_BUCKET_NAME>-dst` (us-west-2), both with versioning enabled. On the source: Management > Replication rules > Create rule (all objects), create a new IAM role.

```bash
aws s3api put-bucket-versioning --bucket <YOUR_BUCKET_NAME>-src --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket <YOUR_BUCKET_NAME>-dst --versioning-configuration Status=Enabled --region us-west-2
echo "hello dr" > hello.txt
aws s3 cp hello.txt s3://<YOUR_BUCKET_NAME>-src/hello.txt
aws s3api head-object --bucket <YOUR_BUCKET_NAME>-src --key hello.txt --query ReplicationStatus
aws s3api head-object --bucket <YOUR_BUCKET_NAME>-dst --key hello.txt --region us-west-2
```

`ReplicationStatus` goes from `PENDING` to `COMPLETED`. Replication applies only to objects uploaded after the rule exists (use S3 Batch Replication for older ones).

### Step 7: Map the lab to DR strategies

| Strategy | RPO / RTO | What it looks like | In this lab |
|---|---|---|---|
| Backup and restore | Hours | Copies in another Region, rebuild on demand | AWS Backup copy + restore (Steps 3-5) |
| Pilot light | Tens of minutes | Data replicated, core (DB) running, compute off | Would add a replicated DB and AMI, no EC2 running |
| Warm standby | Minutes | Scaled-down full stack always running | Would add a small ASG + replica in us-west-2 |
| Multi-site active/active | Near zero | Full capacity in both Regions, Route 53 routing | Would add Route 53 health checks and a live stack |

## Validation

- A backup job shows `COMPLETED` and a recovery point exists in `lab-vault`.
- A copy job shows `COMPLETED` and a recovery point exists in `lab-vault-dr` (`us-west-2`).
- The restored instance contains `orders.txt` with `order-1001,paid`.
- `hello.txt` shows `ReplicationStatus: COMPLETED` and is present in the destination bucket.
- You can state RPO (time since last backup, up to 24 h for a daily plan) and RTO (your measured restore time).

## Cleanup

Delete in dependency order. Recovery points must go before a vault.

```bash
# restored instance/volume in us-west-2 (console: EC2 > Terminate), then:
aws backup delete-recovery-point --backup-vault-name lab-vault --recovery-point-arn <RECOVERY_POINT_ARN> --region us-east-1
aws backup delete-recovery-point --backup-vault-name lab-vault-dr --recovery-point-arn <DR_RECOVERY_POINT_ARN> --region us-west-2
aws backup delete-backup-selection --backup-plan-id <BACKUP_PLAN_ID> --selection-id <SELECTION_ID> --region us-east-1
aws backup delete-backup-plan --backup-plan-id <BACKUP_PLAN_ID> --region us-east-1
aws backup delete-backup-vault --backup-vault-name lab-vault --region us-east-1
aws backup delete-backup-vault --backup-vault-name lab-vault-dr --region us-west-2
aws ec2 terminate-instances --instance-ids <INSTANCE_ID> --region us-east-1
aws s3 rb s3://<YOUR_BUCKET_NAME>-src --force
aws s3 rb s3://<YOUR_BUCKET_NAME>-dst --force --region us-west-2
```

Versioned buckets keep old versions; `rb --force` does not remove them, so empty versions in the console (S3 > Empty) first. Also delete leftover EBS snapshots and AMIs created by restores.

## Exam Takeaway

Backup and restore is the cheapest DR strategy but has the highest RTO/RPO; each step toward pilot light, warm standby and multi-site cuts recovery time and raises cost. When a question gives a tight RTO/RPO, pick the cheapest strategy that still meets it. Remember: AWS Backup centralizes and automates cross-Region and cross-account copies, S3 CRR requires versioning and only replicates new objects, and a backup you never restored is not proven.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
