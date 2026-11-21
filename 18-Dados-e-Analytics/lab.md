# Guided Lab: S3 Data Lake with Glue Crawler and Athena (CSV vs Partitioned Parquet)

## Objective

Land a small dataset in S3, catalog it with a Glue crawler, query it with Athena, then convert it to partitioned Parquet and compare the bytes scanned (which is what Athena bills for).

## Scenario

An analytics team receives daily sales CSV files. They want ad hoc SQL without managing servers. You build a minimal data lake: raw CSV, a catalog, and a cost-optimized Parquet copy partitioned by date.

## Services Used

- Amazon S3
- AWS Glue (Data Catalog, crawler)
- Amazon Athena
- AWS IAM
- Optional stretch: Amazon Kinesis Data Firehose

## Prerequisites

- AWS CLI v2 configured for `us-east-1`, and permissions for S3, Glue, Athena and IAM.
- Pick a globally unique bucket name.

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=<ACCOUNT_ID>
export BUCKET=<YOUR_BUCKET_NAME>
```

## Expected Cost

Under $1 total. Athena is $5 per TB scanned with a 10 MB minimum per query, so this dataset costs fractions of a cent. A Glue crawler is billed per DPU-hour with a 10-minute minimum per run (about $0.07 per run at $0.44 per DPU-hour with 2 DPUs). The Athena CTAS conversion also scans a tiny amount. S3 storage is negligible. Avoid: running crawlers on a schedule, and running the Firehose stretch step longer than a few minutes.

## Step by Step

### Step 1: Create the bucket and generate sample data

Console: S3 > Create bucket (`<YOUR_BUCKET_NAME>`, us-east-1, defaults keep Block Public Access on).

```bash
aws s3 mb s3://$BUCKET --region $AWS_REGION

# Generate ~3,000 rows of fake sales for 3 days
python3 - <<'EOF'
import csv, random
random.seed(1)
with open("sales.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["order_id","sale_date","product","qty","price"])
    for i in range(3000):
        d = f"2025-01-0{random.randint(1,3)}"
        w.writerow([i, d, random.choice(["book","pen","bag","lamp"]),
                    random.randint(1,5), round(random.uniform(2,90),2)])
EOF

aws s3 cp sales.csv s3://$BUCKET/raw/sales/sales.csv
```

### Step 2: Create a Glue IAM role

Console: IAM > Roles > Create role > AWS service > Glue; attach `AWSGlueServiceRole`; name it `glue-lab-role`. Then add an inline policy for the bucket.

```bash
cat > glue-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"glue.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name glue-lab-role \
  --assume-role-policy-document file://glue-trust.json
aws iam attach-role-policy --role-name glue-lab-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

cat > glue-s3.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Action":["s3:GetObject","s3:PutObject","s3:ListBucket"],
 "Resource":["arn:aws:s3:::$BUCKET","arn:aws:s3:::$BUCKET/*"]}]}
EOF
aws iam put-role-policy --role-name glue-lab-role \
  --policy-name lake-access --policy-document file://glue-s3.json
```

### Step 3: Create and run the crawler

Console: Glue > Crawlers > Create crawler. Data source: S3 path `s3://<YOUR_BUCKET_NAME>/raw/sales/`. Role `glue-lab-role`. Target database: create `lake_lab`. Schedule: On demand. Run it.

```bash
aws glue create-database --database-input Name=lake_lab

aws glue create-crawler --name sales-raw-crawler \
  --role glue-lab-role --database-name lake_lab \
  --targets "{\"S3Targets\":[{\"Path\":\"s3://$BUCKET/raw/sales/\"}]}"

aws glue start-crawler --name sales-raw-crawler
# poll until State is READY again (about 1-2 minutes)
aws glue get-crawler --name sales-raw-crawler --query Crawler.State
aws glue get-tables --database-name lake_lab --query 'TableList[].Name'
```

The crawler creates a table named `sales` (derived from the folder name).

### Step 4: Query the CSV with Athena

Console: Athena > Query editor > Settings > Manage > set the query result location to `s3://<YOUR_BUCKET_NAME>/athena-results/`. Select database `lake_lab`.

