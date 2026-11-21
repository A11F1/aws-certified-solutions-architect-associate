# Week 3 Main Lab: Private Network, Private Bucket, and CloudFront Static Site

## Objective
Build a custom VPC with public and private subnets. A private EC2 instance (no NAT, no public IP) reads and writes a private S3 bucket through an S3 gateway endpoint, and the bucket policy accepts requests only from that endpoint. The bucket has versioning and a lifecycle rule. Separately, a static website lives in a second bucket and is served only through CloudFront using Origin Access Control (OAC). You then prove what is blocked: internet from the private instance, direct S3 access from outside the VPC, and direct S3 access to the website bucket.

Estimated time: 90 to 120 minutes (CloudFront deployment adds waiting time).

## Modules Covered
- [06 - Amazon S3 and Storage](../06-Amazon-S3-e-Armazenamento/README.md)
- [07 - Advanced S3 Labs](../07-S3-Avancado-Labs/lab.md)
- [08 - VPC and Networking](../08-VPC-e-Redes/README.md)
- [09 - VPC and Networking Labs](../09-VPC-e-Redes-Labs/lab.md)

## Scenario and Architecture
A company processes sensitive files that must never traverse the public internet and must be reachable only from inside its network. The same company also publishes a public marketing page that must be fast, HTTPS-only, and must not expose its origin bucket.

```
 Public viewers                          Operator (CLI, outside VPC)
      |                                         |
      v HTTPS                                   | aws s3 ls  -> AccessDenied
 +------------+  OAC (SigV4)  +-----------+     |   (no aws:SourceVpce)
 | CloudFront |-------------->| S3: site  |     |
 +------------+               | (private) |     |
                              +-----------+     |
                                                v
 VPC 10.20.0.0/16 (us-east-1)            +-------------------------+
 +----------------------------------+    | S3: data bucket         |
 | Public subnet 10.20.1.0/24       |    |  versioning + lifecycle |
 |  route 0.0.0.0/0 -> IGW          |    |  policy: Deny unless    |
 |  (nothing deployed; no NAT)      |    |  aws:SourceVpce = vpce  |
 +----------------------------------+    +-----------^-------------+
 | Private subnet 10.20.2.0/24      |                |
 |  +----------------------------+  |   +------------+-----------+
 |  | EC2 t3.micro (no public IP)|  |   | S3 Gateway Endpoint    |
 |  | role: S3 + SSM             |--+-->| route-table prefix list|
 |  +----------------------------+  |   +------------------------+
 |  Interface endpoints (SSM x3) ---+--> Systems Manager (no internet)
 |  Route table: local + S3 prefix list ONLY (no 0.0.0.0/0)         |
 +------------------------------------------------------------------+
```

## Services Used
- Amazon VPC (subnets, route tables, Internet Gateway, security groups, gateway endpoint, interface endpoints)
- Amazon S3 (versioning, lifecycle, bucket policies, Block Public Access)
- Amazon CloudFront (distribution, Origin Access Control)
- Amazon EC2 and IAM instance profile
- AWS Systems Manager (Run Command, reached through interface endpoints)

## Prerequisites
- AWS CLI v2, region `us-east-1`, admin-capable identity (as in Week 1), `jq`.
- Completed Week 2 concepts (security groups, instance profiles) is helpful.
- A machine outside the VPC to run the "outside" tests (your laptop or CloudShell).

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SUFFIX=$(date +%s)
export DATA_BUCKET=lab-w3-private-data-${ACCOUNT_ID}-${SUFFIX}
export SITE_BUCKET=lab-w3-static-site-${ACCOUNT_ID}-${SUFFIX}
```

## Expected Cost
Approximate; confirm on the VPC, S3, CloudFront, and EC2 pricing pages. Resources that bill by the hour: the t3.micro instance (about 0.01 USD/hour) and the three SSM interface endpoints (each roughly 0.01 USD per hour per AZ plus data processing; around 0.03 USD/hour in total in one AZ). The S3 gateway endpoint has no hourly charge. No NAT Gateway is created on purpose (it would add an hourly charge plus per-GB processing). S3 and CloudFront usage for a few small files is typically cents or free-tier covered, but CloudFront free tier terms depend on your plan; check the pricing page. Plan to finish and run Cleanup the same day; delete the interface endpoints and instance as soon as you finish the private-network tests.

## Step by Step

### Step 1: Create the VPC, subnets, and routing
```bash
export VPC_ID=$(aws ec2 create-vpc --cidr-block 10.20.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-w3-vpc}]' \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'

