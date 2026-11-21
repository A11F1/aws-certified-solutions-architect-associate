# Week 4 Main Lab: Product-Catalog Data Tier with Global Asset Delivery

## Objective

Build the data tier and edge delivery layer of a product-catalog backend in one sitting: a Multi-AZ RDS instance with a read replica for reporting, a DynamoDB on-demand table for sessions and carts (TTL plus a GSI), and a CloudFront distribution in front of a private S3 bucket (Origin Access Control) for product images. You will force an RDS failover, prove that the bucket cannot be read directly, and finish with a Route 53 routing-policy design and an RDS vs Aurora vs DynamoDB vs ElastiCache decision table.

Time: 90 to 120 minutes (most of it waiting for RDS and CloudFront to provision).

## Modules Covered

- [10 - Databases](../10-Banco-de-Dados/README.md)
- [11 - RDS and Relational Databases Labs](../11-RDS-e-Bancos-Relacionais-Labs/lab.md)
- [12 - DynamoDB](../12-DynamoDB/lab.md)
- [13 - DNS, Route 53 and CloudFront](../13-DNS-Route53-e-CloudFront/README.md)

## Scenario and Architecture

"ShelfCo" is launching a product catalog. Product and order data is relational and needs transactions, so it lives in RDS MySQL with Multi-AZ for availability. The analytics team wants heavy reporting queries that must not slow down the storefront, so they get a read replica. Shopping sessions and carts are key-value access patterns with huge spikes and automatic expiry, so they live in DynamoDB. Product images are static, so they sit in a private S3 bucket and are served worldwide through CloudFront. DNS (Route 53) is designed on paper, and run for real only if you own a domain.

```
                         Users (worldwide)
                               |
                  (Route 53: alias A/AAAA -> CloudFront)   <- design / optional
                               |
                     +---------v----------+
                     |  CloudFront (OAC)  |  edge cache, HTTPS
                     +---------+----------+
                               | SigV4-signed origin request
                     +---------v----------+
                     |  S3 bucket (private)|  product images
                     +--------------------+

   Storefront app (not built in this lab)
        |                                   |
        | SQL (transactions)                | GetItem / Query (single-digit ms)
        v                                   v
 +-----------------------------+     +--------------------------------+
 | RDS MySQL db.t3.micro       |     | DynamoDB  lab-cart (on-demand) |
 |  AZ-a PRIMARY  <== sync ==> |     |  PK userId, SK sk              |
 |  AZ-b STANDBY (Multi-AZ)    |     |  TTL: expiresAt                |
 |          |                  |     |  GSI: sku-index (sku, userId)  |
 |          | async replication|     +--------------------------------+
 |          v                  |
 |  READ REPLICA (reporting)   |
 +-----------------------------+
        (default VPC, private subnets/SG, no public access)
```

## Services Used

- Amazon RDS for MySQL (Multi-AZ, read replica, Secrets Manager managed password)
- Amazon DynamoDB (on-demand, TTL, GSI)
- Amazon S3 (private bucket) and Amazon CloudFront (Origin Access Control, cache policy, invalidation)
- Amazon Route 53 (design and CLI walkthrough; optional real hosted zone)
- AWS CLI v2 (CloudShell works well), optionally `jq`

## Prerequisites

- Personal AWS account, region `us-east-1`, an IAM identity with rights for RDS, EC2 (security groups, describe), DynamoDB, S3, CloudFront and Secrets Manager (an administrator sandbox user is simplest).
- A default VPC in `us-east-1` with subnets in at least two Availability Zones (the default VPC normally has this).
- No domain is required. If you own one, you can run Step 7 for real.

## Expected Cost

All figures are approximate; confirm them on the AWS pricing pages for RDS, DynamoDB, CloudFront, S3 and Route 53 before you start.

