# Guided Lab: Secure, Versioned S3 Bucket with Lifecycle Rules and Default Encryption

## Objective

Build a private S3 bucket and practice the fundamentals the exam keeps testing: versioning and delete markers, storage classes, lifecycle transitions and expiration, Block Public Access, a bucket policy that enforces TLS, and default encryption with SSE-S3 versus SSE-KMS.

## Scenario

A small team stores monthly reports in S3. Reports must be recoverable after accidental deletes, move to cheaper storage as they age, never be publicly readable, and always be encrypted. You will build this in about 40 minutes and prove each control works.

## Services Used

- Amazon S3 (versioning, lifecycle, bucket policy, Block Public Access, default encryption)
- AWS KMS (optional, to compare SSE-KMS with SSE-S3)
- AWS CLI v2

## Prerequisites

- A personal AWS account and an IAM user or role with S3 (and optionally KMS) permissions. Do not use the root user.
- AWS CLI v2 configured: `aws configure` with region `us-east-1`.
- A globally unique bucket name. Throughout this lab, `<YOUR_BUCKET_NAME>` is a placeholder such as `saa-lab06-<ACCOUNT_ID>`.

## Expected Cost

Effectively free. You will store a few KB. S3 Standard is billed per GB-month and per request, so this lab costs fractions of a cent. A customer managed KMS key costs about $1 per month (prorated, and billed even if scheduled for deletion until the deletion date passes), so use the AWS managed key `aws/s3` unless you want to test a customer managed key, and then delete it right after. Avoid leaving the bucket with many versions after the lab. Note that S3 Glacier classes have minimum storage durations and minimum object sizes, so do not transition tiny files to them in a real workload.

## Step by Step

### Step 1: Create the bucket with Block Public Access on

Console: S3 > Create bucket > name `<YOUR_BUCKET_NAME>`, region US East (N. Virginia), keep "Block all public access" checked, ACLs disabled (Bucket owner enforced), Bucket Versioning: Enable, Default encryption: SSE-S3. Create bucket.

CLI:

```bash
export BUCKET=<YOUR_BUCKET_NAME>
aws s3api create-bucket --bucket $BUCKET --region us-east-1

aws s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-versioning --bucket $BUCKET \
  --versioning-configuration Status=Enabled
```

### Step 2: Set default encryption

New buckets already default to SSE-S3, but set it explicitly so you can see the configuration.

Console: bucket > Properties > Default encryption > Edit > Amazon S3 managed keys (SSE-S3).

```bash
aws s3api put-bucket-encryption --bucket $BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

Optional comparison with SSE-KMS using the AWS managed key:

```bash
aws s3api put-bucket-encryption --bucket $BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms", "KMSMasterKeyID": "alias/aws/s3"},
      "BucketKeyEnabled": true
    }]
  }'
```

Bucket Keys reduce the number of KMS requests (and cost) for SSE-KMS. Switch back to AES256 afterwards if you want the cheapest setup.

### Step 3: Upload, overwrite, and delete to see versioning

```bash
echo "report v1" > report.txt
aws s3 cp report.txt s3://$BUCKET/reports/report.txt
echo "report v2" > report.txt
aws s3 cp report.txt s3://$BUCKET/reports/report.txt

aws s3api list-object-versions --bucket $BUCKET --prefix reports/
```

You should see two versions with different `VersionId` values. Now delete the object without a version id:

```bash
aws s3 rm s3://$BUCKET/reports/report.txt
aws s3api list-object-versions --bucket $BUCKET --prefix reports/
```

The output now shows a `DeleteMarkers` entry and both versions still exist. In the console, toggle "List versions" to see the same. Recover by deleting the delete marker:

```bash
aws s3api delete-object --bucket $BUCKET --key reports/report.txt \
  --version-id <DELETE_MARKER_VERSION_ID>
