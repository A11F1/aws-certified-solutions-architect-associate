# Week 6 Main Lab: Serverless Ingest-to-Analytics Pipeline

## Objective

Build a fully serverless pipeline: an API Gateway HTTP API receives JSON events, a Lambda function validates them and writes valid records to Kinesis Data Firehose, which buffers and delivers JSON files to S3. A Glue crawler catalogs the data, and Athena queries it. Then you convert the data to partitioned Parquet with Athena CTAS and compare the bytes scanned. Invalid payloads are diverted to an SQS "rejects" queue, throttling is tested at the API, and each Lambda gets its own least-privilege role.

Time: 90 to 120 minutes.

## Modules Covered

- [17 - Serverless: Lambda and API Gateway](../17-Serverless-Lambda-API-Gateway/README.md)
- [18 - Data and Analytics](../18-Dados-e-Analytics/README.md)

## Scenario and Architecture

"ShelfCo" installs sensors in its warehouses. Devices post small JSON readings over HTTPS. The team wants zero servers to manage, pay-per-use pricing, and SQL over the data lake. Bad payloads must not be silently dropped or poison the lake.

```
 Device / curl
     | POST /events  (JSON)
     v
 +---------------------+  throttling (rate/burst)  -> 429 when exceeded
 | API Gateway HTTP API|
 +----------+----------+
            | AWS_PROXY (payload v2.0)
            v
 +------------------------+   invalid JSON / schema   +-------------------------+
 | Lambda lab-ingest      |-------------------------->| SQS lab-ingest-rejects  |
 | role: firehose:PutRecord|     (400 to caller)      +------------+------------+
 |  + sqs:SendMessage      |                                       | event source mapping
 +-----------+------------+                                        v
             | valid (JSON line)                       +-------------------------+
             v                                         | Lambda lab-reject-logger|
 +------------------------+  buffer 60s / 1 MB        | role: sqs receive/delete|
 | Kinesis Data Firehose  |                           +-------------------------+
 | DirectPut -> S3        |
 +-----------+------------+
             v
 +-------------------------------------------------------------+
 | S3 bucket  lab-iot-<ACCOUNT_ID>                             |
 |   events/dt=YYYY-MM-DD/*.json   (raw JSON lines)            |
 |   curated/dt=YYYY-MM-DD/*.parquet (Athena CTAS)             |
 |   athena-results/                                           |
 +--------------+----------------------------------------------+
                | Glue crawler -> Data Catalog (database lab_iot)
                v
          Athena (workgroup lab-wg, scan cutoff) -> compare JSON vs Parquet bytes scanned
```

## Services Used

- Amazon API Gateway (HTTP API)
- AWS Lambda (Python 3.12), IAM execution roles
- Amazon Kinesis Data Firehose (DirectPut delivery stream to S3)
- Amazon S3, AWS Glue (database, crawler, Data Catalog), Amazon Athena (workgroup, CTAS)
- Amazon SQS (rejects queue), Amazon CloudWatch Logs
- AWS CLI v2, `zip`, `curl`, `jq`, Python 3 (for the sample data generator)

## Prerequisites

- Personal AWS account, region `us-east-1`, administrator-level sandbox permissions.
- Athena in `us-east-1` is usable immediately; the workgroup created below stores results in your bucket.
- No VPC is needed.

## Expected Cost

All approximate; confirm on the pricing pages for API Gateway, Lambda, Firehose, S3, Glue and Athena.

- API Gateway HTTP API, Lambda, SQS: fractions of a cent for a few hundred requests, within free tiers where applicable.
- Firehose bills by data ingested (with a per-record size rounding), not by the hour, so a few hundred small records cost far less than a cent. Do not confuse this with Kinesis Data Streams in provisioned mode, which bills per shard-hour. This lab does not use Data Streams. Still delete the stream when done, and never leave a high-volume producer running.
- Glue crawler: billed per DPU-hour with a minimum billed duration per run; a couple of runs is cents. Delete it afterwards.
- Athena: charged per data scanned (roughly $5 per TB, with a 10 MB minimum per query), so this lab costs well under a cent. The workgroup scan limit below is a guardrail.
- S3 storage for about 25 MB: negligible. CloudWatch Logs: negligible.

## Step by Step

### Step 1: Variables, bucket, and rejects queue

