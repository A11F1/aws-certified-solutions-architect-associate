# Review Questions

## Question 1

A scenario requires the right decision on analytics. Which approach fits best?

A) Apply a generic solution without considering Athena.
B) Use Athena with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of analytics with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take Athena into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine Glue with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of analytics with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of Redshift when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of analytics with the best balance between requirement, risk, and operations.

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
B) The one that ignores Kinesis signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of analytics with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on particionamento and Parquet with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of analytics with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company keeps several terabytes of application logs as JSON files in Amazon S3. Analysts run ad hoc SQL queries a few times a week and want to pay only when they run queries, without provisioning or managing any servers or clusters. Which solution is the best fit?

A) Load the logs into a provisioned Amazon Redshift cluster and keep it running
B) Define the logs in the AWS Glue Data Catalog and query them with Amazon Athena
C) Launch an Amazon EMR cluster with Hive and keep it running for analysts
D) Import the logs into an Amazon RDS for PostgreSQL instance

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Athena is serverless and queries data in place in S3 using standard SQL, with pricing based on the amount of data scanned. With the schema in the Glue Data Catalog, there is nothing to provision and nothing to pay for between queries.

**Why option A is wrong:**
A provisioned Redshift cluster bills while it runs and requires loading data first, which is wasteful for infrequent queries.

**Why option C is wrong:**
A running EMR cluster incurs cost for the entire time it is up and must be managed, which goes against the no-servers requirement.

**Why option D is wrong:**
RDS needs an always-on instance with storage sized for the data, and it is a transactional database not designed for scanning terabytes of logs.

</details>

## Question 7

A gaming company collects clickstream events at about 50,000 events per second. Several independent applications need to read the same events in near real time, and one of them must be able to replay the last 7 days of events after a bug fix. Which service should the architect use for ingestion?

A) Amazon Kinesis Data Streams with the retention period increased to 7 days
B) Amazon Kinesis Data Firehose delivering to S3
C) Amazon SQS standard queue shared by all applications
D) Amazon SNS topic with an HTTPS subscription per application

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Kinesis Data Streams lets multiple consumers read the same records independently, and retention can be extended from the 24-hour default up to 365 days. Consumers can re-read data from an earlier position, which supports replay.

**Why option B is wrong:**
Firehose is a delivery service that buffers and writes data to destinations such as S3. It does not offer multiple independent consumers or replay of records from a stream.

**Why option C is wrong:**
In SQS a message is consumed by one reader and then deleted, so several applications cannot each read the same events, and there is no replay of deleted messages.

**Why option D is wrong:**
SNS pushes messages to subscribers but does not store them for replay. Subscribers that miss a message cannot go back and read it again.

</details>

## Question 8

A retail company streams point-of-sale data and wants to land it in S3 as compressed Apache Parquet files so analysts can query it with Athena. The team wants to write as little code as possible and does not want to manage servers. Which solution should be used?

A) Kinesis Data Streams with a KCL consumer on EC2 instances that writes Parquet files
B) A Lambda function triggered per record that writes each record as a separate S3 object
C) An AWS Glue crawler that runs every minute on the streaming source
D) Amazon Kinesis Data Firehose with record format conversion to Parquet using a Glue Data Catalog table

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Firehose is fully managed, buffers incoming records, converts JSON to Parquet or ORC using a schema from the Glue Data Catalog, and delivers batched files to S3. It needs no custom code and no servers.

**Why option A is wrong:**
This works technically but requires writing and operating consumer code and the EC2 fleet, which the team wants to avoid.

**Why option B is wrong:**
Writing one object per record creates millions of tiny files, which hurts Athena performance and cost, and it does not produce Parquet on its own.

**Why option C is wrong:**
A Glue crawler discovers schemas of data already stored. It does not ingest, convert, or deliver streaming data.

</details>

## Question 9

A company wants to run complex SQL joins and aggregations across 200 TB of structured sales data from several sources. Dozens of analysts and BI dashboards query it concurrently, and the company wants columnar storage and massively parallel processing. Which service is the best choice?

A) Amazon Aurora MySQL with several read replicas
B) Amazon DynamoDB with on-demand capacity
C) Amazon Redshift
D) Amazon OpenSearch Service

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Redshift is a managed data warehouse that uses columnar storage and massively parallel processing, built for complex analytical SQL over very large structured datasets with many concurrent BI users.

**Why option A is wrong:**
Aurora is a row-oriented OLTP database. It is not optimized for scanning and aggregating hundreds of terabytes, and read replicas do not change that.

**Why option B is wrong:**
DynamoDB is a key-value and document store optimized for single-item access patterns. It does not support the complex joins and ad hoc aggregations required.

**Why option D is wrong:**
OpenSearch is designed for search and log analytics over indexed documents, not as a relational data warehouse for complex joins.

</details>

## Question 10

A company stores 5 TB of daily CSV files in S3 in a single prefix. Athena queries that filter on a single day and read only a few columns are slow and expensive because every query scans the whole dataset. Which TWO changes will most reduce the data scanned and the cost?

A) Move the data to the S3 Glacier Flexible Retrieval storage class
B) Convert the data to a columnar format such as Apache Parquet
C) Run SELECT * and filter the needed columns in the application
D) Partition the data by date and filter on the partition column in queries
E) Increase the number of nodes in the Athena cluster

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, D

**Explanation:**
Athena charges by the amount of data scanned. Columnar Parquet lets it read only the requested columns and compresses well, and date partitioning lets it skip every partition that does not match the filter. Together they cut the scanned data dramatically.

**Why option A is wrong:**
Athena cannot query objects in S3 Glacier Flexible Retrieval until they are restored, so this would break the queries rather than reduce scanning.

**Why option C is wrong:**
SELECT * reads every column from every file, which maximizes the data scanned. Filtering afterward in the application does not reduce the cost.

**Why option E is wrong:**
Athena is serverless and has no nodes to add. Cost depends on the data scanned, not on cluster size.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
