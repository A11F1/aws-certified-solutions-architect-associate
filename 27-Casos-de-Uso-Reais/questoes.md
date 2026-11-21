# Review Questions

## Question 1

A scenario requires the right decision on real-world cases. Which approach fits best?

A) Apply a generic solution without considering e-commerce.
B) Use e-commerce with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of real-world cases with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take e-commerce into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine SaaS with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of real-world cases with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of global when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of real-world cases with the best balance between requirement, risk, and operations.

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
B) The one that ignores modernization signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of real-world cases with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on modernization and analytics with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of real-world cases with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

An online retailer runs a flash sale and expects order volume to jump 50 times for about 30 minutes. The web tier (an Application Load Balancer and an Auto Scaling group) can absorb the traffic, but the order-processing backend calls a slow third-party payment API and cannot keep up. No order may be lost, and the backend should scale based on the amount of pending work.

Which architecture meets these requirements?

A) Have the web tier call the order-processing service synchronously through an internal Application Load Balancer and use larger EC2 instances for the backend
B) Publish each order to an Amazon SNS topic with HTTP endpoints that point at the backend instances
C) Have the web tier send each order to an Amazon SQS queue and run the backend as an Auto Scaling group that scales on the queue's ApproximateNumberOfMessagesVisible metric
D) Write each order to an Amazon Kinesis Data Streams stream with a fixed number of shards and process it with a single consumer application on one EC2 instance

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Option C decouples the tiers. SQS durably stores orders until workers process them, so a slow payment API only grows the backlog and nothing is lost. Scaling the worker group on queue depth matches capacity to pending work.

**Why option A is wrong:**
A synchronous call couples the web tier to the slow payment API. Requests time out during the spike, orders can be lost, and larger instances only raise the ceiling without buffering anything.

**Why option B is wrong:**
SNS pushes messages to endpoints and retries only for a limited period. If the backend is saturated or down, messages can be dropped. There is no durable backlog, and no queue depth to scale on.

**Why option D is wrong:**
A single consumer on one instance is a bottleneck and a single point of failure. A fixed shard count also caps throughput, and the design has no scaling mechanism for the backend.

</details>

## Question 7

A startup is building a multi-tenant SaaS product with a single-page web front end and a REST API. Traffic is unpredictable, the team has no operations staff, and users must sign in with email and password as well as social identity providers. The company wants to avoid managing servers.

Which architecture best fits?

A) Host the front end on Amazon S3 behind Amazon CloudFront, authenticate with an Amazon Cognito user pool, and serve the API with Amazon API Gateway, AWS Lambda, and Amazon DynamoDB
B) Run the front end and API on EC2 instances in an Auto Scaling group and deploy a self-managed identity server on separate EC2 instances for sign-in
C) Deploy the application on AWS Elastic Beanstalk and create an IAM user for each customer so they can sign in and call the API
D) Run the application on a single large Amazon Lightsail instance with a local database and handle sign-in in application code

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Option A is fully managed and serverless. S3 with CloudFront serves the front end globally, Cognito user pools support password sign-in and social federation, and API Gateway, Lambda, and DynamoDB scale automatically and need no server management.

**Why option B is wrong:**
This works technically, but it requires patching, scaling, and high availability for both the application servers and the identity server. That is the operational burden the team wants to avoid.

**Why option C is wrong:**
Creating IAM users for end customers is an anti-pattern. IAM is meant for workforce and AWS access, it has account quotas that do not scale to a customer base, and it does not provide social sign-in. Cognito is the service for application users.

**Why option D is wrong:**
A single instance with a local database is a single point of failure and does not absorb unpredictable traffic. Custom sign-in code also adds security and maintenance work.

</details>

## Question 8

A company has 100,000 connected devices that send telemetry as JSON. The company wants to ingest the data continuously, buffer it, and store it in Amazon S3 in a columnar format so that analysts can run ad hoc SQL queries without managing any servers or clusters.

Which solution meets these requirements with the LEAST operational effort?

