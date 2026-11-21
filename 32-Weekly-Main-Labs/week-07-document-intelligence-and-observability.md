# Week 7 Main Lab: Document Intelligence and Observability Pipeline

## Objective

Build a serverless document intake pipeline end to end. Documents arrive in S3 (new uploads plus a DataSync backfill of "legacy" documents from a second bucket), an S3 event invokes Lambda, Lambda calls Amazon Textract (OCR) and Amazon Comprehend (language, entities, sentiment), and the results land in DynamoDB. Around the pipeline you add observability and audit: a CloudWatch Logs metric filter, an alarm that emails you through SNS, a dashboard, and a CloudTrail trail that shows who called Textract. Finally you break the pipeline on purpose with an unsupported file and watch the alarm fire.

Target time: 90 to 120 minutes. Region: `us-east-1`. Personal AWS account.

## Modules Covered

- [19 - Machine Learning and AI](../19-Machine-Learning-e-IA/README.md): Textract and Comprehend as pre-trained AI services (no model to train).
- [20 - Monitoring (CloudWatch and CloudTrail)](../20-Monitoramento-CloudWatch-CloudTrail/README.md): logs, metric filters, alarms, dashboards, and API auditing.
- [21 - Migration and Transfer](../21-Migracao-e-Transferencia/README.md): AWS DataSync to move the legacy documents into the intake bucket.

## Scenario and Architecture

A law firm is digitizing contracts. New scans are uploaded all day, and 10 years of older scans sit in a separate "legacy" bucket that must be pulled into the same pipeline. The firm needs extracted text, detected entities (people, organizations, dates), and sentiment stored in a queryable table. Operations wants to be paged by email when processing fails, and compliance wants proof of which identity called the OCR service.

```
                       +--------------------+
   legacy bucket ----> |  AWS DataSync task | ---+
   (legacy/*)          |  (S3 -> S3, no     |    |
                       |   agent needed)    |    |  s3://intake/backfill/
                       +--------------------+    |
                                                 v
 uploader --> s3://intake/incoming/ ---> [ S3 intake bucket ]
                                                 |  ObjectCreated event
                                                 v
                                      +----------------------+
                                      |  Lambda (python3.12) |--> Amazon Textract (DetectDocumentText)
                                      |  doc-processor       |--> Amazon Comprehend (language, entities, sentiment)
                                      +----------+-----------+
                                                 |
                  +------------------------------+----------------------+
                  v                              v                      v
        DynamoDB table                   CloudWatch Logs         CloudTrail trail
        DocResults (on-demand)           /aws/lambda/doc-...     (management events)
                                              |                      |
                                   metric filter "ERROR"             v
                                              v                S3 trail bucket
                                   Metric DocIntake/DocErrors   (who called Textract)
                                              v
                                   Alarm --> SNS topic --> your email
                                              |
                                   CloudWatch dashboard (Invocations, Errors, Duration, DocErrors)
```

## Services Used

- Amazon S3 (intake bucket, legacy bucket, trail bucket)
- AWS DataSync (S3-to-S3 location pair and task)
- AWS Lambda, AWS IAM
- Amazon Textract, Amazon Comprehend
- Amazon DynamoDB (on-demand)
- Amazon CloudWatch (Logs, metric filter, alarm, dashboard) and Amazon SNS
- AWS CloudTrail

## Prerequisites

- Personal AWS account, region `us-east-1`, an admin-level IAM identity, AWS CLI v2, and `zip`.
- A real email address you can read (for the SNS subscription).
- Three or four sample documents: JPEG, PNG, or single-page PDF files containing printed English text (a screenshot of a web page works). Also one plain `.txt` file for the failure experiment. Do not use real confidential documents.
- A plan to finish and clean up the same day.

