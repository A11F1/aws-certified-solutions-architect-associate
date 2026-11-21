# Guided Lab: Moving Data with DataSync (S3 to S3) and a DMS Walkthrough

## Objective

Use AWS DataSync to copy data between two S3 buckets, observe incremental behavior and task results, then walk through (and optionally run) an AWS DMS migration from a small RDS source to an S3 target. Finish with a comparison of Snow Family, Application Migration Service and the other migration tools.

## Scenario

A company is consolidating data into a new bucket and moving a small MySQL database toward the cloud data lake. There is no on-premises source for this lab, so a second S3 bucket and a small RDS database stand in for the "legacy" sides.

## Services Used

- AWS DataSync
- Amazon S3
- AWS Database Migration Service (DMS)
- Amazon RDS for MySQL (optional source)
- AWS IAM

## Prerequisites

- AWS CLI v2 configured for `us-east-1`.
- Two bucket names.

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=<ACCOUNT_ID>
export SRC=<YOUR_BUCKET_NAME>-source
export DST=<YOUR_BUCKET_NAME>-dest
```

## Expected Cost

- Part A (DataSync): about $0.0125 per GB copied; with a few MB of data it is a fraction of a cent. No agent is needed for S3-to-S3 inside AWS.
- Part B (DMS, optional hands-on): a dms.t3.micro replication instance is about $0.02 per hour, plus a db.t3.micro RDS MySQL (Free Tier eligible for 12 months on older accounts, otherwise about $0.017 per hour) and storage. Keep it under 1-2 hours. If you only read Part B, cost is $0.
- Avoid: leaving the replication instance or RDS running, and RDS Multi-AZ.

## Step by Step

### Part A: DataSync between two S3 buckets

### Step 1: Create buckets and sample data

Console: S3 > Create bucket twice (`<YOUR_BUCKET_NAME>-source`, `<YOUR_BUCKET_NAME>-dest`) in us-east-1.

```bash
aws s3 mb s3://$SRC --region $AWS_REGION
aws s3 mb s3://$DST --region $AWS_REGION
for i in 1 2 3 4 5; do echo "file $i" > file$i.txt; done
aws s3 cp . s3://$SRC/data/ --recursive --exclude "*" --include "file*.txt"
```

### Step 2: Create the IAM role DataSync uses to access S3

Console: DataSync > Locations > Create location > Amazon S3 can create the role for you with "Autogenerate". The CLI version needs an explicit role.

```bash
cat > ds-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"datasync.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name datasync-s3-lab \
  --assume-role-policy-document file://ds-trust.json

cat > ds-policy.json <<EOF
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":["s3:GetBucketLocation","s3:ListBucket","s3:ListBucketMultipartUploads"],
  "Resource":["arn:aws:s3:::$SRC","arn:aws:s3:::$DST"]},
 {"Effect":"Allow","Action":["s3:AbortMultipartUpload","s3:DeleteObject","s3:GetObject","s3:ListMultipartUploadParts","s3:PutObject","s3:GetObjectTagging","s3:PutObjectTagging"],
  "Resource":["arn:aws:s3:::$SRC/*","arn:aws:s3:::$DST/*"]}]}
EOF
aws iam put-role-policy --role-name datasync-s3-lab \
  --policy-name ds-s3 --policy-document file://ds-policy.json
```

### Step 3: Create locations and the task

Console: DataSync > Tasks > Create task. Source: Amazon S3 (`$SRC`, prefix `data/`); Destination: Amazon S3 (`$DST`, prefix `copy/`); Settings: Transfer only data that has changed, Verify only the data transferred.

```bash
ROLE_ARN=arn:aws:iam::$ACCOUNT_ID:role/datasync-s3-lab
SRC_LOC=$(aws datasync create-location-s3 --s3-bucket-arn arn:aws:s3:::$SRC \
  --subdirectory /data --s3-config BucketAccessRoleArn=$ROLE_ARN \
  --query LocationArn --output text)
DST_LOC=$(aws datasync create-location-s3 --s3-bucket-arn arn:aws:s3:::$DST \
  --subdirectory /copy --s3-config BucketAccessRoleArn=$ROLE_ARN \
  --query LocationArn --output text)