- RDS Multi-AZ `db.t3.micro` plus a read replica means three running instances' worth of compute and storage. Expect roughly a few cents per hour combined, so the lab is cents to low single-digit dollars if you finish and clean up the same day. The Free Tier typically covers a single-AZ micro instance, not Multi-AZ and not a replica; verify your account's current terms.
- DynamoDB on-demand: fractions of a cent for this lab. S3 and CloudFront: effectively free at this volume (CloudFront has a monthly free allowance; verify).
- Route 53: a hosted zone is billed monthly (roughly $0.50), so do not create one unless you will use it.
- Hourly-billed resources here: all three RDS instances. Delete them the same day (Cleanup). Do not leave the read replica running overnight: it bills like a full instance.

## Step by Step

### Step 1: Set variables and discover the network

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query 'Vpcs[0].VpcId' --output text)
export SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID \
  --query 'Subnets[].SubnetId' --output text)
echo $ACCOUNT_ID $VPC_ID $SUBNETS
```

Confirm at least two subnets in different AZs (`aws ec2 describe-subnets --subnet-ids <SUBNET_ID> --query 'Subnets[0].AvailabilityZone'`).

### Step 2: Create the RDS Multi-AZ instance

Console view: RDS > Create database > Standard create > MySQL > Templates "Dev/Test" > "Multi-AZ DB instance" > `db.t3.micro` > gp3 20 GiB > "Manage master credentials in AWS Secrets Manager" > Public access "No". The CLI equivalent:

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name lab-catalog-subnets \
  --db-subnet-group-description "Lab catalog subnets" \
  --subnet-ids $SUBNETS

export DB_SG=$(aws ec2 create-security-group --group-name lab-catalog-db \
  --description "Lab catalog DB (no inbound rules)" --vpc-id $VPC_ID \
  --query GroupId --output text)

aws rds create-db-instance \
  --db-instance-identifier lab-catalog \
  --engine mysql \
  --db-instance-class db.t3.micro \
  --allocated-storage 20 --storage-type gp3 \
  --master-username admin \
  --manage-master-user-password \
  --multi-az \
  --no-publicly-accessible \
  --backup-retention-period 1 \
  --db-subnet-group-name lab-catalog-subnets \
  --vpc-security-group-ids $DB_SG

aws rds wait db-instance-available --db-instance-identifier lab-catalog   # ~10-20 min
```

The password is generated and stored in Secrets Manager (no secret in your shell history). The security group has no inbound rules on purpose: this lab validates availability behavior through the control plane. A backup retention above 0 is required for a read replica. Optional stretch: add an inbound 3306 rule from a client security group and connect from a `t3.micro` EC2 instance.

Record the current placement:

```bash
aws rds describe-db-instances --db-instance-identifier lab-catalog \
  --query 'DBInstances[0].[DBInstanceStatus,AvailabilityZone,SecondaryAvailabilityZone,Endpoint.Address]' \
  --output text
```

### Step 3: Force a Multi-AZ failover (failure experiment 1)

Run this once the instance is `available`.

```bash
aws rds reboot-db-instance --db-instance-identifier lab-catalog --force-failover

# poll until available again (usually one to a few minutes)
aws rds wait db-instance-available --db-instance-identifier lab-catalog

aws rds describe-db-instances --db-instance-identifier lab-catalog \
  --query 'DBInstances[0].[AvailabilityZone,SecondaryAvailabilityZone,Endpoint.Address]' --output text

aws rds describe-events --source-type db-instance --source-identifier lab-catalog --duration 60 \
  --query 'Events[].[Date,Message]' --output text
```

Expected: `AvailabilityZone` and `SecondaryAvailabilityZone` swap, the endpoint hostname is unchanged (the DNS name now points to the new primary), and the events list shows a Multi-AZ failover started and completed. That is why applications must connect by endpoint name, never by IP, and must retry on dropped connections.

### Step 4: Add a read replica for reporting (keep it short-lived)

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier lab-catalog-reporting \
  --source-db-instance-identifier lab-catalog \
  --db-instance-class db.t3.micro \
  --no-publicly-accessible

aws rds wait db-instance-available --db-instance-identifier lab-catalog-reporting

aws rds describe-db-instances --db-instance-identifier lab-catalog-reporting \
  --query 'DBInstances[0].[ReadReplicaSourceDBInstanceIdentifier,Endpoint.Address]' --output text