```bash
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET=lab-iot-$ACCOUNT_ID

aws s3api create-bucket --bucket $BUCKET --region us-east-1
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws sqs create-queue --queue-name lab-ingest-rejects \
  --attributes MessageRetentionPeriod=345600,VisibilityTimeout=60
export REJECT_URL=$(aws sqs get-queue-url --queue-name lab-ingest-rejects --query QueueUrl --output text)
export REJECT_ARN=arn:aws:sqs:us-east-1:$ACCOUNT_ID:lab-ingest-rejects
```

### Step 2: Firehose delivery role and stream (small buffer)

Trust policy and a least-privilege S3 policy for Firehose:

```bash
cat > firehose-trust.json <<'EOF'
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Principal": { "Service": "firehose.amazonaws.com" }, "Action": "sts:AssumeRole" }] }
EOF
cat > firehose-policy.json <<EOF
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Action": ["s3:AbortMultipartUpload","s3:GetBucketLocation","s3:GetObject",
             "s3:ListBucket","s3:ListBucketMultipartUploads","s3:PutObject"],
  "Resource": ["arn:aws:s3:::$BUCKET","arn:aws:s3:::$BUCKET/*"] }] }
EOF
aws iam create-role --role-name lab-firehose-role --assume-role-policy-document file://firehose-trust.json
aws iam put-role-policy --role-name lab-firehose-role --policy-name s3-delivery \
  --policy-document file://firehose-policy.json
```

Create the stream. Records are written under a Hive-style `dt=` prefix so Glue and Athena see a partition column. The buffer is 1 MB or 60 seconds, whichever comes first, so you do not wait long to see files. `ErrorOutputPrefix` is required when the prefix uses expressions.

```bash
cat > firehose-dest.json <<EOF
{
  "RoleARN": "arn:aws:iam::$ACCOUNT_ID:role/lab-firehose-role",
  "BucketARN": "arn:aws:s3:::$BUCKET",
  "Prefix": "events/dt=!{timestamp:yyyy-MM-dd}/",
  "ErrorOutputPrefix": "firehose-errors/!{firehose:error-output-type}/dt=!{timestamp:yyyy-MM-dd}/",
  "BufferingHints": { "SizeInMBs": 1, "IntervalInSeconds": 60 },
  "CompressionFormat": "UNCOMPRESSED"
}
EOF
sleep 10   # allow IAM propagation
aws firehose create-delivery-stream --delivery-stream-name lab-ingest \
  --delivery-stream-type DirectPut \
  --extended-s3-destination-configuration file://firehose-dest.json

aws firehose describe-delivery-stream --delivery-stream-name lab-ingest \
  --query 'DeliveryStreamDescription.DeliveryStreamStatus'      # wait for ACTIVE
export STREAM_ARN=arn:aws:firehose:us-east-1:$ACCOUNT_ID:deliverystream/lab-ingest
```

### Step 3: Lambda roles, one per function (least privilege)

```bash
cat > lambda-trust.json <<'EOF'
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Principal": { "Service": "lambda.amazonaws.com" }, "Action": "sts:AssumeRole" }] }
EOF

# Role 1: ingest function
aws iam create-role --role-name lab-ingest-role --assume-role-policy-document file://lambda-trust.json
aws iam attach-role-policy --role-name lab-ingest-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
cat > ingest-policy.json <<EOF
{ "Version": "2012-10-17", "Statement": [
  { "Effect": "Allow", "Action": "firehose:PutRecord", "Resource": "$STREAM_ARN" },
  { "Effect": "Allow", "Action": "sqs:SendMessage",    "Resource": "$REJECT_ARN" } ] }
EOF
aws iam put-role-policy --role-name lab-ingest-role --policy-name ingest-access \
  --policy-document file://ingest-policy.json

# Role 2: reject logger (consumes the rejects queue only)
aws iam create-role --role-name lab-reject-role --assume-role-policy-document file://lambda-trust.json
aws iam attach-role-policy --role-name lab-reject-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
cat > reject-policy.json <<EOF
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Action": ["sqs:ReceiveMessage","sqs:DeleteMessage","sqs:GetQueueAttributes"],
  "Resource": "$REJECT_ARN" }] }
EOF
aws iam put-role-policy --role-name lab-reject-role --policy-name reject-queue-access \
  --policy-document file://reject-policy.json
```

### Step 4: The two Lambda functions

`ingest.py` validates the body, sends rejects to SQS (HTTP 400), and writes valid events as newline-delimited JSON to Firehose (HTTP 202). The trailing newline matters: without it, Firehose concatenates records into one line and Athena cannot parse them.

