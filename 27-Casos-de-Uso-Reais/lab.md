# Guided Lab: Serverless Photo-Upload Pipeline (S3 -> Lambda -> DynamoDB + SNS)

## Objective

Build a small event-driven pipeline end to end: a photo uploaded to S3 triggers a Lambda function, which stores the photo's metadata in DynamoDB and publishes a notification to SNS. You will wire it with a least-privilege IAM role and confirm it works through CloudWatch Logs.

## Scenario

A small photography startup lets customers drop photos into a bucket. The team wants every upload recorded (key, size, upload time) in a database and wants an email when a new photo arrives. They do not want to manage servers, and the function must only be allowed to do exactly what it needs.

```
Upload --> S3 (uploads/) --ObjectCreated--> Lambda --> DynamoDB (PhotoMetadata)
                                               |------> SNS topic --> email
                                               '------> CloudWatch Logs
```

## Services Used

- Amazon S3 (event notifications)
- AWS Lambda (Python 3.12)
- Amazon DynamoDB (on-demand)
- Amazon SNS (email subscription)
- AWS IAM (execution role, resource policy on the function)
- Amazon CloudWatch Logs

## Prerequisites

- A personal AWS account, region `us-east-1`, with an admin-level IAM user or role (not the root user).
- AWS CLI v2 configured (`aws sts get-caller-identity` works) and `zip` available.
- An email address you can read.
- Shell variables used below:

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET=photo-pipeline-lab-$ACCOUNT_ID
```

## Expected Cost

Effectively $0 for a few test uploads. Lambda, DynamoDB on-demand, SNS email, and S3 request/storage charges at this volume are far below a cent (Lambda and DynamoDB also have always-free tiers). Things to avoid: uploading large files, leaving the bucket full, and writing the function's output back into the same S3 prefix that triggers it (a recursive loop that can run up cost fast).

## Step by Step

### Step 1: Create the bucket and the DynamoDB table

Console: S3 > Create bucket (keep Block Public Access on) / DynamoDB > Create table, partition key `photoKey` (String), capacity mode On-demand.

```bash
aws s3api create-bucket --bucket $BUCKET --region us-east-1

aws dynamodb create-table \
  --table-name PhotoMetadata \
  --attribute-definitions AttributeName=photoKey,AttributeType=S \
  --key-schema AttributeName=photoKey,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Step 2: Create the SNS topic and subscribe your email

Console: SNS > Topics > Create topic (Standard) > Create subscription > Protocol Email.

```bash
TOPIC_ARN=$(aws sns create-topic --name photo-uploaded --query TopicArn --output text)
aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint you@example.com
```

Open the confirmation email and click "Confirm subscription". Unconfirmed subscriptions receive nothing.

### Step 3: Create a least-privilege execution role

The role needs a trust policy for Lambda, the AWS managed `AWSLambdaBasicExecutionRole` (CloudWatch Logs), and one small inline policy scoped to your three resources.

```bash
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name photo-pipeline-role \
  --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name photo-pipeline-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

cat > inline.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": "s3:GetObject",
     "Resource": "arn:aws:s3:::$BUCKET/uploads/*"},
    {"Effect": "Allow", "Action": "dynamodb:PutItem",
     "Resource": "arn:aws:dynamodb:us-east-1:$ACCOUNT_ID:table/PhotoMetadata"},
    {"Effect": "Allow", "Action": "sns:Publish",
     "Resource": "$TOPIC_ARN"}
  ]
}
EOF
aws iam put-role-policy --role-name photo-pipeline-role \
  --policy-name photo-pipeline-access --policy-document file://inline.json
```

Note there are no wildcards on actions and no `Resource: "*"`. The function only reads from `uploads/`, writes one table, and publishes to one topic.

### Step 4: Write and deploy the Lambda function