```

Replication is asynchronous, so the replica can lag. CloudWatch metric `ReplicaLag` (namespace `AWS/RDS`, dimension `DBInstanceIdentifier=lab-catalog-reporting`) shows it. The replica has its own endpoint: reporting tools point there, the storefront keeps using the primary. Delete it as soon as you have captured the evidence (Cleanup).

### Step 5: Create the DynamoDB cart/session table with TTL and a GSI

Access patterns: (1) get a user's session and cart items, (2) find which carts hold a given SKU (for "low stock" alerts), (3) expire stale data automatically.

```bash
aws dynamodb create-table \
  --table-name lab-cart \
  --attribute-definitions AttributeName=userId,AttributeType=S \
                          AttributeName=sk,AttributeType=S \
                          AttributeName=sku,AttributeType=S \
  --key-schema AttributeName=userId,KeyType=HASH AttributeName=sk,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes \
    'IndexName=sku-index,KeySchema=[{AttributeName=sku,KeyType=HASH},{AttributeName=userId,KeyType=RANGE}],Projection={ProjectionType=KEYS_ONLY}'

aws dynamodb wait table-exists --table-name lab-cart

aws dynamodb update-time-to-live --table-name lab-cart \
  --time-to-live-specification Enabled=true,AttributeName=expiresAt
```

The same table in Terraform, for reference:

```hcl
resource "aws_dynamodb_table" "cart" {
  name         = "lab-cart"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "userId"
  range_key    = "sk"
  attribute { name = "userId" type = "S" }
  attribute { name = "sk"     type = "S" }
  attribute { name = "sku"    type = "S" }
  ttl { attribute_name = "expiresAt" enabled = true }
  global_secondary_index {
    name            = "sku-index"
    hash_key        = "sku"
    range_key       = "userId"
    projection_type = "KEYS_ONLY"
  }
}
```

Insert a session item (no `sku`, so it is not in the sparse GSI), two cart items, and one already-expired item:

```bash
NOW=$(date +%s); FUTURE=$((NOW+3600)); PAST=$((NOW-3600))

aws dynamodb put-item --table-name lab-cart --item \
  "{\"userId\":{\"S\":\"u-100\"},\"sk\":{\"S\":\"SESSION\"},\"expiresAt\":{\"N\":\"$FUTURE\"}}"
aws dynamodb put-item --table-name lab-cart --item \
  "{\"userId\":{\"S\":\"u-100\"},\"sk\":{\"S\":\"CART#SKU-1\"},\"sku\":{\"S\":\"SKU-1\"},\"qty\":{\"N\":\"2\"},\"expiresAt\":{\"N\":\"$FUTURE\"}}"
aws dynamodb put-item --table-name lab-cart --item \
  "{\"userId\":{\"S\":\"u-200\"},\"sk\":{\"S\":\"CART#SKU-1\"},\"sku\":{\"S\":\"SKU-1\"},\"qty\":{\"N\":\"1\"},\"expiresAt\":{\"N\":\"$FUTURE\"}}"
aws dynamodb put-item --table-name lab-cart --item \
  "{\"userId\":{\"S\":\"u-300\"},\"sk\":{\"S\":\"CART#SKU-9\"},\"sku\":{\"S\":\"SKU-9\"},\"qty\":{\"N\":\"5\"},\"expiresAt\":{\"N\":\"$PAST\"}}"

# Pattern 1: everything for one user (Query on the base table)
aws dynamodb query --table-name lab-cart \
  --key-condition-expression "userId = :u" \
  --expression-attribute-values '{":u":{"S":"u-100"}}'

# Pattern 2: who holds SKU-1 (Query on the GSI)
aws dynamodb query --table-name lab-cart --index-name sku-index \
  --key-condition-expression "sku = :s" \
  --expression-attribute-values '{":s":{"S":"SKU-1"}}'
```

TTL deletes in the background, typically within a couple of days of expiry, not at the exact second. So expired items can still be returned. Always filter on `expiresAt` in reads:

```bash
aws dynamodb query --table-name lab-cart \
  --key-condition-expression "userId = :u" \
  --filter-expression "expiresAt > :now" \
  --expression-attribute-values "{\":u\":{\"S\":\"u-300\"},\":now\":{\"N\":\"$NOW\"}}"