A) Have each device write every reading directly into an Amazon RDS for MySQL database and let analysts query it
B) Send the data to an Amazon SQS queue, poll it with EC2 instances, and write the files to an EBS volume that analysts query with a custom tool
C) Write every reading to Amazon DynamoDB and have analysts run full table scans for their reports
D) Ingest with AWS IoT Core, deliver through Amazon Data Firehose with record format conversion to Apache Parquet into Amazon S3, and query with Amazon Athena

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Option D is a managed pipeline from end to end. Firehose buffers the stream and can convert JSON to Parquet, S3 stores it cheaply, and Athena runs serverless SQL directly on the files.

**Why option A is wrong:**
Writing 100,000 devices directly to a single relational database is a poor fit for the write rate, and analytic queries on the same database would compete with ingestion. It also does not store data in S3 as required.

**Why option B is wrong:**
The EC2 pollers and EBS storage have to be managed and scaled. EBS is also not a data lake store that serverless SQL can query.

**Why option C is wrong:**
DynamoDB can take the writes, but full table scans for analytics are slow and expensive. The data does not land in S3 in a columnar format as the requirement states.

</details>

## Question 9

A company has a legacy three-tier application whose Amazon RDS for MySQL database (Single-AZ) is overloaded by long-running reporting queries from the finance team. The database also has no high-availability protection. The company wants to fix both problems with minimal application changes.

What should a solutions architect recommend?

A) Convert the database to a Multi-AZ DB instance deployment and point the reporting queries at the standby instance
B) Convert the database to Multi-AZ for availability and create a read replica for the reporting queries
C) Add an Amazon ElastiCache cluster in front of the database and cache every report result
D) Migrate the data to Amazon DynamoDB and rewrite the reports as scans

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Multi-AZ provides a synchronous standby and automatic failover for availability. A read replica uses asynchronous replication and serves read-only reporting traffic, which takes the load off the primary with only a connection string change for the reports.

**Why option A is wrong:**
In a Multi-AZ DB instance deployment, the standby exists only for failover and cannot accept read traffic. The reports would have nowhere to run.

**Why option C is wrong:**
Ad hoc, long-running report queries rarely repeat identically, so a cache gives little benefit. It also adds application changes and does nothing for availability.

**Why option D is wrong:**
Moving a relational schema to DynamoDB requires a large rewrite of the application and reports. DynamoDB scans are a poor replacement for relational reporting queries, so this violates the minimal-change requirement.

</details>

## Question 10

A retailer runs its order platform on Amazon Aurora MySQL in us-east-1 with a web tier on EC2. A disaster recovery plan for a full Regional outage requires an RPO of about one second and an RTO of 15 minutes. The company accepts a small running footprint in the recovery Region but does not want to pay for a full duplicate environment.

Which strategy meets these requirements?

A) Copy automated Aurora snapshots to eu-west-1 every hour and restore them if us-east-1 fails
B) Enable Multi-AZ on the Aurora cluster so a replica in another Availability Zone takes over
C) Create an Aurora global database with a secondary cluster in eu-west-1, keep a minimal web tier running there, and use Amazon Route 53 failover routing to shift traffic
D) Replicate backups to S3 in eu-west-1 with cross-Region replication and rebuild the whole stack with AWS CloudFormation during a disaster

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
An Aurora global database replicates storage to the secondary Region with typically under one second of lag, which satisfies the RPO. A minimal running web tier (warm standby) can be scaled up quickly, and Route 53 failover redirects users, which fits the 15-minute RTO at moderate cost.

**Why option A is wrong:**
Hourly snapshot copies imply an RPO of up to an hour, far above one second. Restoring a database from a snapshot also takes longer than the RTO allows.

**Why option B is wrong:**
Multi-AZ protects against an Availability Zone failure inside one Region. It does not help when the entire Region is unavailable.

**Why option D is wrong:**
Backup and restore with a rebuild takes hours and has an RPO equal to the backup frequency. This is the cheapest DR strategy but cannot meet a one-second RPO and a 15-minute RTO.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> To turn this architecture foundation into practice with pipelines, data, and analytics, you can also follow the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
