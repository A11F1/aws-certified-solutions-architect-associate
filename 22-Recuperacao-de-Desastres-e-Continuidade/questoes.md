# Review Questions

## Question 1

A scenario requires the right decision on resilience and DR. Which approach fits best?

A) Apply a generic solution without considering Multi-AZ.
B) Use Multi-AZ with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resilience and DR with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take Multi-AZ into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine failover with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resilience and DR with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Manual handling is slow, error-prone, and does not scale. Combining automation with observability is what actually reduces operational risk.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 3

What should you do to keep cost under control without losing technical quality?

A) Always pick the cheapest option without validating the requirement.
B) Size based on actual usage and take advantage of DR strategies when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resilience and DR with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Cost is only one constraint. The cheapest option can break availability, performance, or security requirements, so it must be validated against the scenario first.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 4

Which alternative tends to be incorrect on the exam for this topic?
A) The one that meets the dominant requirement with the least complexity.
B) The one that ignores RTO/RPO signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resilience and DR with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
This is the pattern that is usually correct on the exam. The question asks which alternative tends to be incorrect, so this is the opposite of what is being asked.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 5

Which review approach improves accuracy in this module?

A) Review only definitions without scenario practice.
B) Practice questions focused on decoupling and health checks with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resilience and DR with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company runs a revenue-critical web application in us-east-1. The business requires an RTO of about 15 minutes and an RPO of seconds for a regional failure, but it will not pay for a fully duplicated active-active environment. Which disaster recovery strategy is the best fit?

A) Backup and restore with snapshots copied to another Region
B) Warm standby with a continuously replicated database and a scaled-down copy of the application running in the DR Region
C) Multi-site active-active with full capacity in both Regions
D) Pilot light with only the database replicated and no application servers running

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Warm standby keeps a functional but smaller version of the environment always running and the data continuously replicated. On failover, the fleet is scaled up and traffic is redirected, which fits a 15-minute RTO and near-zero RPO at a fraction of active-active cost. Option B balances the targets and the budget.

**Why option A is wrong:**
Restoring from snapshots takes hours and loses all data since the last snapshot, which fails both the RTO and the RPO.

**Why option C is wrong:**
Active-active meets the targets, but it duplicates full capacity in both Regions and is the most expensive option, which the company explicitly rejected.

**Why option D is wrong:**
In a pilot light, application servers must be launched and configured during recovery. That generally takes longer than a 15-minute RTO and is a slower, cheaper tier than warm standby.

</details>

## Question 7

A SaaS provider uses Amazon Aurora MySQL in us-east-1. For regional disaster recovery, it needs an RPO of about one second and the ability to promote a secondary Region to read-write in under a minute. Which design should the architect choose?

A) Enable Multi-AZ on the Aurora cluster
B) Copy automated snapshots to the secondary Region every hour
C) Take a daily logical dump of the database and store it in a bucket replicated to another Region
D) Use an Aurora Global Database with a secondary cluster in the DR Region

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Aurora Global Database replicates data at the storage layer to secondary Regions, typically with latency under one second. A secondary cluster can be promoted to a standalone read-write cluster in roughly a minute. Option D meets both the RPO and the RTO.

**Why option A is wrong:**
Multi-AZ protects against the loss of an Availability Zone within one Region. It does not help during a regional outage.

**Why option B is wrong:**
Hourly snapshot copies give an RPO of up to an hour and require a restore that takes far longer than a minute.

**Why option C is wrong:**
Daily dumps give an RPO of up to 24 hours, and importing a dump into a new cluster takes a long time.

</details>

## Question 8

A company has 50 AWS accounts in an organization. Auditors require centralized backup policies for EBS, RDS, EFS, and DynamoDB, with a guarantee that nobody, including administrators, can delete the backups for one year. Which solution meets this requirement with the least operational effort?

A) Use AWS Backup with organization backup policies and AWS Backup Vault Lock in compliance mode
B) Write Lambda functions in each account that take snapshots and tag them for retention
C) Enable S3 Object Lock in compliance mode on a central bucket
D) Use Amazon Data Lifecycle Manager policies in each account

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
AWS Backup centralizes backup plans across supported services and, with AWS Organizations, applies backup policies to many accounts. Vault Lock in compliance mode enforces WORM retention that even the root user cannot override once the cooling-off period ends. Option A covers all four services and the immutability requirement.

**Why option B is wrong:**
Custom Lambda code per account is high-maintenance, and tags cannot prevent anyone with permissions from deleting the snapshots.

**Why option C is wrong:**
S3 Object Lock only protects objects in S3. It does not create or lock backups of EBS, RDS, EFS, or DynamoDB resources.

**Why option D is wrong:**
Data Lifecycle Manager automates EBS snapshots and AMIs only, so it does not cover RDS, EFS, or DynamoDB, and it does not provide immutable vaults.

</details>

## Question 9

An application runs behind an Application Load Balancer in us-east-1, with a standby stack behind another ALB in us-west-2. The company wants Route 53 to send users to the standby stack automatically if the primary becomes unhealthy, and to return to the primary when it recovers. Which Route 53 configuration should be used?

A) A simple routing policy record containing both ALB aliases
B) A single CNAME record that an on-call engineer updates manually during an incident
C) A failover routing policy with a primary record associated with a health check and a secondary record for the standby
D) A geolocation routing policy that maps all users to us-west-2

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Failover routing designates a primary and a secondary record. Route 53 answers with the primary while its health check passes, and with the secondary when it fails, then returns to the primary after recovery. Option C automates the failover and failback.

**Why option A is wrong:**
A simple record with multiple values returns them without a primary and secondary relationship, so some users would be sent to the standby even when the primary is healthy.

**Why option B is wrong:**
Manual updates depend on a person noticing the outage and are delayed by DNS caching, so failover is neither automatic nor fast.

**Why option D is wrong:**
Geolocation routing is based on where the user is, not on the health of the endpoints, and mapping everyone to us-west-2 would make the standby the permanent primary.

</details>

## Question 10

A company has 60 on-premises physical and virtual servers running a critical application. It wants to use AWS as its disaster recovery site with an RPO of seconds and an RTO of minutes, but it wants to avoid paying for a full duplicate environment while no disaster occurs. Which service fits?

A) Nightly backups shipped to AWS with Snowball Edge
B) A fully provisioned fleet of EC2 instances running in parallel as a warm standby
C) AWS Storage Gateway file gateway
D) AWS Elastic Disaster Recovery

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
AWS Elastic Disaster Recovery continuously replicates source servers at the block level to a low-cost staging area in AWS, using lightweight replication servers and EBS volumes. During a drill or real event, it launches full-size recovery instances, typically within minutes. Option D gives low RPO and RTO while keeping idle cost low.

**Why option A is wrong:**
Nightly backups give an RPO of about a day, and rebuilding servers from backup takes much longer than minutes.

**Why option B is wrong:**
A full fleet running all the time works technically but pays for the entire duplicate compute capacity continuously, which the company wants to avoid.

**Why option C is wrong:**
A file gateway provides file access to S3 objects. It does not replicate whole servers or let the company launch them as recovery instances.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> To turn this architecture foundation into practice with pipelines, data, and analytics, you can also follow the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