```

Failure experiment 2 (a design the table refuses): query by a non-key attribute on the base table.

```bash
aws dynamodb query --table-name lab-cart \
  --key-condition-expression "sku = :s" \
  --expression-attribute-values '{":s":{"S":"SKU-1"}}'
# Expect a ValidationException: the condition must use key attributes. Use the GSI, not a Scan.
```

### Step 6: Private S3 bucket behind CloudFront with Origin Access Control

```bash
export BUCKET=lab-catalog-assets-$ACCOUNT_ID
aws s3api create-bucket --bucket $BUCKET --region us-east-1
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

echo "<h1>ShelfCo catalog v1</h1>" > index.html
aws s3 cp index.html s3://$BUCKET/index.html --content-type text/html

export OAC_ID=$(aws cloudfront create-origin-access-control --origin-access-control-config \
  Name=lab-catalog-oac,SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3 \
  --query 'OriginAccessControl.Id' --output text)
```

Create `dist.json` (replace `<BUCKET>` and `<OAC_ID>`; the cache policy ID below is the AWS managed "CachingOptimized" policy; confirm with `aws cloudfront list-cache-policies --type managed`):

```json
{
  "CallerReference": "lab-catalog-1",
  "Comment": "lab catalog assets",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "PriceClass": "PriceClass_100",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "s3-assets",
      "DomainName": "<BUCKET>.s3.us-east-1.amazonaws.com",
      "OriginAccessControlId": "<OAC_ID>",
      "S3OriginConfig": { "OriginAccessIdentity": "" }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-assets",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }
}
```

```bash
sed -i.bak "s/<BUCKET>/$BUCKET/; s/<OAC_ID>/$OAC_ID/" dist.json
read DIST_ID DOMAIN <<< $(aws cloudfront create-distribution --distribution-config file://dist.json \
  --query 'Distribution.[Id,DomainName]' --output text)
echo $DIST_ID $DOMAIN
```

Now allow only that distribution to read the bucket (this is what makes OAC "private"):

```bash
cat > bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*",
    "Condition": { "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID" } }
  }]
}
EOF
aws s3api put-bucket-policy --bucket $BUCKET --policy file://bucket-policy.json
aws cloudfront wait distribution-deployed --id $DIST_ID     # several minutes
```

Failure experiment 3 (deny direct access) and caching behavior:

```bash
curl -sI https://$DOMAIN/index.html | egrep -i 'HTTP/|x-cache'     # 1st: Miss from cloudfront
curl -sI https://$DOMAIN/index.html | egrep -i 'HTTP/|x-cache'     # 2nd: Hit from cloudfront
curl -sI https://$BUCKET.s3.us-east-1.amazonaws.com/index.html | head -1   # expect 403

# Update the object: CloudFront keeps serving the cached v1 until TTL expiry or invalidation
echo "<h1>ShelfCo catalog v2</h1>" > index.html
aws s3 cp index.html s3://$BUCKET/index.html --content-type text/html
curl -s https://$DOMAIN/index.html            # still v1
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/index.html"
# after it completes (aws cloudfront get-invalidation ...), curl again: v2
```

In production you would prefer versioned object names (`app.v2.css`) over invalidations.

### Step 7: Route 53 routing-policy design (design and CLI walkthrough)

Without a domain, treat this as a design exercise. Pick the policy for each requirement:

| Requirement | Routing policy | Notes |
|---|---|---|
| Send `www.shelfco.example` to CloudFront | Simple + Alias (A/AAAA) | Alias to a CloudFront distribution is free of query charges and works at the zone apex |
| Roll out a new API version to 10% of traffic | Weighted | Weights 90/10, shift gradually; weight 0 drains a record |
| Active/passive: primary API in us-east-1, static "maintenance" site as backup | Failover | Needs a health check on the primary; secondary serves when it is unhealthy |
| EU users to eu-west-1 stack, US users to us-east-1 | Geolocation | Based on user location; add a default record |
| Pick the AWS Region with the lowest network latency for each user | Latency | Records tagged by Region; not the same as geography |
| Gradually move traffic toward a Region regardless of user location | Geoproximity | Uses bias; configured with Traffic Flow |
| Return several healthy IPs, let the client choose | Multivalue answer | Not a load balancer; up to 8 healthy records |
| Control which IP ranges or sources get which answer | IP-based | Based on client CIDR |

Optional real walkthrough (only if you own a domain, and remember a hosted zone costs money until deleted):

```bash
aws route53 create-hosted-zone --name <YOUR_DOMAIN> --caller-reference lab-$(date +%s)
# delegate the NS records at your registrar, then:
cat > alias.json <<EOF
{ "Changes": [{ "Action": "UPSERT", "ResourceRecordSet": {
    "Name": "assets.<YOUR_DOMAIN>", "Type": "A",
    "AliasTarget": { "HostedZoneId": "Z2FDTNDATAQYW2",
                     "DNSName": "$DOMAIN", "EvaluateTargetHealth": false } } }] }