TASK_ARN=$(aws datasync create-task --name s3-to-s3-lab \
  --source-location-arn $SRC_LOC --destination-location-arn $DST_LOC \
  --options TransferMode=CHANGED,VerifyMode=ONLY_FILES_TRANSFERRED \
  --query TaskArn --output text)
```

### Step 4: Run and observe

```bash
EXEC_ARN=$(aws datasync start-task-execution --task-arn $TASK_ARN \
  --query TaskExecutionArn --output text)
aws datasync describe-task-execution --task-execution-arn $EXEC_ARN \
  --query '[Status,FilesTransferred,BytesTransferred]'
aws s3 ls s3://$DST/copy/
```

Wait for `Status` to become `SUCCESS` (usually a minute or two). Then add one new file to the source and run the task again. Because of `TransferMode=CHANGED`, only the new file is transferred, which you can confirm in `FilesTransferred`.

```bash
echo "file 6" > file6.txt && aws s3 cp file6.txt s3://$SRC/data/
aws datasync start-task-execution --task-arn $TASK_ARN
```

### Step 5: Failure experiment

Remove the destination permissions (`aws iam delete-role-policy --role-name datasync-s3-lab --policy-name ds-s3`), add another source file and rerun the task. The execution ends in `ERROR` with an access denied result. Re-add the policy to continue. This shows that DataSync uses the role you give it, not your own credentials.

### Part B: DMS replication (walkthrough, optional to run)

### Step 6: Understand the pieces

DMS needs four objects: a replication instance (the compute that moves data), a source endpoint, a target endpoint and a replication task (full load, CDC, or full load plus CDC). For S3 as a target, DMS writes CSV or Parquet files and needs an IAM role it can assume to write to the bucket.

### Step 7: Prepare a small RDS MySQL source

Console: RDS > Create database > Standard > MySQL > Free tier template, `db.t3.micro`, public access Yes only if you will connect from your own machine (restrict the security group to your IP). Create a parameter group with `binlog_format=ROW` (required for CDC) and `binlog_retention_hours` set through `CALL mysql.rds_set_configuration('binlog retention hours', 24);`. Load a tiny table:

```sql
CREATE DATABASE shop;
CREATE TABLE shop.orders (id INT PRIMARY KEY, item VARCHAR(50), qty INT);
INSERT INTO shop.orders VALUES (1,'book',2),(2,'pen',10);
```

### Step 8: Create the replication instance, endpoints and task

Console: DMS > Replication instances > Create (dms.t3.micro, single-AZ, default VPC, Publicly accessible No if RDS is in the same VPC). Endpoints > Create source (RDS MySQL) and target (Amazon S3, bucket `$DST`, folder `dms`, IAM role with `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket`, data format Parquet). Run Test connection on each. Database migration tasks > Create task > Migrate existing data and replicate ongoing changes > table mapping selection rule for schema `shop`.

```bash
aws dms create-replication-instance \
  --replication-instance-identifier dms-lab \
  --replication-instance-class dms.t3.micro \
  --allocated-storage 20 --no-multi-az --no-publicly-accessible

aws dms create-endpoint --endpoint-identifier src-mysql --endpoint-type source \
  --engine-name mysql --server-name <RDS_ENDPOINT> --port 3306 \
  --username <DB_USER> --password <DB_PASSWORD>
aws dms create-endpoint --endpoint-identifier tgt-s3 --endpoint-type target \
  --engine-name s3 \
  --s3-settings ServiceAccessRoleArn=arn:aws:iam::<ACCOUNT_ID>:role/<DMS_S3_ROLE>,BucketName=$DST,BucketFolder=dms,DataFormat=parquet

cat > mapping.json <<'EOF'
{"rules":[{"rule-type":"selection","rule-id":"1","rule-name":"1",
 "object-locator":{"schema-name":"shop","table-name":"%"},"rule-action":"include"}]}