Set shell variables once (use lowercase, globally unique bucket names):

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SUFFIX=<your-initials>-$RANDOM
export INTAKE=docintake-$SUFFIX
export LEGACY=doclegacy-$SUFFIX
export TRAILB=doctrail-$SUFFIX
export EMAIL=<you@example.com>
```

## Expected Cost

All numbers are approximate for `us-east-1` and must be confirmed on the AWS pricing pages (Textract, Comprehend, DataSync, CloudTrail, CloudWatch, DynamoDB).

- Amazon Textract `DetectDocumentText`: roughly $1.50 per 1,000 pages (first tier). Ten test pages cost about a cent. A free tier for new accounts may cover the first pages each month; check the Textract pricing page. Note that `AnalyzeDocument` (forms, tables) costs considerably more per page, so do not switch to it casually.
- Amazon Comprehend: billed per unit, where one unit is 100 characters, with a 3-unit minimum per request. Roughly $0.0001 per unit per feature (entities, sentiment, language). Three calls on a one-page document cost a fraction of a cent.
- AWS DataSync: roughly $0.0125 per GB copied. A few small files are effectively free.
- Lambda, DynamoDB on-demand, SNS, S3: pennies at this volume (often inside free tiers).
- CloudWatch: logs ingestion, a custom metric (about $0.30 per metric per month, prorated), one alarm, and a dashboard (the first three dashboards are free on most accounts; confirm in the pricing page).
- CloudTrail: the first copy of management events delivered to S3 is free; you pay S3 storage for the log files.

Nothing in this lab bills by the hour while idle, but delete the dashboard, alarm, metric, and trail at the end anyway. Set a $5 budget alert first (Billing > Budgets). Total for the session should be well under $1.

## Step by Step

### Step 1: Create the buckets, SNS topic, and DynamoDB table

```bash
aws s3api create-bucket --bucket $INTAKE --region $AWS_REGION
aws s3api create-bucket --bucket $LEGACY --region $AWS_REGION
aws s3api create-bucket --bucket $TRAILB --region $AWS_REGION

for B in $INTAKE $LEGACY $TRAILB; do
  aws s3api put-public-access-block --bucket $B --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
done

aws dynamodb create-table --table-name DocResults \
  --attribute-definitions AttributeName=docId,AttributeType=S \
  --key-schema AttributeName=docId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

TOPIC_ARN=$(aws sns create-topic --name doc-intake-alerts --query TopicArn --output text)
aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint $EMAIL
```

Open your inbox and click "Confirm subscription". An unconfirmed subscription never delivers alarm emails.

### Step 2: Create the Lambda execution role (least privilege)

```bash
cat > trust-lambda.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name doc-processor-role \
  --assume-role-policy-document file://trust-lambda.json

aws iam attach-role-policy --role-name doc-processor-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

cat > doc-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": "s3:GetObject",
     "Resource": "arn:aws:s3:::$INTAKE/*"},
    {"Effect": "Allow",
     "Action": ["textract:DetectDocumentText"],
     "Resource": "*"},
    {"Effect": "Allow",
     "Action": ["comprehend:DetectDominantLanguage","comprehend:DetectEntities","comprehend:DetectSentiment"],
     "Resource": "*"},
    {"Effect": "Allow", "Action": "dynamodb:PutItem",
     "Resource": "arn:aws:dynamodb:us-east-1:$ACCOUNT_ID:table/DocResults"}
  ]
}
EOF

aws iam put-role-policy --role-name doc-processor-role \
  --policy-name doc-processor-inline --policy-document file://doc-policy.json
```

Textract and Comprehend synchronous actions do not support resource-level restrictions, so `Resource: "*"` is expected here. Wait about 10 seconds for IAM propagation before creating the function.

### Step 3: Write and deploy the Lambda function

Create `lambda_function.py`:

```python
import json
import logging
import os
import urllib.parse
from decimal import Decimal

import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

textract = boto3.client("textract")
comprehend = boto3.client("comprehend")
table = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])

# Languages supported by Comprehend sentiment (check current docs before relying on this list).
SENTIMENT_LANGS = {"en", "es", "fr", "de", "it", "pt", "ar", "hi", "ja", "ko", "zh", "zh-TW"}


def handler(event, context):
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
        if key.endswith("/"):
            logger.info("Skipping folder placeholder %s", key)
            continue
        try:
            process(bucket, key)
        except Exception as exc:
            # The word ERROR at the start is what the metric filter matches.
            logger.error("ERROR processing s3://%s/%s : %s", bucket, key, exc)
            raise