EOF
aws route53 change-resource-record-sets --hosted-zone-id <HOSTED_ZONE_ID> --change-batch file://alias.json
```

`Z2FDTNDATAQYW2` is the fixed hosted zone ID used for CloudFront alias targets. A custom domain on CloudFront additionally needs an ACM certificate issued in `us-east-1` and the name added as an alternate domain name (CNAME) on the distribution; that is a stretch goal.

### Step 8: Decide which database for which workload

Fill this in against the catalog and defend each choice:

| Need | RDS (MySQL/PostgreSQL) | Aurora | DynamoDB | ElastiCache (Redis/Valkey, Memcached) |
|---|---|---|---|---|
| Data model | Relational, joins, transactions | Relational (MySQL/PostgreSQL compatible) | Key-value/document, access-pattern driven | In-memory key-value, data structures |
| Scale shape | Vertical scale-up plus read replicas | Cluster storage auto-grows; up to 15 replicas, fast failover | Virtually unlimited horizontal, on-demand or provisioned | Memory-bound, shard and replicate |
| Availability | Multi-AZ standby (not readable) | 6 copies across 3 AZs, readable replicas | Multi-AZ by default; Global Tables for multi-Region | Replicas and Multi-AZ with auto-failover (Redis/Valkey) |
| Latency | Milliseconds | Milliseconds | Single-digit ms (DAX for microseconds) | Sub-millisecond |
| Pick it when | Existing SQL app, lift and shift, simple HA | SQL with higher performance/availability, Aurora Serverless v2 for variable load | Sessions, carts, IoT, huge spiky key-based traffic | Caching, leaderboards, session cache in front of a DB |
| Catalog use | Products, orders, inventory (this lab) | Upgrade path if RDS read scaling or failover time hurts | Carts and sessions (this lab) | Cache hot product pages and queries (not built) |

### Step 9: What would you change for production

- Put the application and DB in private subnets with security-group-to-security-group rules, and use RDS Proxy for connection pooling (especially with Lambda).
- Enable encryption at rest with a customer managed KMS key, longer backup retention, deletion protection, and Performance Insights.
- Consider Aurora if you need faster failover and up to 15 readers; use Multi-AZ DB cluster or Aurora for readable standbys.
- Add DynamoDB point-in-time recovery, and a Global Table if you need multi-Region active-active.
- Add an ACM certificate and custom domain, AWS WAF on CloudFront, response headers policy, and an origin access log bucket. Use versioned asset names instead of invalidations.
- Add ElastiCache or DAX in front of hot reads.

## Validation

- `describe-db-instances` for `lab-catalog` shows `MultiAZ: true`, and the AZ pair swapped after the forced failover while the endpoint name stayed identical.
- `lab-catalog-reporting` is `available` with `ReadReplicaSourceDBInstanceIdentifier` equal to `lab-catalog`.
- The GSI query for `SKU-1` returns two items; the base-table query on `sku` fails with a `ValidationException`.
- `aws dynamodb describe-time-to-live --table-name lab-cart` shows `TimeToLiveStatus` ENABLED (or ENABLING) on `expiresAt`.
- First CloudFront request shows `X-Cache: Miss from cloudfront`, the second shows `Hit`, and the direct S3 URL returns 403.
- After invalidation, the page serves v2.

## Cleanup

Do this the same day. The order matters because of dependencies (replica before source, distribution disabled before delete, bucket emptied before delete).

```bash
# 1. RDS: replica first, then the primary
aws rds delete-db-instance --db-instance-identifier lab-catalog-reporting --skip-final-snapshot
aws rds wait db-instance-deleted --db-instance-identifier lab-catalog-reporting
aws rds delete-db-instance --db-instance-identifier lab-catalog \
  --skip-final-snapshot --delete-automated-backups