export PUB_SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.20.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-w3-public}]' \
  --query Subnet.SubnetId --output text)
export PRIV_SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.20.2.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-w3-private}]' \
  --query Subnet.SubnetId --output text)

export IGW_ID=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

export PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_SUBNET

export PRIV_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 associate-route-table --route-table-id $PRIV_RT --subnet-id $PRIV_SUBNET
```
The difference between "public" and "private" is only the route table: the public subnet has a route to an Internet Gateway, the private one does not. The private subnet does not auto-assign public IPs. The public subnet holds nothing in this lab; it exists so you can compare the two route tables and see that no NAT is needed.

### Step 2: Security groups
```bash
export EC2_SG=$(aws ec2 create-security-group --group-name lab-w3-ec2-sg \
  --description "Private instance, no inbound" --vpc-id $VPC_ID --query GroupId --output text)

export VPCE_SG=$(aws ec2 create-security-group --group-name lab-w3-vpce-sg \
  --description "Interface endpoints" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $VPCE_SG \
  --protocol tcp --port 443 --source-group $EC2_SG
```
The instance group has no inbound rules. Default outbound is allow-all, which does not matter here because there is no route to the internet.

### Step 3: Create the S3 gateway endpoint
```bash
export S3_VPCE=$(aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --vpc-endpoint-type Gateway --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $PRIV_RT \
  --query VpcEndpoint.VpcEndpointId --output text)
echo $S3_VPCE

aws ec2 describe-route-tables --route-table-ids $PRIV_RT \
  --query 'RouteTables[0].Routes' --output table
```
The route table now has a route whose destination is the S3 prefix list (`pl-...`) with target `vpce-...`. There is still no `0.0.0.0/0` route. Console: **VPC > Endpoints**.

### Step 4: Create the private data bucket with versioning and lifecycle
```bash
aws s3api create-bucket --bucket $DATA_BUCKET --region us-east-1
aws s3api put-public-access-block --bucket $DATA_BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-encryption --bucket $DATA_BUCKET \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws s3api put-bucket-versioning --bucket $DATA_BUCKET \
  --versioning-configuration Status=Enabled

cat > lifecycle.json <<'JSON'
{
  "Rules": [{
    "ID": "tier-and-expire",
    "Status": "Enabled",
    "Filter": {"Prefix": ""},
    "Transitions": [{"Days": 30, "StorageClass": "STANDARD_IA"}],
    "NoncurrentVersionExpiration": {"NoncurrentDays": 30},
    "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
  }]
}
JSON
aws s3api put-bucket-lifecycle-configuration --bucket $DATA_BUCKET \
  --lifecycle-configuration file://lifecycle.json
```
Do this configuration before the restrictive bucket policy so your own CLI is not blocked. Notes: 30 days is the minimum before moving objects to Standard-IA, and S3 by default does not transition objects smaller than 128 KB to IA classes. Lifecycle runs asynchronously, so you will not see transitions during this lab; you are verifying the configuration is accepted.

### Step 5: Create the instance role and launch the private instance
```bash
cat > ec2-trust.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
JSON
aws iam create-role --role-name lab-w3-ec2-role --assume-role-policy-document file://ec2-trust.json
aws iam attach-role-policy --role-name lab-w3-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

cat > ec2-s3.json <<JSON
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":["s3:GetObject","s3:PutObject"],
  "Resource":"arn:aws:s3:::${DATA_BUCKET}/*"},
 {"Effect":"Allow","Action":"s3:ListBucket","Resource":"arn:aws:s3:::${DATA_BUCKET}"}]}
JSON
aws iam put-role-policy --role-name lab-w3-ec2-role --policy-name data-bucket-rw \
  --policy-document file://ec2-s3.json

aws iam create-instance-profile --instance-profile-name lab-w3-ec2-profile
aws iam add-role-to-instance-profile --instance-profile-name lab-w3-ec2-profile \
  --role-name lab-w3-ec2-role
sleep 15

AMI=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)

export INSTANCE_ID=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --subnet-id $PRIV_SUBNET --security-group-ids $EC2_SG \
  --no-associate-public-ip-address \
  --iam-instance-profile Name=lab-w3-ec2-profile \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-w3-private}]' \
  --query 'Instances[0].InstanceId' --output text)
```
The instance has no key pair, no public IP, and no inbound rule.

### Step 6: Interface endpoints so Systems Manager works without internet
Session Manager and Run Command need three interface endpoints when there is no NAT or internet route. These bill hourly, so delete them in Cleanup right after testing.

```bash
for SVC in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Interface \
    --service-name com.amazonaws.us-east-1.$SVC \
    --subnet-ids $PRIV_SUBNET --security-group-ids $VPCE_SG \
    --private-dns-enabled \
    --query VpcEndpoint.VpcEndpointId --output text