def process(bucket, key):
    resp = textract.detect_document_text(
        Document={"S3Object": {"Bucket": bucket, "Name": key}}
    )
    lines = [b["Text"] for b in resp["Blocks"] if b["BlockType"] == "LINE"]
    text = "\n".join(lines)
    if not text.strip():
        raise ValueError("Textract returned no text")

    # Comprehend synchronous APIs accept up to 5,000 bytes of UTF-8 text per request.
    sample = text.encode("utf-8")[:4500].decode("utf-8", errors="ignore")

    lang = comprehend.detect_dominant_language(Text=sample)["Languages"][0]["LanguageCode"]
    item = {
        "docId": f"{bucket}/{key}",
        "language": lang,
        "lineCount": len(lines),
        "preview": text[:300],
    }
    if lang in SENTIMENT_LANGS:
        ents = comprehend.detect_entities(Text=sample, LanguageCode=lang)["Entities"]
        sent = comprehend.detect_sentiment(Text=sample, LanguageCode=lang)
        item["sentiment"] = sent["Sentiment"]
        item["entities"] = [
            {"text": e["Text"], "type": e["Type"], "score": Decimal(str(round(e["Score"], 3)))}
            for e in ents[:15]
        ]
    else:
        item["sentiment"] = "UNSUPPORTED_LANGUAGE"
    table.put_item(Item=item)
    logger.info("Processed s3://%s/%s lang=%s lines=%d", bucket, key, lang, len(lines))
```

DynamoDB does not accept Python floats, which is why scores are converted to `Decimal`.

```bash
zip function.zip lambda_function.py

ROLE_ARN=arn:aws:iam::$ACCOUNT_ID:role/doc-processor-role
aws lambda create-function --function-name doc-processor \
  --runtime python3.12 --handler lambda_function.handler \
  --role $ROLE_ARN --zip-file fileb://function.zip \
  --timeout 60 --memory-size 256 \
  --environment Variables={TABLE_NAME=DocResults}

# S3 invokes Lambda asynchronously and retries failures twice by default.
# Disable retries so one bad file produces one clear error, not three.
aws lambda put-function-event-invoke-config --function-name doc-processor \
  --maximum-retry-attempts 0
```

### Step 4: Wire the S3 event trigger

```bash
aws lambda add-permission --function-name doc-processor \
  --statement-id s3-invoke --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::$INTAKE --source-account $ACCOUNT_ID

LAMBDA_ARN=arn:aws:lambda:us-east-1:$ACCOUNT_ID:function:doc-processor
cat > notification.json <<EOF
{
  "LambdaFunctionConfigurations": [
    {"Id": "incoming", "LambdaFunctionArn": "$LAMBDA_ARN",
     "Events": ["s3:ObjectCreated:*"],
     "Filter": {"Key": {"FilterRules": [{"Name": "prefix", "Value": "incoming/"}]}}},
    {"Id": "backfill", "LambdaFunctionArn": "$LAMBDA_ARN",
     "Events": ["s3:ObjectCreated:*"],
     "Filter": {"Key": {"FilterRules": [{"Name": "prefix", "Value": "backfill/"}]}}}
  ]
}
EOF
aws s3api put-bucket-notification-configuration --bucket $INTAKE \
  --notification-configuration file://notification.json
```

The two prefixes do not overlap, which S3 requires for configurations that target the same event type. The function never writes back to the intake bucket, so there is no recursive-invocation loop.

### Step 5: Observability, part 1: metric filter, alarm, SNS

```bash
aws logs create-log-group --log-group-name /aws/lambda/doc-processor 2>/dev/null || true

aws logs put-metric-filter --log-group-name /aws/lambda/doc-processor \
  --filter-name doc-errors \
  --filter-pattern '"ERROR"' \
  --metric-transformations metricName=DocErrors,metricNamespace=DocIntake,metricValue=1

aws cloudwatch put-metric-alarm --alarm-name doc-intake-errors \
  --namespace DocIntake --metric-name DocErrors \
  --statistic Sum --period 60 --evaluation-periods 1 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions $TOPIC_ARN