```python
import base64, json, os
import boto3

firehose = boto3.client("firehose")
sqs = boto3.client("sqs")
STREAM = os.environ["STREAM_NAME"]
REJECT_URL = os.environ["REJECT_URL"]

def valid(item):
    return (isinstance(item, dict)
            and isinstance(item.get("deviceId"), str)
            and isinstance(item.get("ts"), str)
            and isinstance(item.get("value"), (int, float)))

def handler(event, context):
    body = event.get("body") or ""
    if event.get("isBase64Encoded"):
        body = base64.b64decode(body).decode("utf-8", "replace")
    try:
        item = json.loads(body)
        if not valid(item):
            raise ValueError("schema")
    except ValueError:
        sqs.send_message(QueueUrl=REJECT_URL, MessageBody=body[:100000])
        return {"statusCode": 400, "body": json.dumps({"error": "invalid payload"})}
    firehose.put_record(DeliveryStreamName=STREAM,
                        Record={"Data": (json.dumps(item) + "\n").encode()})
    return {"statusCode": 202, "body": json.dumps({"status": "accepted"})}
```

`reject_logger.py` simply logs rejected payloads (in real life it would notify or store them):

```python
def handler(event, context):
    for r in event["Records"]:
        print("REJECTED:", r["body"][:500])
```

Deploy both:

```bash
zip ingest.zip ingest.py
zip reject.zip reject_logger.py
sleep 10   # IAM propagation

aws lambda create-function --function-name lab-ingest --runtime python3.12 \
  --handler ingest.handler --zip-file fileb://ingest.zip --timeout 10 --memory-size 128 \
  --role arn:aws:iam::$ACCOUNT_ID:role/lab-ingest-role \
  --environment "Variables={STREAM_NAME=lab-ingest,REJECT_URL=$REJECT_URL}"

aws lambda create-function --function-name lab-reject-logger --runtime python3.12 \
  --handler reject_logger.handler --zip-file fileb://reject.zip --timeout 10 \
  --role arn:aws:iam::$ACCOUNT_ID:role/lab-reject-role

aws lambda create-event-source-mapping --function-name lab-reject-logger \
  --event-source-arn $REJECT_ARN --batch-size 5
```

### Step 5: API Gateway HTTP API

Console: API Gateway > Create API > HTTP API > Build. CLI:

```bash
export API_ID=$(aws apigatewayv2 create-api --name lab-ingest-api --protocol-type HTTP \
  --query ApiId --output text)

export INT_ID=$(aws apigatewayv2 create-integration --api-id $API_ID \
  --integration-type AWS_PROXY --payload-format-version 2.0 \
  --integration-uri arn:aws:lambda:us-east-1:$ACCOUNT_ID:function:lab-ingest \
  --query IntegrationId --output text)

aws apigatewayv2 create-route --api-id $API_ID --route-key "POST /events" --target integrations/$INT_ID
aws apigatewayv2 create-stage --api-id $API_ID --stage-name '$default' --auto-deploy

aws lambda add-permission --function-name lab-ingest --statement-id apigw-invoke \
  --action lambda:InvokeFunction --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:$ACCOUNT_ID:$API_ID/*/*/events"

export URL=https://$API_ID.execute-api.us-east-1.amazonaws.com/events
```

The resource-based permission is what lets API Gateway invoke the function; the execution role is what lets the function call other services. Do not mix them up (a classic exam point).

Smoke test:

```bash
curl -s -X POST $URL -H 'content-type: application/json' \
  -d '{"deviceId":"d-001","ts":"2026-01-01T10:00:00Z","value":21.5}'
# {"status": "accepted"}
```

This endpoint is public and unauthenticated for the lab. Do not leave it up. Production uses an authorizer (IAM, JWT or Lambda) and usage controls.

### Step 6: Send real traffic and verify files land in S3

```bash
for i in $(seq 1 100); do
  curl -s -o /dev/null -X POST $URL -H 'content-type: application/json' \
    -d "{\"deviceId\":\"d-$((i % 10))\",\"ts\":\"2026-01-01T10:$(printf %02d $((i % 60))):00Z\",\"value\":$((20 + i % 7)).5}"
done

sleep 90
aws s3 ls s3://$BUCKET/events/ --recursive
```

Firehose flushes after 60 seconds (or 1 MB), so a file under `events/dt=<today>/` appears within a couple of minutes. If it does not, check `firehose-errors/` and the Firehose monitoring tab.

