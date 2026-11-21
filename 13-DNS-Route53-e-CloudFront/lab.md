# Guided Lab: Private S3 Website Behind CloudFront (OAC) and Route 53 Routing Policies

## Objective

Serve a static site from a private S3 bucket through CloudFront using Origin Access Control, verify caching behavior from response headers, prove that direct S3 access is denied, and learn how Route 53 routing policies (simple, weighted, latency, failover with health checks) map to requirements.

## Scenario

A marketing site must be fast worldwide, HTTPS-only, and the bucket must never be public. A domain is optional: the CloudFront-part works with the default `*.cloudfront.net` name, and the Route 53 part only needs a domain if you own one.

## Services Used

- Amazon S3 (private bucket)
- Amazon CloudFront (distribution, Origin Access Control, invalidation)
- Amazon Route 53 (hosted zone, records, health checks; optional)
- AWS CLI v2

## Prerequisites

- AWS account, region `us-east-1` (CloudFront is global; certificates for CloudFront must live in us-east-1).
- Permissions for S3, CloudFront and (optionally) Route 53.
- `curl`. A registered domain is optional.

## Expected Cost

- S3 and CloudFront for this lab: cents at most. CloudFront's always-free tier includes 1 TB data transfer out and 10 million requests per month.
- Route 53 hosted zone: $0.50/month (a zone deleted within 12 hours of creation is not charged); each health check is about $0.50/month for AWS endpoints. Registering a domain costs from about $12/year, so skip it if you do not need it.
- Avoid: leaving hosted zones and health checks around after the lab.

## Step by Step

### Step 1: Create a private bucket with a test page

Console: S3 > Create bucket `<YOUR_BUCKET_NAME>` in us-east-1, keep "Block all public access" ON.

```bash
BUCKET=<YOUR_BUCKET_NAME>
aws s3api create-bucket --bucket $BUCKET --region us-east-1
echo '<h1>Hello from CloudFront</h1>' > index.html
aws s3 cp index.html s3://$BUCKET/index.html --content-type text/html \
  --cache-control "max-age=300"
```

### Step 2: Create an Origin Access Control (OAC)

Console: CloudFront > Security > Origin access > Create control setting. Name `lab-oac`, origin type S3, sign requests (recommended).

```bash
cat > oac.json <<'EOF'
{
  "Name": "lab-oac",
  "Description": "OAC for lab bucket",
  "SigningProtocol": "sigv4",
  "SigningBehavior": "always",
  "OriginAccessControlOriginType": "s3"
}
EOF
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config file://oac.json --query 'OriginAccessControl.Id' --output text)
```

### Step 3: Create the distribution

Console: CloudFront > Create distribution > Origin domain: your S3 bucket (REST endpoint, not the website endpoint) > Origin access: Origin access control settings > select `lab-oac` > Viewer protocol policy: Redirect HTTP to HTTPS > Default root object `index.html` > WAF off > Create. Copy the suggested bucket policy afterwards.

```bash
cat > dist.json <<EOF
{
  "CallerReference": "lab-$(date +%s)",
  "Comment": "SAA lab",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "Origins": { "Quantity": 1, "Items": [{
    "Id": "s3-origin",
    "DomainName": "$BUCKET.s3.us-east-1.amazonaws.com",
    "OriginAccessControlId": "$OAC_ID",
    "S3OriginConfig": { "OriginAccessIdentity": "" }
  }]},
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  }
}
EOF
aws cloudfront create-distribution --distribution-config file://dist.json \
  --query 'Distribution.[Id,DomainName]' --output text
```

The cache policy ID above is the AWS managed `CachingOptimized` policy.

### Step 4: Allow only CloudFront to read the bucket

```bash
cat > policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*",
    "Condition": { "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
    }}
  }]
}
EOF
aws s3api put-bucket-policy --bucket $BUCKET --policy file://policy.json
aws cloudfront wait distribution-deployed --id <DISTRIBUTION_ID>
```

### Step 5: Test caching and the access-denied experiment

```bash
DOMAIN=<DISTRIBUTION_DOMAIN>.cloudfront.net
curl -sI https://$DOMAIN/ | egrep -i 'HTTP|x-cache|age|cache-control|via'   # Miss from cloudfront
curl -sI https://$DOMAIN/ | egrep -i 'HTTP|x-cache|age'                      # Hit from cloudfront, Age > 0

# Failure experiment: bypass CloudFront
curl -sI https://$BUCKET.s3.us-east-1.amazonaws.com/index.html | head -1     # HTTP/1.1 403 Forbidden
```