```

Lambda creates its log group automatically on first invocation; creating it earlier is harmless and lets the filter exist before the first run. A metric filter only counts log events ingested after it is created. `notBreaching` keeps the alarm in OK while there are no errors (no data).

### Step 6: Observability, part 2: dashboard

```bash
cat > dashboard.json <<'EOF'
{
  "widgets": [
    {"type": "metric", "x": 0, "y": 0, "width": 12, "height": 6,
     "properties": {"title": "Lambda invocations and errors", "region": "us-east-1", "stat": "Sum", "period": 60,
       "metrics": [["AWS/Lambda", "Invocations", "FunctionName", "doc-processor"],
                   ["AWS/Lambda", "Errors", "FunctionName", "doc-processor"]]}},
    {"type": "metric", "x": 12, "y": 0, "width": 12, "height": 6,
     "properties": {"title": "Lambda duration (ms)", "region": "us-east-1", "stat": "Average", "period": 60,
       "metrics": [["AWS/Lambda", "Duration", "FunctionName", "doc-processor"]]}},
    {"type": "metric", "x": 0, "y": 6, "width": 12, "height": 6,
     "properties": {"title": "Pipeline errors from logs (metric filter)", "region": "us-east-1", "stat": "Sum", "period": 60,
       "metrics": [["DocIntake", "DocErrors"]]}},
    {"type": "metric", "x": 12, "y": 6, "width": 12, "height": 6,
     "properties": {"title": "DynamoDB write capacity consumed", "region": "us-east-1", "stat": "Sum", "period": 60,
       "metrics": [["AWS/DynamoDB", "ConsumedWriteCapacityUnits", "TableName", "DocResults"]]}}
  ]
}
EOF
aws cloudwatch put-dashboard --dashboard-name doc-intake --dashboard-body file://dashboard.json
```

Console: CloudWatch > Dashboards > `doc-intake`.

### Step 7: Audit with CloudTrail

Create the trail bucket policy that allows CloudTrail to write logs:

```bash
cat > trail-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Sid": "AclCheck", "Effect": "Allow",
     "Principal": {"Service": "cloudtrail.amazonaws.com"},
     "Action": "s3:GetBucketAcl", "Resource": "arn:aws:s3:::$TRAILB",
     "Condition": {"StringEquals": {"aws:SourceArn": "arn:aws:cloudtrail:us-east-1:$ACCOUNT_ID:trail/doc-intake-trail"}}},
    {"Sid": "Write", "Effect": "Allow",
     "Principal": {"Service": "cloudtrail.amazonaws.com"},
     "Action": "s3:PutObject", "Resource": "arn:aws:s3:::$TRAILB/AWSLogs/$ACCOUNT_ID/*",
     "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control",
       "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:$ACCOUNT_ID:trail/doc-intake-trail"}}}
  ]
}
EOF
aws s3api put-bucket-policy --bucket $TRAILB --policy file://trail-policy.json

aws cloudtrail create-trail --name doc-intake-trail --s3-bucket-name $TRAILB \
  --is-multi-region-trail --enable-log-file-validation
aws cloudtrail start-logging --name doc-intake-trail
```

Textract calls are recorded as management events, so no extra data-event configuration (and no extra cost) is needed to see them. The CloudTrail Event history (last 90 days) also shows them without any trail; the trail adds durable delivery to S3 and log file validation. A trail with a single copy of management events is free apart from S3 storage, but verify on the CloudTrail pricing page, because additional trails and data events are charged.

### Step 8: Backfill legacy documents with DataSync

Upload your sample files to the legacy bucket:

```bash
aws s3 cp ./samples/ s3://$LEGACY/legacy/ --recursive --exclude "*.txt"
```

Create an IAM role that DataSync can assume to access both buckets:

```bash
cat > trust-datasync.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "datasync.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF
aws iam create-role --role-name datasync-s3-role --assume-role-policy-document file://trust-datasync.json

cat > datasync-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow",
     "Action": ["s3:GetBucketLocation","s3:ListBucket","s3:ListBucketMultipartUploads"],
     "Resource": ["arn:aws:s3:::$LEGACY","arn:aws:s3:::$INTAKE"]},
    {"Effect": "Allow",
     "Action": ["s3:AbortMultipartUpload","s3:DeleteObject","s3:GetObject","s3:ListMultipartUploadParts","s3:PutObject","s3:GetObjectTagging","s3:PutObjectTagging"],
     "Resource": ["arn:aws:s3:::$LEGACY/*","arn:aws:s3:::$INTAKE/*"]}
  ]
}
EOF
aws iam put-role-policy --role-name datasync-s3-role --policy-name datasync-s3 \
  --policy-document file://datasync-policy.json
