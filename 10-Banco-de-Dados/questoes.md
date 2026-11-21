# Review Questions

## Question 1

A scenario requires the right decision on databases. Which approach fits best?

A) Apply a generic solution without considering RDS.
B) Use RDS with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of databases with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take RDS into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine Aurora with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of databases with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of DynamoDB when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of databases with the best balance between requirement, risk, and operations.

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
B) The one that ignores ElastiCache signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of databases with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on backups and read replicas with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of databases with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

An e-commerce company runs its order system on a Multi-AZ Amazon RDS for MySQL instance. Analysts run long reporting queries during business hours, and order processing latency increases noticeably when they do. The reports can tolerate data that is a few seconds old. What is the MOST effective way to protect order processing performance?

A) Create a read replica and direct the reporting queries to the replica endpoint
B) Direct the reporting queries to the standby instance of the Multi-AZ deployment
C) Place the reporting queries behind the endpoint of an Amazon RDS Proxy
D) Add an ElastiCache cluster in front of the database and cache every report result

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
A read replica uses asynchronous replication and serves read traffic from separate compute and storage, so heavy queries no longer compete with order writes on the primary. Replica lag of a few seconds is acceptable for reports. The reporting tool only needs the replica's endpoint.

**Why option B is wrong:**
In a standard Multi-AZ DB instance deployment, the standby is for failover only and cannot accept connections or read traffic.

**Why option C is wrong:**
RDS Proxy pools and shares database connections and improves failover handling, but queries still run on the same primary instance, so the load stays the same.

**Why option D is wrong:**
Caching helps repeated identical reads. Ad hoc analytical queries rarely repeat, so most would still hit the primary, and it adds an application change and a cache to manage.

</details>

## Question 7

A global SaaS company runs its application on Amazon RDS for MySQL in us-east-1. A new disaster recovery requirement states that, if the Region fails, the database must be available in eu-west-1 with an RPO of about 1 second and an RTO of under 1 minute, with minimal data-loss risk and low operational effort. Which solution should the company choose?

A) Create a cross-Region read replica of the RDS for MySQL instance and promote it manually during a disaster
B) Enable automated backups and copy the snapshots to eu-west-1 every hour
C) Migrate to Amazon Aurora MySQL and use an Aurora Global Database with a secondary cluster in eu-west-1
D) Convert the RDS instance to a Multi-AZ deployment

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Aurora Global Database replicates storage-level changes to secondary Regions, typically with under one second of lag. A secondary cluster can be promoted to a standalone read/write cluster in about a minute, matching the RPO and RTO. It is MySQL-compatible, so the application changes are small.

**Why option A is wrong:**
A cross-Region RDS read replica uses asynchronous binlog replication with higher and more variable lag, and promotion plus application re-pointing is a manual process that is unlikely to meet a one-minute RTO.

**Why option B is wrong:**
Hourly snapshot copies imply an RPO of up to an hour, and restoring a snapshot takes far longer than a minute.

**Why option D is wrong:**
Multi-AZ protects against failures of an instance or an Availability Zone within one Region only. It does nothing for a Regional failure.

</details>

## Question 8

A startup is launching a mobile game backed by DynamoDB. Traffic is completely unpredictable: it can sit near zero for hours and then jump to tens of thousands of requests per second after a social media mention. The team has no capacity planning experience and wants to avoid throttling while paying only for what it uses. What should the solutions architect configure?

A) Provisioned capacity mode with Auto Scaling configured with a target utilization of 70 percent
B) Provisioned capacity mode with read and write capacity set for the expected peak
C) Provisioned capacity mode and add DynamoDB Accelerator (DAX) to absorb the spikes
D) On-demand capacity mode

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
On-demand capacity mode accommodates sudden traffic changes without capacity planning and charges per request, so idle periods cost almost nothing. It is designed for new tables with unknown or spiky workloads.

**Why option A is wrong:**
Auto Scaling reacts to CloudWatch metrics over minutes, so a sudden jump can exceed the provisioned capacity and cause throttling before scaling catches up. It also still requires tuning.

**Why option B is wrong:**
Provisioning for peak avoids throttling but you pay for that capacity during the long idle periods, which fails the "pay only for what it uses" goal.

**Why option C is wrong:**
DAX is a read cache for DynamoDB and does not reduce write capacity needs. It also adds cost, and it does not remove the need for sizing the table's capacity.

</details>

## Question 9

A gaming company needs a leaderboard and session store with sub-millisecond latency. The leaderboard requires sorted sets, the data must survive a node failure with automatic failover, and the company wants read replicas for scaling. Which solution should be used?

A) Amazon ElastiCache for Memcached with multiple nodes
B) Amazon ElastiCache for Redis with a replication group and Multi-AZ automatic failover
C) Amazon DynamoDB with DynamoDB Accelerator (DAX)
D) A larger Amazon RDS for PostgreSQL instance with Multi-AZ enabled

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
ElastiCache for Redis supports sorted sets natively, which suit leaderboards. Replication groups provide read replicas and Multi-AZ automatic failover, and Redis also supports data persistence options. This satisfies the latency and availability requirements.

**Why option A is wrong:**
Memcached is a simple key-value cache without sorted sets, replication, persistence, or automatic failover. A node failure loses that node's data.

**Why option C is wrong:**
DAX caches DynamoDB reads with microsecond latency, but DynamoDB has no sorted-set data structure, so ranking operations must be built in application code and would be costly.

**Why option D is wrong:**
Even with Multi-AZ, a relational database cannot reach sub-millisecond latency, and sorting thousands of rows on every request would add load to the database.

</details>

## Question 10

A company has an existing, unencrypted Amazon RDS for PostgreSQL instance. A new security policy requires all databases to be encrypted at rest with AWS KMS. The company wants to encrypt this database with minimal downtime. What should the solutions architect do?

A) Modify the DB instance and turn on the encryption option
B) Create a read replica, enable encryption on the read replica, and promote it
C) Take a snapshot, copy it with encryption enabled using a KMS key, restore a new DB instance from the encrypted copy, and cut the application over to it
D) Turn on default EBS encryption for the account and restart the DB instance

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Encryption at rest can only be enabled when a DB instance is created. For an existing instance, the supported path is to snapshot it, copy the snapshot with encryption enabled, and restore a new instance from the encrypted snapshot. To reduce downtime, the application is switched to the new endpoint after the restore (optionally keeping data in sync with AWS DMS).

**Why option A is wrong:**
The encryption setting cannot be changed on an existing unencrypted RDS instance, so the modify operation will not offer it.

**Why option B is wrong:**
A read replica of an unencrypted source must also be unencrypted when replicating within the same Region, so you cannot encrypt it separately and promote it.

**Why option D is wrong:**
Account-level default EBS encryption applies to newly created EBS volumes. It does not convert an existing RDS instance's storage, and a restart does not change that.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