### Step 7: Failure experiments (invalid payload and throttling)

Experiment 1, invalid payloads are diverted, not lost and not mixed into the lake:

```bash
curl -s -X POST $URL -d 'this is not json'
curl -s -X POST $URL -H 'content-type: application/json' -d '{"deviceId":"d-1","value":"high"}'
# both return {"error": "invalid payload"} with HTTP 400

aws sqs get-queue-attributes --queue-url $REJECT_URL \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
aws logs tail /aws/lambda/lab-reject-logger --since 5m | grep REJECTED
```

The reject logger consumes the queue, so the visible count may already be 0; the proof is the `REJECTED:` log lines and nothing malformed in S3. (Why not the Lambda DLQ feature? A Lambda DLQ or on-failure destination applies to asynchronous invocations. API Gateway calls the function synchronously, so the function itself must route bad input, as done here.)

Experiment 2, throttling at the API:

```bash
aws apigatewayv2 update-stage --api-id $API_ID --stage-name '$default' \
  --default-route-settings ThrottlingBurstLimit=1,ThrottlingRateLimit=1

for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST $URL -H 'content-type: application/json' \
    -d '{"deviceId":"d-99","ts":"2026-01-01T11:00:00Z","value":1.0}' &
done; wait
```

Expect a mix of `202` and `429 Too Many Requests`. Callers should retry with exponential backoff and jitter. Restore sensible limits afterwards:

```bash
aws apigatewayv2 update-stage --api-id $API_ID --stage-name '$default' \
  --default-route-settings ThrottlingBurstLimit=100,ThrottlingRateLimit=50
```

Experiment 3, prove least privilege:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::$ACCOUNT_ID:role/lab-reject-role \
  --action-names firehose:PutRecord \
  --resource-arns $STREAM_ARN --query 'EvaluationResults[0].EvalDecision' --output text
# implicitDeny
```

### Step 8: Add volume so scanned-bytes comparisons are meaningful

Athena rounds every query up to a 10 MB minimum for billing, and tiny files hide the benefit of Parquet. Generate about 300,000 synthetic rows (a few MB per day) under three extra partitions. This is local data written straight to S3 (it does not pass through Firehose).

```bash
cat > gen.py <<'EOF'
import json, random, sys
day = sys.argv[1]
with open(f"bulk-{day}.json", "w") as f:
    for i in range(100000):
        f.write(json.dumps({"deviceId": f"d-{random.randint(0, 999):03d}",
                            "ts": f"{day}T{random.randint(0,23):02d}:{random.randint(0,59):02d}:00Z",
                            "value": round(random.uniform(10, 40), 2),
                            "type": random.choice(["temp", "humidity", "co2"])}) + "\n")
EOF
for d in 2026-01-02 2026-01-03 2026-01-04; do
  python3 gen.py $d
  aws s3 cp bulk-$d.json s3://$BUCKET/events/dt=$d/bulk.json
done
```

Note that the Firehose-written rows lack a `type` field; the crawler will merge the schema (the column is simply null for those rows).

### Step 9: Glue crawler and Data Catalog

```bash
cat > glue-trust.json <<'EOF'
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Principal": { "Service": "glue.amazonaws.com" }, "Action": "sts:AssumeRole" }] }
EOF
cat > glue-s3.json <<EOF
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Action": ["s3:GetObject","s3:ListBucket"],
  "Resource": ["arn:aws:s3:::$BUCKET","arn:aws:s3:::$BUCKET/events/*"] }] }
EOF
aws iam create-role --role-name lab-glue-role --assume-role-policy-document file://glue-trust.json
aws iam attach-role-policy --role-name lab-glue-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
aws iam put-role-policy --role-name lab-glue-role --policy-name read-events --policy-document file://glue-s3.json
sleep 10

aws glue create-database --database-input Name=lab_iot
aws glue create-crawler --name lab-events-crawler --role lab-glue-role --database-name lab_iot \
  --targets "{\"S3Targets\":[{\"Path\":\"s3://$BUCKET/events/\"}]}"
aws glue start-crawler --name lab-events-crawler

# poll until READY (a few minutes)
aws glue get-crawler --name lab-events-crawler --query 'Crawler.[State,LastCrawl.Status]' --output text
aws glue get-table --database-name lab_iot --name events \
  --query 'Table.[StorageDescriptor.Columns[].Name,PartitionKeys[].Name]'
