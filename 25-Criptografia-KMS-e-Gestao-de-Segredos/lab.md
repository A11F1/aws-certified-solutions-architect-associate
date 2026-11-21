# Guided Lab: KMS Customer Managed Key, Envelope Encryption, SSE-KMS and Secrets Manager

## Objective

Create a customer managed KMS key, prove that the key policy controls access, encrypt and decrypt with the CLI, perform envelope encryption with `generate-data-key`, enforce SSE-KMS on an S3 bucket, and compare Secrets Manager with SSM Parameter Store SecureString.

## Scenario

An application stores customer files in S3 and a database password. Security requires keys the company controls, an audit trail, and no plaintext secrets in code.

## Services Used

- AWS KMS (customer managed key, alias, key policy)
- Amazon S3 (default encryption SSE-KMS)
- AWS Secrets Manager
- AWS Systems Manager Parameter Store
- AWS IAM, AWS CloudTrail (optional, to see KMS calls)

## Prerequisites

- AWS CLI v2, Region `us-east-1`, an admin IAM user or role.
- `base64` and `openssl` available in your shell.
- Optional: a second IAM user `kms-test` with only `s3:*` and no KMS permissions, for the denial test (create a throwaway access key and delete it afterwards; never commit keys).

## Expected Cost

A customer managed key costs 1 USD per month (prorated hourly) plus 0.03 USD per 10,000 requests. A Secrets Manager secret is 0.40 USD per month (prorated) and the first 30 days are free for new secrets. Standard Parameter Store parameters are free. S3 usage is pennies. Scheduling key deletion stops the monthly charge once the key is in the pending-deletion state. Avoid keeping keys or secrets "just in case".

## Step by Step

### Step 1: Create the customer managed key

Console: KMS > Customer managed keys > Create key > Symmetric, Encrypt and decrypt > alias `lab-key` > key administrators: your user > key users: your user.

```bash
KEY_ID=$(aws kms create-key --description "lab key" --query KeyMetadata.KeyId --output text)
aws kms create-alias --alias-name alias/lab-key --target-key-id $KEY_ID
aws kms describe-key --key-id alias/lab-key
```

### Step 2: Understand key policy versus IAM

Every KMS key must have a key policy; IAM policies work only if the key policy allows it, typically through the root statement that delegates to IAM.

```bash
aws kms get-key-policy --key-id $KEY_ID --policy-name default --query Policy --output text
```

The default policy has `"Principal": {"AWS": "arn:aws:iam::<ACCOUNT_ID>:root"}` with `"Action": "kms:*"`. That statement is what lets IAM policies grant access. If you delete it (do not do this on the lab key without another admin), IAM alone can no longer grant access.

### Step 3: Encrypt and decrypt with the CLI

```bash
echo -n "card-4111-demo" > plain.txt
aws kms encrypt --key-id alias/lab-key --plaintext fileb://plain.txt \
  --query CiphertextBlob --output text | base64 --decode > secret.enc
aws kms decrypt --ciphertext-blob fileb://secret.enc \
  --query Plaintext --output text | base64 --decode
```

`decrypt` needs no key ID for symmetric keys because it is stored in the ciphertext. Direct `encrypt` is limited to 4 KB of data, which is why envelope encryption exists.

### Step 4: Envelope encryption with generate-data-key

```bash
aws kms generate-data-key --key-id alias/lab-key --key-spec AES_256 --output json > dk.json
jq -r .Plaintext dk.json | base64 --decode > datakey.bin
jq -r .CiphertextBlob dk.json | base64 --decode > datakey.enc
head -c 1000000 /dev/urandom > bigfile.bin
openssl enc -aes-256-cbc -pbkdf2 -in bigfile.bin -out bigfile.enc -pass file:datakey.bin
shred -u datakey.bin 2>/dev/null || rm -f datakey.bin   # discard plaintext data key
# later: recover the data key and decrypt
aws kms decrypt --ciphertext-blob fileb://datakey.enc --query Plaintext --output text | base64 --decode > datakey.bin
openssl enc -d -aes-256-cbc -pbkdf2 -in bigfile.enc -out bigfile.out -pass file:datakey.bin
cmp bigfile.bin bigfile.out && echo "round trip OK"
```

You store `bigfile.enc` plus the encrypted data key; KMS never sees the large data.

### Step 5: S3 default encryption with SSE-KMS

