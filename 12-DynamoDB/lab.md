# Guided Lab: DynamoDB Table Design with GSI, TTL, Streams and Capacity Modes

## Objective

Design a DynamoDB table around its access patterns, then compare Query vs Scan cost, add a Global Secondary Index, enable TTL and Streams, and observe the difference between on-demand and provisioned capacity.

## Scenario

An e-commerce app stores orders. It must (1) list all orders for a customer sorted by date, (2) list all orders in a given status such as `SHIPPED`, and (3) automatically expire abandoned carts. You model this in one table named `lab-orders`.

## Services Used

- Amazon DynamoDB (table, GSI, TTL, Streams)
- AWS CLI v2 (or CloudShell)

## Prerequisites

- AWS account, region `us-east-1`, permissions for `dynamodb:*` and `dynamodbstreams:*`.
- AWS CLI v2 configured.

## Expected Cost

Effectively free. On-demand charges only per request (fractions of a cent for this lab), and the Free Tier includes 25 GB of storage, plus 25 provisioned RCU/WCU. Avoid: leaving large provisioned capacity on tables and forgetting Point-in-Time Recovery or backups, which are billed separately.

## Step by Step

### Step 1: Create the table (on-demand) with a GSI and Streams

Access pattern 1 drives the primary key: partition key `customerId`, sort key `orderDate` (ISO-8601 string so it sorts chronologically). Pattern 2 needs a GSI keyed on `status` with `orderDate` as sort key.

Console: DynamoDB > Tables > Create table > name `lab-orders`, partition key `customerId` (String), sort key `orderDate` (String), Customize settings > Capacity mode On-demand > Secondary indexes > Create global index: PK `status` (String), SK `orderDate` (String), name `status-orderDate-index` > Create.

```bash
aws dynamodb create-table \
  --table-name lab-orders \
  --attribute-definitions \
      AttributeName=customerId,AttributeType=S \
      AttributeName=orderDate,AttributeType=S \
      AttributeName=status,AttributeType=S \
  --key-schema AttributeName=customerId,KeyType=HASH AttributeName=orderDate,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[{
      "IndexName":"status-orderDate-index",
      "KeySchema":[{"AttributeName":"status","KeyType":"HASH"},{"AttributeName":"orderDate","KeyType":"RANGE"}],
      "Projection":{"ProjectionType":"ALL"}}]' \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

aws dynamodb wait table-exists --table-name lab-orders
```

### Step 2: Insert items

```bash
put() { aws dynamodb put-item --table-name lab-orders --item "$1"; }
put '{"customerId":{"S":"C1"},"orderDate":{"S":"2026-01-05T10:00:00Z"},"status":{"S":"SHIPPED"},"total":{"N":"40"}}'
put '{"customerId":{"S":"C1"},"orderDate":{"S":"2026-02-11T09:30:00Z"},"status":{"S":"PENDING"},"total":{"N":"15"}}'
put '{"customerId":{"S":"C1"},"orderDate":{"S":"2026-03-02T17:45:00Z"},"status":{"S":"SHIPPED"},"total":{"N":"99"}}'
put '{"customerId":{"S":"C2"},"orderDate":{"S":"2026-01-20T08:00:00Z"},"status":{"S":"SHIPPED"},"total":{"N":"12"}}'
put '{"customerId":{"S":"C3"},"orderDate":{"S":"2026-02-28T12:00:00Z"},"status":{"S":"CANCELLED"},"total":{"N":"70"}}'
```

### Step 3: Query vs Scan, and the GSI

```bash
# Query by partition key and sort-key range: reads only C1's items
aws dynamodb query --table-name lab-orders \
  --key-condition-expression "customerId = :c AND orderDate >= :d" \
  --expression-attribute-values '{":c":{"S":"C1"},":d":{"S":"2026-02-01"}}' \
  --return-consumed-capacity TOTAL

# Scan reads EVERY item, then filters afterwards
aws dynamodb scan --table-name lab-orders \
  --filter-expression "#s = :s" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":s":{"S":"SHIPPED"}}' \
  --return-consumed-capacity TOTAL

# Same question answered efficiently through the GSI
aws dynamodb query --table-name lab-orders --index-name status-orderDate-index \
  --key-condition-expression "#s = :s" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":s":{"S":"SHIPPED"}}' \
  --return-consumed-capacity TOTAL
```