```

You should see the columns (`deviceid`, `ts`, `value`, `type`; Glue lowercases names) and the partition key `dt`.

### Step 10: Athena workgroup and queries on JSON

```bash
aws athena create-work-group --name lab-wg --configuration \
  "ResultConfiguration={OutputLocation=s3://$BUCKET/athena-results/},EnforceWorkGroupConfiguration=true,BytesScannedCutoffPerQuery=1073741824"

run_query () {
  QID=$(aws athena start-query-execution --work-group lab-wg \
        --query-string "$1" --query QueryExecutionId --output text)
  while true; do
    STATE=$(aws athena get-query-execution --query-execution-id $QID \
            --query QueryExecution.Status.State --output text)
    case $STATE in QUEUED|RUNNING) sleep 2;; *) break;; esac
  done
  aws athena get-query-execution --query-execution-id $QID \
    --query 'QueryExecution.[Status.State,Statistics.DataScannedInBytes,Statistics.EngineExecutionTimeInMillis]' \
    --output text
}

run_query "SELECT deviceid, avg(value) AS avg_value FROM lab_iot.events WHERE dt = '2026-01-02' GROUP BY deviceid ORDER BY 2 DESC LIMIT 5"
```

Record the scanned bytes (second column). Partition pruning (`WHERE dt = ...`) already limited the scan to one day. Show the benefit of partitions by running the same query without the `dt` filter and noting the larger number. You can read result rows with `aws athena get-query-results --query-execution-id <QID>`.

### Step 11: Convert to partitioned Parquet with CTAS and compare

The partition column must be last in the SELECT list, and the `external_location` prefix must be empty.

```bash
run_query "CREATE TABLE lab_iot.events_parquet WITH (format='PARQUET', parquet_compression='SNAPPY', external_location='s3://$BUCKET/curated/', partitioned_by=ARRAY['dt']) AS SELECT deviceid, ts, value, type, dt FROM lab_iot.events"

run_query "SELECT deviceid, avg(value) AS avg_value FROM lab_iot.events_parquet WHERE dt = '2026-01-02' GROUP BY deviceid ORDER BY 2 DESC LIMIT 5"

aws s3 ls s3://$BUCKET/curated/ --recursive --human-readable --summarize | tail -3
aws s3 ls s3://$BUCKET/events/ --recursive --human-readable --summarize | tail -3
```

Fill in this table from your results:

| Query (same SQL) | Format | Data scanned | Notes |
|---|---|---|---|
| `avg(value)` for `dt='2026-01-02'` | JSON | measured | reads whole rows |
| same | Parquet + Snappy, partitioned | measured | reads only `value`, `deviceid` columns; typically far smaller |
| same, no `dt` filter | JSON vs Parquet | measured | shows partition pruning gain |

Expected: Parquet scans much less because it is columnar (only needed columns are read), compressed, and carries statistics. Exact ratios depend on your data; Athena still bills at least 10 MB per query, so cost savings show at larger scale. For a continuous pipeline you would use Firehose record format conversion, a Glue ETL job, or scheduled CTAS/INSERT INTO.

### Step 12: What would you change for production

- Authenticate the API (IAM, JWT authorizer or Lambda authorizer), attach AWS WAF if you use a REST API, and use a custom domain with ACM.
- Use Firehose's built-in record format conversion to Parquet (needs a Glue table schema) and larger buffers (for example 64 to 128 MB) to avoid many tiny files; use dynamic partitioning if you need a device-based layout.
- Encrypt the bucket, stream and queue with KMS; add S3 lifecycle rules to move raw JSON to cheaper classes; set CloudWatch alarms on Lambda errors, Firehose delivery failures and the rejects queue depth.
- Set reserved or provisioned concurrency based on traffic and consider RDS Proxy-style protection for any downstream limits; use Lambda Powertools for structured logs; deploy with IaC and versioned aliases.
- Schedule crawlers or use partition projection to avoid crawling every run, and use Lake Formation for fine-grained access if many teams query.

## Validation

- `curl` with a valid payload returns 202, invalid payloads return 400 and appear in the `REJECTED:` logs, and nothing malformed is in `events/`.
- JSON files exist under `s3://<BUCKET>/events/dt=<date>/`, delivered by Firehose after the 60-second buffer.
- The Glue table `lab_iot.events` shows `dt` as a partition key; the Athena query succeeds and returns rows.
- The Parquet query scanned noticeably fewer bytes than the JSON query (record both numbers).
- The throttling test produced `429` responses; `simulate-principal-policy` returned `implicitDeny` for the reject role on Firehose.

