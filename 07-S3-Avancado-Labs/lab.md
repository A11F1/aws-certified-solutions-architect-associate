# Guided Lab: Cross-Region Replication, Presigned URLs, Event Notifications and a Private CloudFront Website

## Objective

Practice four advanced S3 patterns in one small project: cross-region replication (CRR), time-limited presigned URLs, S3 event notifications to SQS, and serving a static site privately through CloudFront with Origin Access Control (OAC).

## Scenario

A marketing team publishes a static site and a downloadable PDF. The site must be served over HTTPS from a private bucket, the PDF must be shared with a partner by a link that expires, uploads must trigger downstream processing, and the site content must be copied to a second region for disaster recovery.

## Services Used

- Amazon S3 (replication, presigned URLs, event notifications)
- Amazon CloudFront (OAC)
- Amazon SQS
- AWS IAM
- AWS CLI v2

## Prerequisites

- Personal AWS account, admin-level IAM user or role (not root), AWS CLI v2, region `us-east-1`.
- Two bucket names: `<SOURCE_BUCKET>` in us-east-1 and `<DEST_BUCKET>` in us-west-2. Versioning is required on both for replication.
- Basic familiarity with the S3 lab in module 06.

## Expected Cost

Low, usually under $1 if you clean up the same day. CloudFront has a free tier of 1 TB of data transfer and 10 million requests per month, but a distribution that sticks around is still worth deleting. CRR charges for inter-region data transfer and replication PUTs, which is trivial for a few KB. SQS has a free tier of 1 million requests per month. Avoid enabling Replication Time Control (S3 RTC), which carries an extra per-GB fee, and avoid leaving CloudFront distributions or replicated copies behind.

## Step by Step

### Step 1: Create versioned source and destination buckets

Console: S3 > Create bucket, twice, enabling Bucket Versioning on both and keeping Block Public Access on.

```bash
export SRC=<SOURCE_BUCKET>
export DST=<DEST_BUCKET>
aws s3api create-bucket --bucket $SRC --region us-east-1
aws s3api create-bucket --bucket $DST --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2
aws s3api put-bucket-versioning --bucket $SRC --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket $DST --versioning-configuration Status=Enabled
```

### Step 2: Configure cross-region replication

Console: source bucket > Management > Replication rules > Create replication rule > apply to all objects > destination `<DEST_BUCKET>` > create a new IAM role.

CLI: create a role S3 can assume, then attach permissions.

```bash
cat > trust.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"s3.amazonaws.com"},"Action":"sts:AssumeRole"}]}
JSON
aws iam create-role --role-name s3-crr-lab-role --assume-role-policy-document file://trust.json

cat > perms.json <<'JSON'
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":["s3:GetReplicationConfiguration","s3:ListBucket"],
  "Resource":"arn:aws:s3:::<SOURCE_BUCKET>"},
 {"Effect":"Allow","Action":["s3:GetObjectVersionForReplication","s3:GetObjectVersionAcl","s3:GetObjectVersionTagging"],
  "Resource":"arn:aws:s3:::<SOURCE_BUCKET>/*"},
 {"Effect":"Allow","Action":["s3:ReplicateObject","s3:ReplicateDelete","s3:ReplicateTags"],
  "Resource":"arn:aws:s3:::<DEST_BUCKET>/*"}]}
JSON
aws iam put-role-policy --role-name s3-crr-lab-role --policy-name crr --policy-document file://perms.json

cat > repl.json <<'JSON'
{
  "Role": "arn:aws:iam::<ACCOUNT_ID>:role/s3-crr-lab-role",
  "Rules": [{
    "ID": "all-to-dr",
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "DeleteMarkerReplication": {"Status": "Disabled"},
    "Destination": {"Bucket": "arn:aws:s3:::<DEST_BUCKET>", "StorageClass": "STANDARD_IA"}
  }]
}
JSON
aws s3api put-bucket-replication --bucket $SRC --replication-configuration file://repl.json
```

Test it. Replication applies only to objects written after the rule exists:

```bash
echo "<h1>Hello CRR</h1>" > index.html
aws s3 cp index.html s3://$SRC/index.html
sleep 60
aws s3api head-object --bucket $DST --key index.html --region us-west-2
aws s3api head-object --bucket $SRC --key index.html   # ReplicationStatus: COMPLETED
```

The replica's `ReplicationStatus` is `REPLICA`. Replication is asynchronous, usually seconds to minutes.

### Step 3: Share a private PDF with a presigned URL

```bash
echo "partner brochure" > brochure.pdf
aws s3 cp brochure.pdf s3://$SRC/downloads/brochure.pdf
aws s3 presign s3://$SRC/downloads/brochure.pdf --expires-in 120
```

Open the URL in a browser: the file downloads. Wait 2 minutes and retry: you get `AccessDenied` with `Request has expired`. Try the plain object URL without a signature and you get 403. A presigned URL carries the permissions of the identity that signed it, so if that identity loses access, the URL stops working. Presigned URLs made with temporary role credentials also expire when the session does.