Console: S3 > Create bucket > Default encryption > SSE-KMS > choose `lab-key`; enable Bucket Key to cut KMS request costs.

```bash
aws s3api create-bucket --bucket <YOUR_BUCKET_NAME> --region us-east-1
aws s3api put-bucket-encryption --bucket <YOUR_BUCKET_NAME> --server-side-encryption-configuration '{
  "Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"alias/lab-key"},"BucketKeyEnabled":true}]}'
aws s3 cp plain.txt s3://<YOUR_BUCKET_NAME>/plain.txt
aws s3api head-object --bucket <YOUR_BUCKET_NAME> --key plain.txt --query "[ServerSideEncryption,SSEKMSKeyId]"
```

### Step 6: Denial experiment

Disable the key and try to read the object; then re-enable it.

```bash
aws kms disable-key --key-id $KEY_ID
aws s3 cp s3://<YOUR_BUCKET_NAME>/plain.txt -      # fails: KMS.DisabledException
aws kms enable-key --key-id $KEY_ID
```

Optional with `kms-test`: it has `s3:*` but no KMS rights, so `aws s3 cp s3://<YOUR_BUCKET_NAME>/plain.txt - --profile kms-test` returns AccessDenied because the key policy does not allow it. Then add that user as a key user in the key policy (or add `kms:Decrypt` via IAM, which works only while the root statement exists) and retry.

### Step 7: Secrets Manager versus Parameter Store

```bash
aws secretsmanager create-secret --name lab/db-password --secret-string '{"username":"app","password":"<PLACEHOLDER_NOT_REAL>"}' --kms-key-id alias/lab-key
aws secretsmanager get-secret-value --secret-id lab/db-password --query SecretString --output text

aws ssm put-parameter --name /lab/db-password --type SecureString --key-id alias/lab-key --value '<PLACEHOLDER_NOT_REAL>'
aws ssm get-parameter --name /lab/db-password --with-decryption --query Parameter.Value --output text
```

Rotation concept (read, do not enable without a database): console Secrets Manager > secret > Rotation > Edit rotation. Secrets Manager runs a Lambda function on a schedule (for example every 30 days) that creates a new password, updates the database and updates the secret's version stages (`AWSCURRENT`/`AWSPREVIOUS`). Native templates exist for RDS, Aurora, Redshift and DocumentDB.

| Feature | Secrets Manager | Parameter Store SecureString |
|---|---|---|
| Built-in rotation | Yes | No (DIY with Lambda) |
| Cost | Per secret per month plus API calls | Standard tier free |
| Cross-account sharing | Resource policy | Advanced tier with RAM |
| Size limit | 64 KB | 4 KB (standard), 8 KB (advanced) |
| Best for | DB credentials, rotation | Config values and simple secrets |

## Validation

- `describe-key` shows the key `Enabled`, and `decrypt` returns the original text.
- `round trip OK` is printed for envelope encryption.
- `head-object` shows `ServerSideEncryption: aws:kms` and your key ARN.
- Reads fail while the key is disabled and work again after enabling it.
- Both the secret and the parameter return the value only with permission.
- Optional: CloudTrail Event history shows `Decrypt` and `GenerateDataKey` events from `kms.amazonaws.com`.

## Cleanup

```bash
aws secretsmanager delete-secret --secret-id lab/db-password --force-delete-without-recovery
aws ssm delete-parameter --name /lab/db-password
aws s3 rm s3://<YOUR_BUCKET_NAME> --recursive
aws s3api delete-bucket --bucket <YOUR_BUCKET_NAME>
aws kms delete-alias --alias-name alias/lab-key
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
rm -f plain.txt secret.enc dk.json datakey.bin datakey.enc bigfile.* 
```

KMS keys cannot be deleted immediately: the waiting period is 7 to 30 days (default 30). During it the key is unusable and can be recovered with `aws kms cancel-key-deletion`. Disable the key instead if you are unsure; deletion makes all data encrypted under it permanently unrecoverable. Delete the `kms-test` user and its access key if you created them.

## Exam Takeaway

KMS access needs the key policy to allow it (directly or via the root statement that enables IAM policies); IAM alone is not enough. Use envelope encryption for data larger than 4 KB: KMS protects the data key, and the data key encrypts the data. Choose customer managed keys when you need control, rotation settings and audit; choose Secrets Manager when automatic rotation is required and Parameter Store for cheap configuration values. Key deletion is delayed 7 to 30 days as a safety net.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