aws s3 cp s3://$BUCKET/reports/report.txt -
```

### Step 4: Upload with a specific storage class

```bash
aws s3 cp report.txt s3://$BUCKET/archive/report-ia.txt --storage-class STANDARD_IA
aws s3api head-object --bucket $BUCKET --key archive/report-ia.txt
```

`StorageClass` appears in the response. Note that STANDARD_IA bills a minimum of 128 KB per object and a 30-day minimum duration, so it is a poor fit for tiny or short-lived objects.

### Step 5: Add a lifecycle configuration

Transition `archive/` to Standard-IA after 30 days and Glacier Flexible Retrieval after 90 days, expire current versions after 365 days, remove noncurrent versions after 30 days, and abort stale multipart uploads. (S3 requires at least 30 days in Standard before a transition to Standard-IA.)

Console: bucket > Management > Create lifecycle rule.

```bash
cat > lifecycle.json <<'JSON'
{
  "Rules": [
    {
      "ID": "archive-tiering",
      "Status": "Enabled",
      "Filter": {"Prefix": "archive/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 365},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
JSON
aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET \
  --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET
```

### Step 6: Bucket policy that denies non-TLS requests

```bash
cat > policy.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyInsecureTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::<YOUR_BUCKET_NAME>",
      "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
    ],
    "Condition": {"Bool": {"aws:SecureTransport": "false"}}
  }]
}
JSON
aws s3api put-bucket-policy --bucket $BUCKET --policy file://policy.json
```

### Step 7: Security experiment, try to make it public

Attempt to attach a policy that grants public read:

```bash
cat > public.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicRead", "Effect": "Allow", "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
  }]
}
JSON
aws s3api put-bucket-policy --bucket $BUCKET --policy file://public.json
```

Expected: an `AccessDenied` error mentioning that the public policy is blocked by Block Public Access (`BlockPublicPolicy`). Then try an anonymous request:

```bash
curl -I https://$BUCKET.s3.us-east-1.amazonaws.com/reports/report.txt
```

Expected: `HTTP/1.1 403 Forbidden`. Re-apply the TLS policy from Step 6 afterwards, since a rejected put leaves the previous policy in place.

## Validation

- `list-object-versions` shows multiple versions plus a delete marker after Step 3, and the object is readable again after you remove the marker.
- `head-object` on the encrypted object returns `ServerSideEncryption` of `AES256` (or `aws:kms`).
- `get-bucket-lifecycle-configuration` returns your rule.
- `get-public-access-block` shows all four settings true.
- The public policy is rejected and the anonymous `curl` returns 403.

## Cleanup

A versioned bucket cannot be deleted until all versions and delete markers are gone.

```bash
aws s3api delete-bucket-policy --bucket $BUCKET
aws s3api list-object-versions --bucket $BUCKET \
  --query '{Objects: [Versions,DeleteMarkers][][].{Key:Key,VersionId:VersionId}}' --output json > del.json
aws s3api delete-objects --bucket $BUCKET --delete file://del.json
aws s3api delete-bucket --bucket $BUCKET
```

If the list is large, the console "Empty bucket" action then "Delete bucket" is simpler. If you created a customer managed KMS key, schedule its deletion (minimum 7 days): `aws kms schedule-key-deletion --key-id <KEY_ID> --pending-window-in-days 7`.

## Exam Takeaway

Versioning protects against accidental deletes and overwrites (a simple delete only adds a delete marker), while lifecycle rules control cost by moving data to cheaper classes and expiring noncurrent versions. Choose the storage class by access pattern: Standard for frequent access, Standard-IA or One Zone-IA for infrequent but millisecond access, Glacier classes for archives, and Intelligent-Tiering when patterns are unknown. For security, Block Public Access overrides public bucket policies and ACLs, a `aws:SecureTransport` deny enforces encryption in transit, and SSE-S3 is the zero-effort default while SSE-KMS adds key-level audit and access control at the cost of KMS requests.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