```

Create the two locations and the task. Both buckets are in the same Region, so no agent is required.

```bash
SRC=$(aws datasync create-location-s3 --s3-bucket-arn arn:aws:s3:::$LEGACY \
  --subdirectory /legacy/ \
  --s3-config BucketAccessRoleArn=arn:aws:iam::$ACCOUNT_ID:role/datasync-s3-role \
  --query LocationArn --output text)

DST=$(aws datasync create-location-s3 --s3-bucket-arn arn:aws:s3:::$INTAKE \
  --subdirectory /backfill/ \
  --s3-config BucketAccessRoleArn=arn:aws:iam::$ACCOUNT_ID:role/datasync-s3-role \
  --query LocationArn --output text)

TASK=$(aws datasync create-task --name legacy-backfill \
  --source-location-arn $SRC --destination-location-arn $DST \
  --options VerifyMode=ONLY_FILES_TRANSFERRED,TransferMode=CHANGED \
  --query TaskArn --output text)

EXEC=$(aws datasync start-task-execution --task-arn $TASK \
  --query TaskExecutionArn --output text)

aws datasync describe-task-execution --task-execution-arn $EXEC \
  --query '[Status,FilesTransferred,BytesTransferred]'
```

Repeat the last command until `Status` is `SUCCESS` (usually a minute or two). Console equivalent: DataSync > Tasks > `legacy-backfill` > History. If the first role-based call fails because of IAM propagation, wait a minute and retry. DataSync may create zero-byte "folder" placeholder objects; the Lambda code skips keys ending in `/`.

### Step 9: Push live documents through the pipeline

```bash
aws s3 cp ./samples/<file1>.png s3://$INTAKE/incoming/<file1>.png
aws s3 cp ./samples/<file2>.pdf s3://$INTAKE/incoming/<file2>.pdf   # single-page PDF only

sleep 15
aws logs tail /aws/lambda/doc-processor --since 5m
aws dynamodb scan --table-name DocResults \
  --projection-expression "docId, #l, sentiment, lineCount" \
  --expression-attribute-names '{"#l":"language"}'
```

You should see one item per backfilled file (keys under `backfill/`) and one per live upload (keys under `incoming/`).

### Step 10: Failure experiment: an unsupported file

Textract's synchronous `DetectDocumentText` accepts JPEG, PNG, PDF (single page), and TIFF. A text file should be rejected.

```bash
echo "this is not a scan" > not-a-scan.txt
aws s3 cp not-a-scan.txt s3://$INTAKE/incoming/not-a-scan.txt

sleep 30
aws logs tail /aws/lambda/doc-processor --since 2m | grep -i error
aws cloudwatch describe-alarms --alarm-names doc-intake-errors \
  --query 'MetricAlarms[0].[StateValue,StateReason]'
```

Expected: the log shows `ERROR processing s3://.../incoming/not-a-scan.txt` with an `UnsupportedDocumentException` message, the metric `DocIntake/DocErrors` records 1, the alarm moves from `OK` to `ALARM` within a minute or two, and an email arrives from SNS. The AWS/Lambda `Errors` line on the dashboard also ticks up. No DynamoDB item is written for that file, and the valid documents are unaffected.

### Step 11: Audit who called Textract

CloudTrail events can lag several minutes behind the call.

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DetectDocumentText \
  --max-results 5 \
  --query 'Events[].[EventTime,Username,EventSource]' --output table
```

Open one event in the console (CloudTrail > Event history) and read `userIdentity`: the principal is an `AssumedRole` for `doc-processor-role` with session name `doc-processor`. That proves the Lambda role, not a human user, made the call. Try the same lookup for `DetectEntities` (Comprehend). The failed Textract call for the `.txt` file should also be present with an `errorCode`.

## Validation

- DynamoDB `DocResults` holds one item for each valid document, from both `backfill/` and `incoming/` prefixes, with `language`, `sentiment`, and `entities`.
- `aws datasync describe-task-execution` reports `SUCCESS` with the expected file count.
- Alarm `doc-intake-errors` went to `ALARM` after the bad file and you received the SNS email. Reset by waiting for it to return to `OK`.
- Dashboard `doc-intake` shows invocations, one error, duration, and the filter metric.
- CloudTrail lookup shows `DetectDocumentText` called by the `doc-processor-role` session. After about 5 to 15 minutes, log files also appear under `s3://$TRAILB/AWSLogs/$ACCOUNT_ID/CloudTrail/`.

