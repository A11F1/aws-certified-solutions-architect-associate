# Guided Lab: Build Your Own AWS Glossary by Exploring Real Accounts and APIs

## Objective

Learn the recurring architecture vocabulary by seeing it in a real account. You will run a few CLI commands and console checks, note the Shared Responsibility split for each service you touch, and then write your own definitions for 15 terms before comparing them with reference definitions.

## Scenario

Exam questions rely on precise terms: a design that is "highly available" is not necessarily "fault tolerant", and RTO is not RPO. You will anchor each term to something you can see: Availability Zones, Regions, quotas, tags, and policy evaluation.

## Services Used

- Amazon S3 (bucket and tags)
- AWS IAM (policy simulator)
- Amazon EC2 control plane (read-only describe calls)
- Service Quotas
- AWS Artifact / documentation page for the Shared Responsibility Model (read-only)

## Prerequisites

- Personal AWS account, region `us-east-1`, AWS CLI v2 configured.
- An IAM identity that can create buckets and run read-only describe calls.

## Expected Cost

$0. An empty S3 bucket and read-only API calls cost nothing. Avoid uploading objects to the bucket, and delete the bucket at the end.

## Step by Step

### Step 1: Explore Regions and Availability Zones

```bash
aws ec2 describe-regions --all-regions \
  --query "Regions[].[RegionName,OptInStatus]" --output table

aws ec2 describe-availability-zones --region us-east-1 \
  --query "AvailabilityZones[].[ZoneName,ZoneId,ZoneType,State]" --output table
```

Observe: regions marked `opt-in-not-required` are on by default; others need to be enabled. Every AZ has a zone name (`us-east-1a`) and a zone ID (`use1-az1`). Zone names are mapped per account, so the same name can be a different physical AZ in another account; zone IDs are consistent. Write this next to "multi-AZ" and "multi-Region" in your table.

### Step 2: Create a bucket and tag it

```bash
BUCKET=glossary-lab-$(aws sts get-caller-identity --query Account --output text)
aws s3api create-bucket --bucket $BUCKET --region us-east-1

aws s3api put-bucket-tagging --bucket $BUCKET \
  --tagging 'TagSet=[{Key=Project,Value=glossary-lab},{Key=Owner,Value=me}]'
aws s3api get-bucket-tagging --bucket $BUCKET
```

Console: S3 > bucket > Properties > Tags. Observe that S3 is a regional service: the bucket lives in one Region and S3 stores objects redundantly across multiple AZs by default (in S3 Standard). Tags are metadata for cost allocation and access control, not security by themselves.

### Step 3: Test a policy with the IAM policy simulator

Console: open the IAM Policy Simulator at policysim.aws.amazon.com, pick a user or role, select service S3, action `GetObject`, and run the simulation. Or by CLI:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<ACCOUNT_ID>:user/<USER_NAME> \
  --action-names s3:GetObject s3:DeleteBucket \
  --resource-arns arn:aws:s3:::<BUCKET_NAME> \
  --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output table
```

You will see `allowed`, `explicitDeny`, or `implicitDeny` per action. By default everything is implicitly denied; an explicit allow grants it; an explicit deny always wins. Use the result to write your definition of "least privilege".

### Step 4: View Service Quotas

Console: Service Quotas > AWS services > Amazon EC2 > search "Running On-Demand Standard". CLI:

```bash
aws service-quotas list-service-quotas --service-code ec2 \
  --query "Quotas[?contains(QuotaName,'On-Demand Standard')].[QuotaName,Value,Adjustable]" --output table
