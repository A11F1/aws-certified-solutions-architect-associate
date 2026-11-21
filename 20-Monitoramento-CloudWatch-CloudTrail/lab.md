# Guided Lab: Monitoring and Auditing with CloudWatch, SNS and CloudTrail

## Objective

Monitor an EC2 instance with a CloudWatch alarm that sends an email through SNS, turn application log lines into a metric with a metric filter, look up API activity in CloudTrail event history, and create a trail that delivers to S3.

## Scenario

An operations team needs to know when a server is overloaded, when an application logs errors, and who deleted or changed something in the account. You build the minimum alerting and audit setup.

## Services Used

- Amazon CloudWatch (metrics, alarms, Logs, metric filters)
- Amazon SNS
- AWS CloudTrail
- Amazon S3
- Amazon EC2 (a t3.micro as the monitored target)
- Optional: AWS Config

## Prerequisites

- AWS CLI v2 configured for `us-east-1`, and an email address you can read.
- A default VPC with a public subnet (the account default is fine).

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=<ACCOUNT_ID>
export BUCKET=<YOUR_BUCKET_NAME>
export EMAIL=<YOUR_EMAIL>
```

## Expected Cost

Under $1 if you clean up the same day. t3.micro (or t2.micro) is Free Tier eligible for the first 12 months on older accounts; otherwise about $0.01 per hour. The first copy of management events in a CloudTrail trail is free; S3 storage is pennies. Standard alarms cost $0.10 per month each (prorated; the first 10 are in the Free Tier). SNS email is free at this volume. AWS Config (optional) charges per configuration item and per rule evaluation, so delete it when done. Avoid: enabling data events or multi-region extra trails (the second copy of management events is billed), and leaving log groups without retention.

## Step by Step

### Step 1: Create an SNS topic and subscribe your email

Console: SNS > Topics > Create topic > Standard, name `ops-alerts`. Create subscription > Protocol Email > your address. Open the email and click **Confirm subscription**.

```bash
TOPIC_ARN=$(aws sns create-topic --name ops-alerts --query TopicArn --output text)
aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint $EMAIL
# confirm via the email link, then:
aws sns list-subscriptions-by-topic --topic-arn $TOPIC_ARN
```

The subscription stays `PendingConfirmation` until you click the link.

### Step 2: Launch an instance and create a CPU alarm

Console: EC2 > Launch instance > Amazon Linux 2023, t3.micro, default VPC, no key pair needed (use SSM or just generate load later with Session Manager if the role is attached; alternatively, skip load and use the manual alarm test below).

```bash
AMI=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)
INSTANCE_ID=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=monitor-lab}]' \
  --query 'Instances[0].InstanceId' --output text)

aws cloudwatch put-metric-alarm \
  --alarm-name monitor-lab-cpu-high \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --statistic Average --period 300 --evaluation-periods 1 \
  --threshold 70 --comparison-operator GreaterThanThreshold \
  --alarm-actions $TOPIC_ARN
```

EC2 basic monitoring publishes at 5-minute intervals. Detailed monitoring (1 minute) costs extra.

Force the alarm without generating load, to see the notification:

```bash
aws cloudwatch set-alarm-state --alarm-name monitor-lab-cpu-high \
  --state-value ALARM --state-reason "Manual test"
```

You should receive an email. The state returns to `OK` or `INSUFFICIENT_DATA` at the next evaluation.

### Step 3: Logs, metric filter and a log-based alarm

Console: CloudWatch > Log groups > Create log group `/lab/app`, retention 1 day. Open it > Metric filters > Create metric filter. Pattern `ERROR`, namespace `LabApp`, metric name `ErrorCount`, value `1`.

```bash
aws logs create-log-group --log-group-name /lab/app
aws logs put-retention-policy --log-group-name /lab/app --retention-in-days 1

aws logs put-metric-filter --log-group-name /lab/app \
  --filter-name error-count --filter-pattern "ERROR" \
  --metric-transformations metricName=ErrorCount,metricNamespace=LabApp,metricValue=1

aws logs create-log-stream --log-group-name /lab/app --log-stream-name s1
NOW=$(($(date +%s)*1000))
aws logs put-log-events --log-group-name /lab/app --log-stream-name s1 \
  --log-events timestamp=$NOW,message="ERROR payment failed order=42" \
               timestamp=$NOW,message="INFO order ok order=43"

aws cloudwatch put-metric-alarm --alarm-name app-errors \
  --namespace LabApp --metric-name ErrorCount --statistic Sum \
  --period 60 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching --alarm-actions $TOPIC_ARN
```

Metric filters only count events ingested after the filter exists. Wait 1-2 minutes, then check:

```bash
aws cloudwatch get-metric-statistics --namespace LabApp --metric-name ErrorCount \
  --start-time $(date -u -v-15M +%FT%TZ 2>/dev/null || date -u -d '-15 min' +%FT%TZ) \
  --end-time $(date -u +%FT%TZ) --period 60 --statistics Sum