aws rds wait db-instance-deleted --db-instance-identifier lab-catalog
aws rds delete-db-subnet-group --db-subnet-group-name lab-catalog-subnets
aws ec2 delete-security-group --group-id $DB_SG

# 2. DynamoDB
aws dynamodb delete-table --table-name lab-cart

# 3. CloudFront: disable, wait, delete, then OAC
ETAG=$(aws cloudfront get-distribution-config --id $DIST_ID --query ETag --output text)
aws cloudfront get-distribution-config --id $DIST_ID --query DistributionConfig \
  | jq '.Enabled=false' > dist-disabled.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file://dist-disabled.json
aws cloudfront wait distribution-deployed --id $DIST_ID
ETAG=$(aws cloudfront get-distribution --id $DIST_ID --query ETag --output text)
aws cloudfront delete-distribution --id $DIST_ID --if-match $ETAG
OAC_ETAG=$(aws cloudfront get-origin-access-control --id $OAC_ID --query ETag --output text)
aws cloudfront delete-origin-access-control --id $OAC_ID --if-match $OAC_ETAG

# 4. S3
aws s3 rm s3://$BUCKET --recursive
aws s3api delete-bucket --bucket $BUCKET

# 5. Route 53 (only if you created a zone): delete your records, then
# aws route53 delete-hosted-zone --id <HOSTED_ZONE_ID>
```

Check that the master-user Secrets Manager secret is gone (RDS-managed secrets are removed with the instance) with `aws secretsmanager list-secrets`. Then verify in the console that no RDS instance, snapshot or CloudFront distribution remains.

## Exam Takeaway

- Multi-AZ RDS is for high availability (synchronous standby, automatic failover via DNS, not for read scaling). Read replicas are for read scaling and can be promoted or cross-Region, using asynchronous replication. Know both and never confuse them. (Domain 2: Design Resilient Architectures; Domain 3: High-Performing Architectures.)
- Pick the database by access pattern: relational transactions are RDS/Aurora, key-based spiky workloads are DynamoDB, microsecond or caching needs are ElastiCache or DAX. (Domain 3.)
- DynamoDB: Query needs the key (or a GSI), Scan reads the whole table, TTL deletion is eventual and free of write cost, on-demand suits unpredictable traffic. (Domain 3, Domain 4 for cost.)
- CloudFront with OAC plus a bucket policy keeps S3 private. OAC replaces the legacy OAI. Caching lowers origin load and latency; invalidations clear stale objects. (Domain 1: Secure Architectures; Domain 3.)
- Route 53 policy matching (failover, weighted, latency, geolocation, geoproximity, multivalue) is a classic question; alias records are free to query for AWS resources and work at the apex. (Domain 2, Domain 3.)
- Unneeded read replicas and idle Multi-AZ instances are cost traps. Delete short-lived resources. (Domain 4: Cost-Optimized Architectures.)

## Stretch Goals

1. Connect to RDS from a `t3.micro` EC2 instance (or an RDS Proxy), create a `products` table, and watch `ReplicaLag` while loading rows. Promote the replica and observe that it becomes an independent instance.
2. Add a CloudFront custom domain with an ACM certificate in `us-east-1` and a Route 53 alias, then add a failover record pair with a health check.
3. Enable DynamoDB Streams on `lab-cart` and trigger a Lambda that writes "low stock" alerts when many carts hold the same SKU.
4. Add a CloudFront cache behavior with a different cache policy for `/api/*`, and compare `Age` headers.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