```

Observe the value, whether it is adjustable, and that quotas are per Region. Hitting a quota is related to "throttling" for API rate limits, but a quota is a resource ceiling you can often raise, while throttling is rate-based and answered with retries and backoff.

### Step 5: Map the Shared Responsibility Model

Open the AWS Shared Responsibility Model page in the AWS documentation. For each service you touched, fill this table:

| Service | AWS is responsible for | I am responsible for |
|---------|------------------------|----------------------|
| S3 | Durability of storage, hardware, facility, service software | Bucket policy, public access settings, encryption choices, data classification |
| EC2 | Hypervisor, hardware, facilities | Guest OS patching, security groups, IAM roles, application, data |
| IAM | The IAM service itself | Users, roles, policies, MFA, key rotation |
| Service Quotas / EC2 API | Enforcing and operating the API | Requesting increases, designing around throttling |

The rule of thumb: AWS is responsible for security "of" the cloud; you are responsible for security "in" the cloud, and the split shifts toward AWS as services become more managed (for example RDS patches the database engine, but EC2 with a self-installed database does not).

### Step 6: Write your definitions for 15 terms

Fill the "My definition" column from memory first, and write one real AWS example. Do not open the reference block until you are done.

| # | Term | My definition | AWS example |
|---|------|---------------|-------------|
| 1 | RTO | | |
| 2 | RPO | | |
| 3 | High availability | | |
| 4 | Fault tolerance | | |
| 5 | Elasticity | | |
| 6 | Scalability | | |
| 7 | Eventual consistency | | |
| 8 | Idempotency | | |
| 9 | Blast radius | | |
| 10 | Least privilege | | |
| 11 | Stateless vs. stateful | | |
| 12 | Multi-AZ | | |
| 13 | Multi-Region | | |
| 14 | Throttling | | |
| 15 | SLA | | |

<details>
<summary>Reference definitions (open after you have written yours)</summary>

1. **RTO (Recovery Time Objective):** the maximum acceptable time between an outage and restoring service. Example: a warm standby environment targets an RTO of minutes.
2. **RPO (Recovery Point Objective):** the maximum acceptable amount of data loss measured in time. Example: hourly backups imply an RPO of up to one hour; continuous replication gives seconds.
3. **High availability:** designing to minimize downtime by removing single points of failure, typically with redundancy across AZs. A brief interruption during failover is acceptable. Example: an ASG and ALB across two AZs.
4. **Fault tolerance:** the system continues operating without interruption when a component fails, usually at higher cost and complexity. Example: S3 and DynamoDB tolerate AZ failure transparently.
5. **Elasticity:** the ability to add and remove resources automatically to match demand, in both directions. Example: Auto Scaling scale-out and scale-in.
6. **Scalability:** the ability to handle increased load by adding capacity (vertically by a larger instance, or horizontally by more instances). Elasticity is scalability that also shrinks automatically.
7. **Eventual consistency:** after a write, reads may briefly return old data but all copies converge. Example: DynamoDB default reads (strongly consistent reads are optional); S3 itself now provides strong read-after-write consistency.
8. **Idempotency:** performing the same operation multiple times produces the same result as once. Example: retrying a Lambda event that writes by a fixed key.
9. **Blast radius:** the scope of impact when something fails or is compromised. Reduced by separate accounts, cells, AZ isolation, and narrow permissions.
10. **Least privilege:** grant only the permissions needed for a task and nothing more, scoped to specific actions and resources, and review them over time.
11. **Stateless vs. stateful:** stateless components keep no session or data locally between requests, so any instance can serve any request and be replaced freely; stateful components hold data that must persist or be shared. Example: web servers (stateless) versus databases (stateful).
12. **Multi-AZ:** deploying across multiple Availability Zones in one Region for availability. Example: RDS Multi-AZ standby with automatic failover.
13. **Multi-Region:** deploying across multiple Regions for disaster recovery, latency, or compliance. Example: DynamoDB global tables, S3 Cross-Region Replication.
14. **Throttling:** a service rejecting or slowing requests that exceed a rate limit, returning errors such as HTTP 429 or `ThrottlingException`. Clients should retry with exponential backoff and jitter.
15. **SLA (Service Level Agreement):** AWS's published availability commitment for a service, with service credits as the remedy if it is missed. It is not a guarantee of uptime, and your architecture can affect which SLA applies.

</details>

## Validation

- Your `describe-availability-zones` output lists six AZs for us-east-1 (as of this writing, though counts can change) and you can explain zone name versus zone ID.
- `get-bucket-tagging` returns your two tags.
- The simulator shows an explicit decision (`allowed` or `implicitDeny`) for each action.
- All 15 terms have your own definition and an AWS example, and you corrected at least three after reading the reference.

## Cleanup

```bash
aws s3api delete-bucket --bucket $BUCKET --region us-east-1
```

The simulator and describe calls create nothing. Keep your completed glossary table.

## Exam Takeaway

Many exam options differ by one term. "Highly available" means survive failures with brief interruption (multi-AZ), "fault tolerant" means no interruption, and "disaster recovery" means recovering in another Region with RTO and RPO targets that drive the cost of the solution. Lower RTO/RPO means higher cost. Also remember the Shared Responsibility rule: the more managed the service, the more AWS handles, but you always own your data, identity, and access configuration.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