```bash
aws athena start-query-execution \
  --query-string "SELECT product, SUM(qty*price) AS revenue FROM lake_lab.sales WHERE sale_date='2025-01-02' GROUP BY product" \
  --result-configuration OutputLocation=s3://$BUCKET/athena-results/ \
  --query-execution-context Database=lake_lab
# note QueryExecutionId, then:
aws athena get-query-execution --query-execution-id <QUERY_ID> \
  --query 'QueryExecution.Statistics.DataScannedInBytes'
```

Write down the bytes scanned: Athena read the whole CSV even though you filtered on one day.

### Step 5: Convert to partitioned Parquet with CTAS

```sql
CREATE TABLE lake_lab.sales_parquet
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY',
  partitioned_by = ARRAY['sale_date'],
  external_location = 's3://<YOUR_BUCKET_NAME>/curated/sales_parquet/'
) AS
SELECT order_id, product, qty, price, sale_date
FROM lake_lab.sales;
```

Partition columns must be last in the SELECT list. Run it from the console or with `start-query-execution` as above. Then repeat the Step 4 query against `lake_lab.sales_parquet` and compare `DataScannedInBytes`. Only the `sale_date=2025-01-02` partition is read, and Parquet only reads the columns used. With this tiny file the 10 MB billing minimum hides the savings in dollars, but the bytes scanned will clearly drop; at terabyte scale this is the 10x-100x cost reduction.

### Step 6 (Optional stretch): Kinesis Data Firehose into the lake

Cost warning: Firehose bills per GB ingested (about $0.029/GB), so a few test records cost nothing, but do not leave a producer running. Prefer Firehose over Kinesis Data Streams when you only need delivery to S3, because Streams bills per shard-hour (about $0.015 per shard-hour, roughly $11 per month per provisioned shard) even when idle.

Console: Amazon Data Firehose > Create Firehose stream. Source: Direct PUT. Destination: Amazon S3 (`s3://<YOUR_BUCKET_NAME>/stream/`). Buffer interval 60 seconds. Let it create the IAM role.

```bash
aws firehose put-record --delivery-stream-name sales-stream \
  --record '{"Data":"MTAwMCwyMDI1LTAxLTAxLGJvb2ssMSw5LjUwCg=="}'
# wait ~60-120 seconds, then
aws s3 ls s3://$BUCKET/stream/ --recursive
```

### Step 7: Failure experiment

Remove `s3:GetObject` from the `lake-access` policy (`aws iam delete-role-policy --role-name glue-lab-role --policy-name lake-access`) and rerun the crawler. It ends in a failed or empty result with an access error in the crawler logs (CloudWatch log group `/aws-glue/crawlers`). Restore the policy afterwards if you want to rerun.

## Validation

- `aws glue get-tables --database-name lake_lab` lists `sales` and `sales_parquet`.
- The same aggregate query returns the same revenue numbers on both tables.
- `DataScannedInBytes` for the Parquet-partitioned query is much smaller than for the CSV query.

## Cleanup

```bash
aws firehose delete-delivery-stream --delivery-stream-name sales-stream   # if created
aws glue delete-crawler --name sales-raw-crawler
aws glue delete-table --database-name lake_lab --name sales
aws glue delete-table --database-name lake_lab --name sales_parquet
aws glue delete-database --name lake_lab
aws s3 rm s3://$BUCKET --recursive
aws s3 rb s3://$BUCKET
aws iam delete-role-policy --role-name glue-lab-role --policy-name lake-access
aws iam detach-role-policy --role-name glue-lab-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
aws iam delete-role --role-name glue-lab-role
```

If you created a Firehose stream through the console, also delete the IAM role it generated.

## Exam Takeaway

- Athena charges by data scanned, so the standard cost-reduction answer is columnar format (Parquet/ORC), compression and partitioning, not a bigger cluster.
- Glue Data Catalog is the shared metadata store for Athena, EMR and Redshift Spectrum; a crawler infers schema and partitions.
- Athena (serverless, ad hoc SQL on S3) versus Redshift (provisioned or serverless warehouse for heavy, repeated BI workloads).
- Firehose is the managed, near real-time delivery service (no consumers to write, optional Lambda transformation, S3/Redshift/OpenSearch targets); Kinesis Data Streams is for custom consumers, replay and sub-second latency.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