EOF
aws dms create-replication-task --replication-task-identifier mysql-to-s3 \
  --source-endpoint-arn <SRC_ENDPOINT_ARN> --target-endpoint-arn <TGT_ENDPOINT_ARN> \
  --replication-instance-arn <REPL_INSTANCE_ARN> \
  --migration-type full-load-and-cdc \
  --table-mappings file://mapping.json
aws dms start-replication-task --replication-task-arn <TASK_ARN> \
  --start-replication-task-type start-replication
```

Insert another row in `shop.orders` and watch a new CDC file land in `s3://<YOUR_BUCKET_NAME>-dest/dms/shop/orders/`. Never put real passwords in commands; use Secrets Manager for real work. For a heterogeneous migration (for example Oracle to Aurora PostgreSQL) you would first run the AWS Schema Conversion Tool to convert schema and code, because DMS moves data, not complex schema objects.

### Step 9: Migration tool comparison (conceptual)

| Need | Tool | Key point |
|---|---|---|
| Online file transfer from NFS/SMB/HDFS/S3/EFS/FSx, scheduled, verified | DataSync | Agent on premises; managed, up to 10x faster than open source tools |
| Relational DB migration with minimal downtime | DMS (+ SCT for heterogeneous) | Full load plus CDC |
| Lift-and-shift of servers (VMs/physical) into EC2 | Application Migration Service (MGN) | Continuous block-level replication, cutover with minimal downtime; replaces CloudEndure/SMS |
| Tens of TB to PB with poor or no bandwidth | Snow Family (Snowcone, Snowball Edge) | Ship a device; Snowmobile is for exabyte-scale (retired for new customers by AWS in 2024, but still appears in older material) |
| Ongoing hybrid access to cloud storage | Storage Gateway | File, volume and tape gateways with local cache |
| Simple, internet-based SFTP/FTPS/FTP into S3 or EFS | AWS Transfer Family | Managed protocol endpoints |

Rule of thumb: if transfer time over your link exceeds roughly a week, evaluate Snow Family; otherwise DataSync or S3 transfer acceleration.

## Validation

- `aws s3 ls s3://$DST/copy/` shows all source files.
- The first task execution transfers all files; the second transfers only the new one.
- A task execution with removed permissions fails with an access error.
- If you ran Part B: Parquet files with the `orders` rows appear under `dms/shop/orders/`, and a later insert shows up as a CDC file.

## Cleanup

Delete in dependency order (DMS first, since the task depends on endpoints and instance).

```bash
# DMS (if created)
aws dms stop-replication-task --replication-task-arn <TASK_ARN>
aws dms delete-replication-task --replication-task-arn <TASK_ARN>
aws dms delete-endpoint --endpoint-arn <SRC_ENDPOINT_ARN>
aws dms delete-endpoint --endpoint-arn <TGT_ENDPOINT_ARN>
aws dms delete-replication-instance --replication-instance-arn <REPL_INSTANCE_ARN>
aws rds delete-db-instance --db-instance-identifier <DB_ID> \
  --skip-final-snapshot --delete-automated-backups

# DataSync
aws datasync delete-task --task-arn $TASK_ARN
aws datasync delete-location --location-arn $SRC_LOC
aws datasync delete-location --location-arn $DST_LOC

# IAM and S3
aws iam delete-role-policy --role-name datasync-s3-lab --policy-name ds-s3
aws iam delete-role --role-name datasync-s3-lab
aws s3 rm s3://$SRC --recursive && aws s3 rb s3://$SRC
aws s3 rm s3://$DST --recursive && aws s3 rb s3://$DST
```

Also delete the DMS S3 role, the custom RDS parameter group and any security group rules you opened.

## Exam Takeaway

- Match the tool to the data: files go through DataSync (online) or Snow Family (offline, limited bandwidth); databases go through DMS (with SCT for different engines); whole servers go through Application Migration Service.
- DMS keeps the source online during migration via change data capture, so it is the answer for "minimal downtime database migration". The replication instance is billed hourly and must be removed afterwards.
- DataSync handles scheduling, incremental transfer, integrity verification and bandwidth throttling, which scripts such as `aws s3 sync` do not give you at scale.
- Think about timing: compute transfer time from data size and bandwidth before choosing between network transfer and a Snow device.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