Now change the file and watch stale content, then fix it with an invalidation:

```bash
echo '<h1>Version 2</h1>' > index.html
aws s3 cp index.html s3://$BUCKET/index.html --content-type text/html --cache-control "max-age=300"
curl -s https://$DOMAIN/            # still Version 1 (cached)
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/index.html"
curl -s https://$DOMAIN/            # Version 2 after the invalidation completes (about a minute)
```

### Step 6: Route 53 routing policies (optional, needs a domain)

If you own `example.com`, create a hosted zone and point an alias at the distribution. Without a domain, just read the JSON to see how each policy is expressed.

```bash
ZONE_ID=$(aws route53 create-hosted-zone --name example.com --caller-reference lab-$(date +%s) \
  --query 'HostedZone.Id' --output text)
# then update the registrar's name servers to the 4 values from: aws route53 get-hosted-zone --id $ZONE_ID
```

Alias to CloudFront (the hosted zone ID `Z2FDTNDATAQYW2` is fixed for all CloudFront distributions):

```json
{ "Changes": [{ "Action": "UPSERT", "ResourceRecordSet": {
  "Name": "www.example.com", "Type": "A",
  "AliasTarget": { "HostedZoneId": "Z2FDTNDATAQYW2",
    "DNSName": "<DISTRIBUTION_DOMAIN>.cloudfront.net", "EvaluateTargetHealth": false } } }] }
```

Apply with `aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID --change-batch file://alias.json`. For the other policies, each record needs a unique `SetIdentifier`:

```json
{ "Name": "app.example.com", "Type": "A", "SetIdentifier": "v1", "Weight": 90, "TTL": 60, "ResourceRecords": [{"Value": "203.0.113.10"}] }
{ "Name": "app.example.com", "Type": "A", "SetIdentifier": "us", "Region": "us-east-1", "TTL": 60, "ResourceRecords": [{"Value": "203.0.113.10"}] }
{ "Name": "app.example.com", "Type": "A", "SetIdentifier": "primary", "Failover": "PRIMARY", "HealthCheckId": "<HEALTH_CHECK_ID>", "TTL": 60, "ResourceRecords": [{"Value": "203.0.113.10"}] }
```

First is weighted (90/10 canary), second is latency-based, third is failover (needs a matching `SECONDARY` record). Health check example:

```bash
aws route53 create-health-check --caller-reference hc-$(date +%s) --health-check-config \
  Type=HTTPS,FullyQualifiedDomainName=$DOMAIN,Port=443,ResourcePath=/,RequestInterval=30,FailureThreshold=3
```

Documentation addresses (203.0.113.x) are placeholders, so these records do not serve real traffic.

## Validation

- First request shows `X-Cache: Miss from cloudfront`, the second `Hit from cloudfront` with an `Age` header.
- The direct S3 URL returns 403, while the CloudFront URL returns 200.
- After the invalidation the new content is served.
- If you used a domain: `dig www.example.com` resolves to CloudFront IPs.

## Cleanup

Dependency order: distribution first (it must be disabled, deployed, then deleted), then bucket, then Route 53.

```bash
aws cloudfront get-distribution-config --id <DISTRIBUTION_ID> > cfg.json   # edit "Enabled": false, keep the ETag
aws cloudfront update-distribution --id <DISTRIBUTION_ID> --if-match <ETAG> --distribution-config file://disabled-config.json
aws cloudfront wait distribution-deployed --id <DISTRIBUTION_ID>
aws cloudfront delete-distribution --id <DISTRIBUTION_ID> --if-match <NEW_ETAG>
aws cloudfront delete-origin-access-control --id $OAC_ID --if-match <OAC_ETAG>
aws s3 rb s3://$BUCKET --force
# Route 53 (if created): delete non-default records, health checks, then the zone
aws route53 delete-health-check --health-check-id <HEALTH_CHECK_ID>
aws route53 delete-hosted-zone --id $ZONE_ID
```

The console is easier for the disable-then-delete dance: CloudFront > select distribution > Disable > wait > Delete.

## Exam Takeaway

CloudFront with OAC is the answer for "serve S3 content globally without making the bucket public" (OAC replaces the legacy OAI). Caching is controlled by cache policies and `Cache-Control`/TTL, and invalidations (or versioned file names) fix stale content. For Route 53, match policy to requirement: weighted for canary/A-B splits, latency for lowest-latency Region, failover for active-passive DR (requires health checks), geolocation for compliance by user location, and alias records (free, work at the zone apex) for AWS resources.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