Compare `ScannedCount` and `ConsumedCapacity`. The scan's `ScannedCount` equals the whole table while the query's only covers matching keys. With 5 items the difference is small; with millions it is the difference between pennies and a bill.

### Step 4: Enable TTL for expiring items

TTL uses a numeric attribute holding an epoch time in seconds.

```bash
aws dynamodb update-time-to-live --table-name lab-orders \
  --time-to-live-specification Enabled=true,AttributeName=expiresAt

EXP=$(( $(date +%s) - 60 ))   # already expired
aws dynamodb put-item --table-name lab-orders --item \
  "{\"customerId\":{\"S\":\"C9\"},\"orderDate\":{\"S\":\"2026-03-10T00:00:00Z\"},\"status\":{\"S\":\"CART\"},\"expiresAt\":{\"N\":\"$EXP\"}}"
```

TTL deletion is a background process and is typically completed within a few days (it is not instant), and expired-but-not-yet-deleted items can still be returned. Use a filter such as `expiresAt > :now` if your app must hide them. TTL deletes are free and do show up in the stream.

### Step 5: Read the stream

```bash
STREAM_ARN=$(aws dynamodb describe-table --table-name lab-orders \
  --query 'Table.LatestStreamArn' --output text)
SHARD=$(aws dynamodbstreams describe-stream --stream-arn $STREAM_ARN \
  --query 'StreamDescription.Shards[0].ShardId' --output text)
IT=$(aws dynamodbstreams get-shard-iterator --stream-arn $STREAM_ARN \
  --shard-id $SHARD --shard-iterator-type TRIM_HORIZON --query ShardIterator --output text)
aws dynamodbstreams get-records --shard-iterator $IT

# Generate a modify event and read again (OldImage + NewImage appear)
aws dynamodb update-item --table-name lab-orders \
  --key '{"customerId":{"S":"C1"},"orderDate":{"S":"2026-02-11T09:30:00Z"}}' \
  --update-expression "SET #s = :s" --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":s":{"S":"SHIPPED"}}'
```

In production a Lambda function would consume this stream (event source mapping) for downstream actions.

### Step 6: Compare provisioned capacity and throttling

Create a second, tiny provisioned table (no GSI, to keep it simple):

```bash
aws dynamodb create-table --table-name lab-orders-prov \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1
aws dynamodb wait table-exists --table-name lab-orders-prov

for i in $(seq 1 200); do
  aws dynamodb put-item --table-name lab-orders-prov \
    --item "{\"id\":{\"S\":\"k$i\"},\"pad\":{\"S\":\"$(head -c 900 /dev/zero | tr '\0' x)\"}}" \
    --cli-read-timeout 5 2>&1 | grep -i Throughput
done
```

A 1 KB write consumes 1 WCU, so at 1 WCU/sec (plus a short burst reserve) you should eventually see `ProvisionedThroughputExceededException`. Note that the CLI retries automatically, so errors may appear only after retries are exhausted. The on-demand table never throttles at this volume. You can switch billing mode with `aws dynamodb update-table --billing-mode PAY_PER_REQUEST` (allowed once per 24 hours per table).

## Validation

- The `query` on `customerId = C1` returns three orders in ascending date order.
- The GSI query returns all `SHIPPED` orders across customers; the base-table scan consumed more capacity for the same answer.
- `describe-table` shows `TimeToLiveStatus: ENABLED` via `aws dynamodb describe-time-to-live --table-name lab-orders`.
- `get-records` returns `INSERT` and `MODIFY` records with `NewImage`/`OldImage`.
- The provisioned table throttles under the burst; the on-demand one does not.

## Cleanup

```bash
aws dynamodb delete-table --table-name lab-orders-prov
aws dynamodb delete-table --table-name lab-orders   # also removes its GSI and stream
aws dynamodb list-tables
```

## Exam Takeaway

In DynamoDB you design for access patterns: Query on a key is cheap and fast, Scan is a full-table read, and a GSI gives you an alternate key for another pattern (with its own capacity and eventual consistency). Choose on-demand for unpredictable or spiky traffic with no capacity planning, and provisioned (with auto scaling or reserved capacity) for steady, predictable load at lower unit cost. TTL removes stale data for free, and Streams are the answer to "react to item changes" (Lambda triggers, replication, audit).

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