done
```
Wait 2 to 5 minutes, then confirm the instance is registered:

```bash
aws ssm describe-instance-information \
  --query 'InstanceInformationList[].[InstanceId,PingStatus]' --output text
```
It should show your instance as `Online`. Private DNS on the endpoints is why the SSM agent reaches the service with no internet route.

### Step 7: Static website bucket and CloudFront with OAC
```bash
aws s3api create-bucket --bucket $SITE_BUCKET --region us-east-1
aws s3api put-public-access-block --bucket $SITE_BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

cat > index.html <<'HTML'
<!doctype html><html><body style="font-family:sans-serif">
<h1>Week 3 static site</h1><p>Served by CloudFront through OAC.</p></body></html>
HTML
aws s3 cp index.html s3://$SITE_BUCKET/index.html --content-type text/html
```
Bucket stays fully private (all Block Public Access settings on, no S3 website hosting endpoint). Create the Origin Access Control and the distribution:

```bash
export OAC_ID=$(aws cloudfront create-origin-access-control --origin-access-control-config \
  Name=lab-w3-oac,SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3 \
  --query OriginAccessControl.Id --output text)

cat > cf-config.json <<JSON
{
  "CallerReference": "lab-w3-${SUFFIX}",
  "Comment": "lab-w3 static site",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "site-s3-origin",
      "DomainName": "${SITE_BUCKET}.s3.us-east-1.amazonaws.com",
      "OriginAccessControlId": "${OAC_ID}",
      "S3OriginConfig": {"OriginAccessIdentity": ""}
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "site-s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }
}
JSON

export DIST_ID=$(aws cloudfront create-distribution --distribution-config file://cf-config.json \
  --query Distribution.Id --output text)
export CF_DOMAIN=$(aws cloudfront get-distribution --id $DIST_ID \
  --query Distribution.DomainName --output text)