## Cleanup

Delete in dependency order, same day:

```bash
# Trigger and function first so nothing processes while you delete
aws s3api put-bucket-notification-configuration --bucket $INTAKE --notification-configuration '{}'
aws lambda delete-function --function-name doc-processor

# Observability
aws cloudwatch delete-alarms --alarm-names doc-intake-errors
aws cloudwatch delete-dashboards --dashboard-names doc-intake
aws logs delete-metric-filter --log-group-name /aws/lambda/doc-processor --filter-name doc-errors
aws logs delete-log-group --log-group-name /aws/lambda/doc-processor
aws sns delete-topic --topic-arn $TOPIC_ARN

# CloudTrail
aws cloudtrail stop-logging --name doc-intake-trail
aws cloudtrail delete-trail --name doc-intake-trail

# DataSync (task before locations)
aws datasync delete-task --task-arn $TASK
aws datasync delete-location --location-arn $SRC
aws datasync delete-location --location-arn $DST

# Data
aws dynamodb delete-table --table-name DocResults
for B in $INTAKE $LEGACY $TRAILB; do
  aws s3 rm s3://$B --recursive
  aws s3api delete-bucket --bucket $B --region us-east-1
done

# IAM
aws iam delete-role-policy --role-name doc-processor-role --policy-name doc-processor-inline
aws iam detach-role-policy --role-name doc-processor-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name doc-processor-role
aws iam delete-role-policy --role-name datasync-s3-role --policy-name datasync-s3
aws iam delete-role --role-name datasync-s3-role
```

None of these buckets are versioned, so `rm --recursive` empties them fully. The custom metric stops receiving data once the filter is deleted and ages out of the console after its retention period.

## Exam Takeaway

- Pre-trained AI services are the right answer when the question says "no ML expertise" or "no model training": Textract (OCR, forms, tables), Comprehend (NLP: entities, sentiment, language, PII), Rekognition (images/video), Transcribe (speech to text), Translate, Polly, Lex, Kendra. Textract answers "extract text/data from scanned documents", Comprehend answers "analyze text".
- Decoupled event-driven design (S3 event, Lambda, DynamoDB) is the standard serverless, cost-efficient pattern. For retries or back-pressure, put SQS between S3 and Lambda, and use a dead-letter queue or on-failure destination for failures.
- CloudWatch vs CloudTrail: CloudWatch is performance and operations (metrics, logs, alarms); CloudTrail is who did what via the API (audit). "Who deleted/called X" means CloudTrail. Metric filter turns log text into a metric you can alarm on; alarms notify through SNS.
- DataSync is for online, scheduled, verified transfers between on-premises/other clouds/AWS storage (S3, EFS, FSx) and is faster to operate than hand-written scripts. Compare: Snowball Edge for offline bulk transfer when the network is too slow, Storage Gateway for hybrid access to cloud storage, Transfer Family for SFTP/FTPS into S3 or EFS, `aws s3 sync` for simple one-off copies, DMS for databases.
- Domain mapping: Design Secure Architectures (least-privilege roles, CloudTrail auditing); Design Resilient Architectures (decoupling, retries, alarms); Design High-Performing Architectures (serverless, managed AI services, DataSync throughput); Design Cost-Optimized Architectures (on-demand DynamoDB, pay-per-use Textract/Comprehend, free first trail).

## Stretch Goals

1. Add an SQS queue between S3 and Lambda with a dead-letter queue and `ReportBatchItemFailures`, then re-run the failure experiment and observe the message in the DLQ instead of a lost event.
2. Add an Amazon Comprehend PII detection call (`detect_pii_entities`) and store only a redacted preview, then restrict the DynamoDB table with a KMS customer managed key.
3. Switch Textract to the asynchronous `StartDocumentTextDetection` API for multi-page PDFs, with an SNS completion topic.
4. Send the CloudTrail trail to CloudWatch Logs and create a metric filter that alerts when anyone other than `doc-processor-role` calls Textract.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