```python
# lambda_function.py
import os, urllib.parse, datetime
import boto3

s3 = boto3.client("s3")
ddb = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])
sns = boto3.client("sns")

def handler(event, context):
    for rec in event["Records"]:
        bucket = rec["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(rec["s3"]["object"]["key"])
        head = s3.head_object(Bucket=bucket, Key=key)
        ddb.put_item(Item={
            "photoKey": key,
            "sizeBytes": head["ContentLength"],
            "contentType": head.get("ContentType", "unknown"),
            "uploadedAt": datetime.datetime.utcnow().isoformat() + "Z",
        })
        sns.publish(
            TopicArn=os.environ["TOPIC_ARN"],
            Subject="New photo uploaded",
            Message=f"s3://{bucket}/{key} ({head['ContentLength']} bytes)",
        )
        print(f"Processed {key}")
    return {"status": "ok"}
```

`head_object` needs `s3:GetObject` permission on the key, which the role already has. The `put_item` call is idempotent by key: S3 can deliver an event more than once, and a repeat just overwrites the same item.

```bash
zip function.zip lambda_function.py
sleep 10   # let the new role propagate
aws lambda create-function \
  --function-name photo-processor \
  --runtime python3.12 \
  --handler lambda_function.handler \
  --role arn:aws:iam::$ACCOUNT_ID:role/photo-pipeline-role \
  --zip-file fileb://function.zip \
  --timeout 15 \
  --environment "Variables={TABLE_NAME=PhotoMetadata,TOPIC_ARN=$TOPIC_ARN}"
```

### Step 5: Allow S3 to invoke the function, then add the trigger

S3 is a different principal than your role, so the function needs a resource-based policy. Do this before configuring the notification, otherwise S3 rejects the configuration.

```bash
aws lambda add-permission \
  --function-name photo-processor \
  --statement-id s3-invoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::$BUCKET \
  --source-account $ACCOUNT_ID

cat > notif.json <<EOF
{
  "LambdaFunctionConfigurations": [{
    "LambdaFunctionArn": "arn:aws:lambda:us-east-1:$ACCOUNT_ID:function:photo-processor",
    "Events": ["s3:ObjectCreated:*"],
    "Filter": {"Key": {"FilterRules": [{"Name": "prefix", "Value": "uploads/"}]}}
  }]
}
EOF
aws s3api put-bucket-notification-configuration \
  --bucket $BUCKET --notification-configuration file://notif.json
```

Console equivalent: S3 > bucket > Properties > Event notifications > Create event notification.

### Step 6: Upload a photo

```bash
echo "fake image bytes" > test1.jpg
aws s3 cp test1.jpg s3://$BUCKET/uploads/test1.jpg --content-type image/jpeg
```

## Validation

1. DynamoDB item exists:
   ```bash
   aws dynamodb scan --table-name PhotoMetadata
   ```
2. You received the "New photo uploaded" email.
3. Logs show `Processed uploads/test1.jpg`:
   ```bash
   aws logs tail /aws/lambda/photo-processor --since 10m
   ```
4. Least-privilege check: edit the inline policy to remove `sns:Publish`, upload again, and observe an `AccessDenied` error in the logs. Restore the policy afterward.

If nothing happens, check that the prefix is `uploads/` (a file at the bucket root will not trigger) and that the SNS subscription is confirmed.

## Cleanup

```bash
aws s3 rm s3://$BUCKET --recursive
aws s3api delete-bucket --bucket $BUCKET
aws lambda delete-function --function-name photo-processor
aws dynamodb delete-table --table-name PhotoMetadata
aws sns delete-topic --topic-arn $TOPIC_ARN
aws iam delete-role-policy --role-name photo-pipeline-role --policy-name photo-pipeline-access
aws iam detach-role-policy --role-name photo-pipeline-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name photo-pipeline-role
aws logs delete-log-group --log-group-name /aws/lambda/photo-processor
```

## Exam Takeaway

Event-driven, serverless designs (S3 event -> Lambda -> DynamoDB/SNS) are the default answer when the question asks for no server management, automatic scaling, and pay-per-use. Remember the two separate IAM concerns: the function's execution role controls what the function can call, while a resource-based policy (`lambda add-permission`) controls who can invoke it. Scope permissions to specific actions and ARNs rather than `*`. Since S3 delivers events at-least-once, make the handler idempotent. If a question needs buffering or retries for spiky load, think SQS between S3 and Lambda; if it needs several subscribers, think SNS fan-out.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