## Cleanup

Delete in dependency order. Firehose, the API and the crawler are the things you do not want left around.

```bash
# 1. Front door and functions
aws apigatewayv2 delete-api --api-id $API_ID
UUID=$(aws lambda list-event-source-mappings --function-name lab-reject-logger \
  --query 'EventSourceMappings[0].UUID' --output text)
aws lambda delete-event-source-mapping --uuid $UUID
aws lambda delete-function --function-name lab-ingest
aws lambda delete-function --function-name lab-reject-logger
aws logs delete-log-group --log-group-name /aws/lambda/lab-ingest
aws logs delete-log-group --log-group-name /aws/lambda/lab-reject-logger

# 2. Firehose and queue
aws firehose delete-delivery-stream --delivery-stream-name lab-ingest
aws sqs delete-queue --queue-url $REJECT_URL

# 3. Glue and Athena (drop tables first; the crawler-created table and the CTAS table)
aws athena delete-work-group --work-group lab-wg --recursive-delete-option
aws glue delete-crawler --name lab-events-crawler
aws glue delete-table --database-name lab_iot --name events
aws glue delete-table --database-name lab_iot --name events_parquet
aws glue delete-database --name lab_iot

# 4. S3 (all prefixes: events, curated, athena-results, firehose-errors)
aws s3 rm s3://$BUCKET --recursive
aws s3api delete-bucket --bucket $BUCKET

# 5. IAM roles
for r in lab-ingest lab-reject; do
  aws iam detach-role-policy --role-name $r-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
done
aws iam delete-role-policy --role-name lab-ingest-role --policy-name ingest-access
aws iam delete-role-policy --role-name lab-reject-role --policy-name reject-queue-access
aws iam delete-role --role-name lab-ingest-role
aws iam delete-role --role-name lab-reject-role
aws iam delete-role-policy --role-name lab-firehose-role --policy-name s3-delivery
aws iam delete-role --role-name lab-firehose-role
aws iam detach-role-policy --role-name lab-glue-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
aws iam delete-role-policy --role-name lab-glue-role --policy-name read-events
aws iam delete-role --role-name lab-glue-role
```

If the Firehose delete is still in progress when you empty the bucket, wait for it to finish and re-run `aws s3 rm` so no late file remains. Deleting the Firehose stream first (as ordered above) avoids new writes. Finally, check that no Firehose stream, crawler or API remains.

## Exam Takeaway

- HTTP API plus Lambda is the lowest-ops, pay-per-request pattern. A Lambda needs an execution role to call services and a resource-based permission for the caller (API Gateway) to invoke it. (Domain 1: Secure Architectures; Domain 2: Resilient Architectures.)
- Synchronous invocations (API Gateway) do not use the Lambda DLQ or destinations; those are for asynchronous and stream/queue sources. Handle bad input in code and use SQS as the buffer or reject store. (Domain 2.)
- Throttling protects backends: API Gateway returns 429; clients should retry with exponential backoff and jitter. (Domain 2, Domain 3.)
- Firehose is the fully managed, near-real-time loader to S3, Redshift or OpenSearch with buffering; Kinesis Data Streams is for custom consumers and replay, and bills per shard in provisioned mode. (Domain 3: High-Performing Architectures.)
- Query in place on S3: Glue Data Catalog plus Athena. Cut scanned bytes with columnar formats (Parquet/ORC), compression and partitioning, and you cut cost, because Athena bills per data scanned. (Domain 3, Domain 4: Cost-Optimized Architectures.)
- Least privilege means one role per function with only the actions and ARNs it needs; verify with the policy simulator. (Domain 1.)
- Delete Firehose, crawlers and APIs when finished; watch for tiny-file proliferation and unbounded scans (workgroup scan limits). (Domain 4.)

## Stretch Goals

1. Enable Firehose record format conversion to Parquet (using the Glue table schema) so new data lands as Parquet directly, and compare file counts and bytes scanned.
2. Protect the API with a JWT authorizer (Amazon Cognito) or IAM authorization, and add a usage plan with an API key using a REST API for comparison.
3. Add an S3 event notification or EventBridge rule that triggers a Lambda to run a daily `INSERT INTO` CTAS compaction, and use Athena partition projection instead of the crawler.
4. Add an alarm on `lab-ingest` Lambda `Errors` and `Throttles` metrics, then set reserved concurrency to 1 and observe the throttling behavior from the caller's side.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