```

### Step 4: CloudTrail event history lookup

Console: CloudTrail > Event history. Filter by Event name `RunInstances` or by Event source `ec2.amazonaws.com`. Open an event and read `userIdentity`, `sourceIPAddress` and `requestParameters`.

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --max-results 5 \
  --query 'Events[].[EventTime,Username,EventName]' --output text
```

Event history covers the last 90 days of management events in the current region, for free, but it cannot be queried with SQL and it is not a durable archive.

### Step 5: Create a trail that writes to S3

Console: CloudTrail > Trails > Create trail. Name `lab-trail`, create a new S3 bucket, Log file validation enabled, management events Read and Write.

```bash
aws s3 mb s3://$BUCKET --region $AWS_REGION

cat > bucket-policy.json <<EOF
{"Version":"2012-10-17","Statement":[
 {"Sid":"AclCheck","Effect":"Allow","Principal":{"Service":"cloudtrail.amazonaws.com"},
  "Action":"s3:GetBucketAcl","Resource":"arn:aws:s3:::$BUCKET"},
 {"Sid":"Write","Effect":"Allow","Principal":{"Service":"cloudtrail.amazonaws.com"},
  "Action":"s3:PutObject","Resource":"arn:aws:s3:::$BUCKET/AWSLogs/$ACCOUNT_ID/*",
  "Condition":{"StringEquals":{"s3:x-amz-acl":"bucket-owner-full-control"}}}]}
EOF
aws s3api put-bucket-policy --bucket $BUCKET --policy file://bucket-policy.json

aws cloudtrail create-trail --name lab-trail --s3-bucket-name $BUCKET \
  --enable-log-file-validation
aws cloudtrail start-logging --name lab-trail
```

Generate a recognizable event, then wait about 5-15 minutes for delivery:

```bash
aws ec2 create-tags --resources $INSTANCE_ID --tags Key=audit,Value=test
aws s3 ls s3://$BUCKET/AWSLogs/$ACCOUNT_ID/CloudTrail/ --recursive | head
```

### Step 6: Audit and failure experiment

1. Stop the instance, then find who did it:

```bash
aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=StopInstances \
  --max-results 1
```

2. Try to tamper with the audit trail's integrity: `aws cloudtrail stop-logging --name lab-trail`, then look up `StopLogging` in event history. The action that disabled auditing is itself audited. Restart logging with `start-logging`.

3. Validate log integrity (digest files are delivered hourly, so run this after an hour or skip):

```bash
aws cloudtrail validate-logs --trail-arn arn:aws:cloudtrail:$AWS_REGION:$ACCOUNT_ID:trail/lab-trail \
  --start-time $(date -u -v-2H +%FT%TZ 2>/dev/null || date -u -d '-2 hours' +%FT%TZ)
```

### Step 7 (Optional): AWS Config managed rule

Console: AWS Config > Get started (record specific resource types only, for example S3 buckets, to limit cost) > Rules > Add rule > `s3-bucket-public-read-prohibited`. This flags non-compliant S3 buckets and shows a configuration timeline. Config needs a delivery bucket and a service role, which the setup wizard creates. Delete the recorder and rules in cleanup, since recording is billed per configuration item.

## Validation

- An email arrived from SNS for the manual alarm test.
- `LabApp/ErrorCount` shows a datapoint of 1 (only the `ERROR` line counted).
- Event history shows `RunInstances` and `StopInstances` with your IAM identity.
- Log files appear under `AWSLogs/<ACCOUNT_ID>/CloudTrail/us-east-1/` in the S3 bucket.

## Cleanup

```bash
aws cloudtrail delete-trail --name lab-trail
aws s3 rm s3://$BUCKET --recursive
aws s3 rb s3://$BUCKET
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws cloudwatch delete-alarms --alarm-names monitor-lab-cpu-high app-errors
aws logs delete-metric-filter --log-group-name /lab/app --filter-name error-count
aws logs delete-log-group --log-group-name /lab/app
aws sns delete-topic --topic-arn $TOPIC_ARN
```

If you enabled AWS Config: stop the configuration recorder, delete the delivery channel, recorder and rules, and delete its S3 bucket.

## Exam Takeaway

- CloudWatch answers "how is it performing" (metrics, logs, alarms); CloudTrail answers "who did what" (API audit); Config answers "what does the configuration look like and is it compliant".
- Alarms notify through SNS (or trigger Auto Scaling or EC2 actions); metric filters turn log text into metrics you can alarm on.
- CloudTrail event history is 90 days of management events only; for longer retention, data events, Athena analysis or multi-account aggregation, create a trail to S3 (with log file validation, SSE and optionally CloudWatch Logs delivery).
- Memory and disk usage are not EC2 default metrics; they need the CloudWatch agent.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