echo $CF_DOMAIN
```
The cache policy ID above is the AWS managed `Managed-CachingOptimized` policy; confirm with `aws cloudfront list-cache-policies --type managed` if the call is rejected. Now allow only this distribution to read the bucket:

```bash
cat > site-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::${SITE_BUCKET}/*",
    "Condition": {"StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::${ACCOUNT_ID}:distribution/${DIST_ID}"}}
  }]
}
JSON
aws s3api put-bucket-policy --bucket $SITE_BUCKET --policy file://site-policy.json
aws cloudfront wait distribution-deployed --id $DIST_ID
```
Waiting can take around 5 to 15 minutes. Console: **CloudFront > Distributions > Origins** shows the OAC attached to the S3 origin. Do the remaining steps while you wait if you prefer (run the wait later).

### Step 8: Lock the data bucket to the endpoint
Deny only the data-plane actions, not `s3:*`, so you cannot lock yourself out of managing the policy.

```bash
cat > data-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnlessFromVpce",
    "Effect": "Deny",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::${DATA_BUCKET}",
      "arn:aws:s3:::${DATA_BUCKET}/*"
    ],
    "Condition": {"StringNotEquals": {"aws:SourceVpce": "${S3_VPCE}"}}
  }]
}
JSON
aws s3api put-bucket-policy --bucket $DATA_BUCKET --policy file://data-policy.json
```
An explicit Deny overrides any Allow, so even an administrator with `AdministratorAccess` is denied object access from outside the endpoint. Recovery path if you ever need it: `aws s3api delete-bucket-policy` is not in the Deny list, so an admin can still remove the policy. (The root user can also always fix a bad bucket policy.)

### Step 9: Prove the private instance can use S3 and cannot reach the internet
Run commands on the instance via SSM (no SSH). Helper to run and print output:

```bash
run() {
  CMD_ID=$(aws ssm send-command --instance-ids $INSTANCE_ID \
    --document-name AWS-RunShellScript \
    --parameters "commands=[\"$1\"]" --query Command.CommandId --output text)
  sleep 6
  aws ssm get-command-invocation --command-id $CMD_ID --instance-id $INSTANCE_ID \
    --query '[Status,StandardOutputContent,StandardErrorContent]' --output text
}

run "echo hello-from-private > /tmp/a.txt && aws s3 cp /tmp/a.txt s3://${DATA_BUCKET}/a.txt --region us-east-1"
run "aws s3 ls s3://${DATA_BUCKET}/ --region us-east-1"
run "curl -sS -m 5 https://example.com || echo NO-INTERNET"
run "curl -sS -m 5 https://checkip.amazonaws.com || echo NO-INTERNET"
```
Expected: upload and list succeed through the gateway endpoint, while the `curl` commands time out and print `NO-INTERNET`. If a command is still `InProgress`, call `get-command-invocation` again after a few seconds.

### Step 10: Failure experiment A, direct access from outside the VPC is denied
From your laptop or CloudShell with your admin credentials:

```bash
aws s3 ls s3://$DATA_BUCKET/
aws s3 cp s3://$DATA_BUCKET/a.txt -
```
Expected: `AccessDenied` with an explicit deny in a resource-based policy. Your IAM permissions are admin, yet the bucket policy wins because explicit Deny always beats Allow. (If the console is used from your browser, listing objects is also denied for the same reason.)

### Step 11: Failure experiment B, remove the endpoint and watch the instance lose S3
This shows that the private instance depends on the endpoint route.

```bash
aws ec2 modify-vpc-endpoint --vpc-endpoint-id $S3_VPCE --remove-route-table-ids $PRIV_RT
sleep 30
run "timeout 15 aws s3 ls s3://${DATA_BUCKET}/ --region us-east-1 || echo S3-UNREACHABLE"
aws ec2 modify-vpc-endpoint --vpc-endpoint-id $S3_VPCE --add-route-table-ids $PRIV_RT
sleep 30
run "aws s3 ls s3://${DATA_BUCKET}/ --region us-east-1"
```
Expected: unreachable (timeout) without the route, and working again after you restore it.

### Step 12: Versioning in action
From the instance, overwrite the same key, then list versions from your laptop. Listing versions with `list-object-versions` also needs `s3:ListBucketVersions`, which the bucket policy above does not deny, so your admin identity can see the metadata:

```bash
run "echo version-two > /tmp/a.txt && aws s3 cp /tmp/a.txt s3://${DATA_BUCKET}/a.txt --region us-east-1"
aws s3api list-object-versions --bucket $DATA_BUCKET --prefix a.txt \
  --query 'Versions[].[Key,VersionId,IsLatest]' --output table
```
You should see two versions. The older one is a noncurrent version covered by the 30-day expiration rule. Reading it directly from outside the VPC is still denied.

### Step 13: Prove the website is served only through CloudFront
```bash
curl -sI https://$CF_DOMAIN/ | head -5            # 200 OK, served by CloudFront
curl -s https://$CF_DOMAIN/
curl -sI https://${SITE_BUCKET}.s3.us-east-1.amazonaws.com/index.html | head -3   # 403 Forbidden
```
If CloudFront shows a 403 right after creation, wait a few minutes for the distribution to finish deploying and for the bucket policy to take effect. Also try an HTTP request: `curl -sI http://$CF_DOMAIN/` should return a redirect to HTTPS.

## Validation
- Private instance: no public IP (`aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].PublicIpAddress'` is `None`), and the private route table has only `local` and the S3 prefix-list route.
- Instance S3 upload and list succeed; `curl` to the internet fails.
- Outside the VPC, `aws s3 ls s3://$DATA_BUCKET` returns AccessDenied.
- `aws s3api get-bucket-versioning` shows `Enabled` and `get-bucket-lifecycle-configuration` returns your rule.
- `https://$CF_DOMAIN/` returns the page; the direct S3 URL returns 403.

## Cleanup
Do this the same day. Interface endpoints and the instance are the hourly-billed items; remove them first.

```bash
# 1. Instance, then its profile and role
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
aws iam remove-role-from-instance-profile --instance-profile-name lab-w3-ec2-profile \
  --role-name lab-w3-ec2-role
aws iam delete-instance-profile --instance-profile-name lab-w3-ec2-profile
aws iam delete-role-policy --role-name lab-w3-ec2-role --policy-name data-bucket-rw
aws iam detach-role-policy --role-name lab-w3-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name lab-w3-ec2-role

# 2. VPC endpoints (interface and gateway)
VPCE_IDS=$(aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC_ID \
  --query 'VpcEndpoints[].VpcEndpointId' --output text)
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $VPCE_IDS
sleep 90   # interface endpoint network interfaces take a moment to disappear

# 3. Data bucket: remove the Deny policy first, then delete ALL versions and delete markers
aws s3api delete-bucket-policy --bucket $DATA_BUCKET
aws s3api list-object-versions --bucket $DATA_BUCKET --output json \
  --query '{Objects: [Versions,DeleteMarkers][][].{Key:Key,VersionId:VersionId}}' > del.json
aws s3api delete-objects --bucket $DATA_BUCKET --delete file://del.json
aws s3api delete-bucket --bucket $DATA_BUCKET
# (If delete-objects reports no objects, the bucket is already empty. Re-run list-object-versions to confirm.)

# 4. CloudFront: disable, wait, delete the distribution, then the OAC
ETAG=$(aws cloudfront get-distribution-config --id $DIST_ID --query ETag --output text)
aws cloudfront get-distribution-config --id $DIST_ID --query DistributionConfig \
  | jq '.Enabled=false' > cf-disabled.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file://cf-disabled.json
aws cloudfront wait distribution-deployed --id $DIST_ID
ETAG=$(aws cloudfront get-distribution --id $DIST_ID --query ETag --output text)
aws cloudfront delete-distribution --id $DIST_ID --if-match $ETAG
OAC_ETAG=$(aws cloudfront get-origin-access-control --id $OAC_ID --query ETag --output text)
aws cloudfront delete-origin-access-control --id $OAC_ID --if-match $OAC_ETAG

# 5. Site bucket
aws s3 rm s3://$SITE_BUCKET --recursive
aws s3api delete-bucket --bucket $SITE_BUCKET

# 6. Network (dependents first)
aws ec2 delete-security-group --group-id $VPCE_SG
aws ec2 delete-security-group --group-id $EC2_SG
aws ec2 delete-subnet --subnet-id $PRIV_SUBNET
aws ec2 delete-subnet --subnet-id $PUB_SUBNET
aws ec2 delete-route-table --route-table-id $PRIV_RT
aws ec2 delete-route-table --route-table-id $PUB_RT
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-vpc --vpc-id $VPC_ID
```
If `delete-objects` rejects an empty list (when the bucket is already empty), skip it. If a subnet or security group will not delete because of a lingering endpoint network interface, wait a couple of minutes and retry. Verify in **VPC > Endpoints** and **CloudFront** that nothing remains.

### What Would You Change for Production
- Use two or more AZs with a private subnet per AZ, and one interface endpoint network interface per AZ; use NAT Gateways only if workloads genuinely need outbound internet.
- Add an endpoint policy on the S3 gateway endpoint (restrict to your buckets), and keep bucket policies using `aws:SourceVpce` or `aws:SourceVpc` plus IAM roles. Combine with S3 Block Public Access at the account level.
- Use SSE-KMS or SSE-S3 per data classification, S3 Versioning plus MFA Delete or Object Lock for ransomware resilience, and replication to another Region if required.
- Put an ACM certificate and your own domain (Route 53 alias) on CloudFront, add AWS WAF, security response headers, and logging to a separate bucket. Add VPC Flow Logs for the private subnet.
- Manage everything as code (CloudFormation or Terraform) rather than CLI scripts.

## Exam Takeaway
- Domain 1, Design Secure Architectures: private subnets, security groups versus network ACLs (stateful versus stateless), VPC endpoints for private AWS service access, bucket policy conditions (`aws:SourceVpce`, `aws:SecureTransport`), Block Public Access, OAC for CloudFront-to-S3 (OAC replaces the older OAI), and explicit Deny precedence.
- Gateway endpoints exist only for S3 and DynamoDB, are free, and work through route tables. Interface endpoints (PrivateLink) cover most other services, use ENIs and security groups, and are billed hourly. A NAT Gateway is the answer for outbound internet from private subnets, but it is not needed to reach S3 or DynamoDB.
- Domain 3, High-Performing Architectures: CloudFront caches at edge locations for lower latency; S3 gateway endpoints avoid NAT data processing for S3 traffic.
- Domain 4, Cost-Optimized Architectures: use gateway endpoints instead of NAT for S3 traffic; use lifecycle rules (Standard to Standard-IA, expire noncurrent versions) to control storage cost; know the 30-day minimum for IA transitions.
- Domain 2, Design Resilient Architectures: versioning protects against accidental overwrite or deletion; S3 durability is designed for 99.999999999%, and Standard stores data across multiple AZs.

## Stretch Goals
1. Add an S3 gateway endpoint policy that allows only your two lab buckets, and show that a request to another bucket from the instance is denied.
2. Add a second private subnet in `us-east-1b` and a second route table association, and discuss which resources in this lab are AZ-scoped.
3. Enable VPC Flow Logs on the private subnet to a CloudWatch Logs group, then find the rejected outbound internet attempts from the `curl` test.
4. Add a CloudFront custom error response for 403/404 to a friendly `error.html`, and invalidate the cache with `aws cloudfront create-invalidation`.


---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