### Step 4: Event notification to SQS

```bash
aws sqs create-queue --queue-name s3-lab-events --region us-east-1
aws sqs get-queue-url --queue-name s3-lab-events
```

Allow S3 to send messages (replace placeholders with the queue ARN):

```bash
cat > q-policy.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"s3.amazonaws.com"},"Action":"sqs:SendMessage",
 "Resource":"arn:aws:sqs:us-east-1:<ACCOUNT_ID>:s3-lab-events",
 "Condition":{"ArnLike":{"aws:SourceArn":"arn:aws:s3:::<SOURCE_BUCKET>"},
              "StringEquals":{"aws:SourceAccount":"<ACCOUNT_ID>"}}}]}
JSON
python3 - <<'PY'
import json
p=open('q-policy.json').read()
json.dump({"Policy":p},open('q-attr.json','w'))
PY
aws sqs set-queue-attributes --queue-url <QUEUE_URL> --attributes file://q-attr.json

cat > notif.json <<'JSON'
{"QueueConfigurations":[{"QueueArn":"arn:aws:sqs:us-east-1:<ACCOUNT_ID>:s3-lab-events",
 "Events":["s3:ObjectCreated:*"],"Filter":{"Key":{"FilterRules":[{"Name":"prefix","Value":"uploads/"}]}}}]}
JSON
aws s3api put-bucket-notification-configuration --bucket $SRC --notification-configuration file://notif.json
```

Console equivalent: bucket > Properties > Event notifications > Create event notification. Trigger it and read the message:

```bash
aws s3 cp index.html s3://$SRC/uploads/test.html
aws sqs receive-message --queue-url <QUEUE_URL> --wait-time-seconds 10
```

Uploading outside `uploads/` produces no message.

### Step 5: Serve the site privately through CloudFront with OAC

Console (simplest path): CloudFront > Create distribution > Origin domain: choose `<SOURCE_BUCKET>` (the S3 REST endpoint, not the website endpoint) > Origin access: Origin access control settings > Create new OAC (sign requests, S3) > Default root object `index.html` > Viewer protocol policy: Redirect HTTP to HTTPS > Create. Copy the bucket policy that CloudFront offers and apply it to the bucket.

The resulting bucket policy has this shape:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<SOURCE_BUCKET>/*",
    "Condition": {"StringEquals": {"AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"}}
  }]
}
```

```bash
aws s3api put-bucket-policy --bucket $SRC --policy file://oac-policy.json
aws cloudfront get-distribution --id <DISTRIBUTION_ID> --query 'Distribution.Status'
```

Wait until the status is `Deployed` (several minutes), then:

```bash
curl -I https://<DISTRIBUTION_DOMAIN>.cloudfront.net/            # 200, served from index.html
curl -I https://$SRC.s3.us-east-1.amazonaws.com/index.html       # 403, bucket stays private
```

Note: if the bucket is replicated and you put the OAC policy on it, remember the destination bucket is a separate bucket with its own policy.

## Validation

- `index.html` appears in the destination bucket in us-west-2 with `ReplicationStatus: REPLICA`.
- The presigned URL works inside its window and fails after expiry.
- An upload under `uploads/` produces an S3 event message in SQS and nothing for other prefixes.
- The CloudFront URL returns 200 while direct S3 access returns 403.

## Cleanup

```bash
aws cloudfront get-distribution-config --id <DISTRIBUTION_ID> > dist.json   # note the ETag; set Enabled=false in the config and update
aws cloudfront update-distribution --id <DISTRIBUTION_ID> --if-match <ETAG> --distribution-config file://dist-config-disabled.json
# after status is Deployed:
aws cloudfront delete-distribution --id <DISTRIBUTION_ID> --if-match <NEW_ETAG>
aws cloudfront delete-origin-access-control --id <OAC_ID> --if-match <OAC_ETAG>
aws s3api delete-bucket-replication --bucket $SRC
aws sqs delete-queue --queue-url <QUEUE_URL>
aws iam delete-role-policy --role-name s3-crr-lab-role --policy-name crr
aws iam delete-role --role-name s3-crr-lab-role
```

Then empty both buckets, including all versions and delete markers (console "Empty bucket" is easiest), and delete them. In the console you can also disable and delete the distribution from the CloudFront page.

## Exam Takeaway

Replication needs versioning on both buckets, is asynchronous, and only copies new objects (use S3 Batch Replication for existing ones). CRR serves disaster recovery and latency needs, while SRR serves log aggregation or compliance copies. Use presigned URLs for temporary access to private objects instead of making buckets public, and OAC (the successor to OAI) with CloudFront when you want HTTPS and edge caching in front of a private bucket. S3 website endpoints are HTTP only and require a public bucket, so the exam answer for "secure static site with HTTPS" is CloudFront plus OAC. Event notifications decouple uploads from processing via SQS, SNS, Lambda, or EventBridge.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
