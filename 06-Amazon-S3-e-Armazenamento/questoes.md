# Review Questions

## Question 1

A scenario requires the right decision on S3. Which approach fits best?

A) Apply a generic solution without considering lifecycle.
B) Use lifecycle with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of S3 with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take lifecycle into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine replication with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of S3 with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of storage classes when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of S3 with the best balance between requirement, risk, and operations.

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
B) The one that ignores versioning signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of S3 with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on acesso publico and durabilidade with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of S3 with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A financial services company must store trade confirmations in Amazon S3 for 7 years to meet a regulatory WORM (write once, read many) requirement. Auditors require that no user, including the AWS account root user, can delete or overwrite the objects during the retention period. Which solution meets this requirement?

A) Enable S3 Object Lock in governance mode with a 7-year retention period and restrict the s3:BypassGovernanceRetention permission to administrators
B) Enable S3 Versioning and MFA Delete on the bucket, and give the root user's MFA device to the compliance officer
C) Enable S3 Object Lock in compliance mode with a 7-year retention period
D) Add a bucket policy that denies s3:DeleteObject and s3:PutObject for all principals, and enable AWS CloudTrail data events

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Object Lock in compliance mode prevents an object version from being overwritten or deleted by any user, including the root user, until the retention period expires. The retention mode and period cannot be shortened or removed during that time. This is the only option that provides a true WORM guarantee that satisfies the auditors.

**Why option A is wrong:**
Governance mode protects objects from most users, but any principal with the s3:BypassGovernanceRetention permission (and the bypass header) can still delete or alter the objects, so it does not meet the "no user, including root" requirement.

**Why option B is wrong:**
MFA Delete only requires an MFA code to permanently delete a version or change the versioning state. Whoever holds the MFA device can still delete data, so it is not immutable storage.

**Why option D is wrong:**
A bucket policy can be edited or removed by an administrator (or the root user), after which deletes become possible again. CloudTrail only records activity and does not prevent it.

</details>

## Question 7

An application writes about 2 TB of log files per month to Amazon S3. The logs are read frequently during the first 30 days for troubleshooting, then rarely accessed, but when they are needed they must be available within milliseconds. Logs must be kept for 1 year, then deleted, and the data must survive the loss of an Availability Zone. Which approach is the MOST cost-effective?

A) Create a lifecycle rule that transitions objects to S3 Standard-IA after 30 days and expires them after 365 days
B) Create a lifecycle rule that transitions objects to S3 Glacier Flexible Retrieval after 30 days and expires them after 365 days
C) Create a lifecycle rule that transitions objects to S3 Glacier Deep Archive after 30 days and expires them after 365 days
D) Create a lifecycle rule that transitions objects to S3 One Zone-IA after 30 days and expires them after 365 days

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
S3 Standard-IA offers lower storage prices for infrequently accessed data while still providing millisecond access and storing data across multiple Availability Zones. A lifecycle expiration action then deletes the objects after 365 days without any manual work. Objects must stay in S3 Standard for at least 30 days before moving to Standard-IA, which matches this access pattern.

**Why option B is wrong:**
Glacier Flexible Retrieval requires a restore operation that takes minutes to hours (expedited retrievals take 1 to 5 minutes), so it cannot provide millisecond access during an incident.

**Why option C is wrong:**
Glacier Deep Archive is the cheapest class but retrieval takes 12 hours or more, which is far from the millisecond requirement.

**Why option D is wrong:**
One Zone-IA stores data in a single Availability Zone, so an AZ failure could destroy the logs. That violates the explicit resilience requirement.

</details>

## Question 8

A media company has users around the world who upload 5 GB video files directly to a single S3 bucket in us-east-1. Users in Asia and Australia report slow and frequently failing uploads over long-distance internet paths. The company wants to improve upload speed and reliability with minimal changes to the application. What should a solutions architect recommend?

A) Enable S3 Cross-Region Replication to buckets in Singapore and Sydney and let users upload to the nearest bucket
B) Spread objects across many random key prefixes to increase the request rate per prefix
C) Use S3 Select on the destination bucket to speed up the ingestion of large objects
D) Enable S3 Transfer Acceleration on the bucket and have the application use the accelerated endpoint with multipart upload

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
S3 Transfer Acceleration routes uploads to the nearest CloudFront edge location, which then forwards the data over the optimized AWS backbone to the bucket. Combined with multipart upload, large files are split into parts that upload in parallel and can be retried individually. The only application change is using the accelerated endpoint.

**Why option A is wrong:**
Cross-Region Replication copies objects after they have been written to a source bucket. Users would also need logic to choose a bucket, and replication does not make the upload from the user faster.

**Why option B is wrong:**
Prefix design helps with very high request rates per prefix (thousands of requests per second). The problem here is network distance and path quality, not request throttling.

**Why option C is wrong:**
S3 Select retrieves a subset of data from an object using SQL expressions. It is a read feature and has no effect on upload performance.

</details>

## Question 9

A healthcare company stores patient documents in S3. Security policy requires that the objects be encrypted at rest with keys the company controls, that every use of the key be auditable in AWS CloudTrail, and that uploads without the right encryption be rejected. Which solution meets these requirements with the LEAST operational overhead?

A) Use SSE-S3 and enable default encryption on the bucket, then review S3 server access logs for key usage
B) Use SSE-KMS with a customer managed key, and add a bucket policy that denies s3:PutObject requests that do not specify aws:kms server-side encryption
C) Use SSE-C and have the application send the encryption key with every upload and download request
D) Encrypt the files on the application servers with a library before upload, and store the data key in an application configuration file

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
SSE-KMS with a customer managed key lets the company control the key policy, rotation, and permissions, and every use of the key is logged in CloudTrail. A bucket policy with a condition on the s3:x-amz-server-side-encryption header rejects uploads that do not use KMS. S3 handles the encryption itself, so the operational burden is low.

**Why option A is wrong:**
SSE-S3 keys are owned and managed by S3, so the company has no control over them and no per-key CloudTrail audit trail. Server access logs record requests to objects, not key usage.

**Why option C is wrong:**
With SSE-C the client supplies the key on every request and AWS does not store it. This pushes key management to the company, increases overhead, and provides no KMS-style key audit trail.

**Why option D is wrong:**
Client-side encryption with a key stored in a configuration file is insecure, creates heavy key management work, and gives no CloudTrail audit of key usage.

</details>

## Question 10

A company keeps customer invoices in an S3 bucket in eu-west-1. To meet a disaster recovery requirement, all newly uploaded objects must be copied automatically to a bucket in us-east-1. Which TWO actions are required to configure this? (Choose two.)

A) Enable S3 Transfer Acceleration on both buckets
B) Enable versioning on both the source and destination buckets
C) Enable S3 Object Lock on the destination bucket
D) Create a replication rule on the source bucket that uses an IAM role S3 can assume to replicate objects
E) Configure an S3 event notification that invokes a Lambda function to copy each object to the destination bucket

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, D

**Explanation:**
S3 replication requires versioning to be enabled on both the source and the destination buckets. The replication rule on the source bucket specifies the destination and an IAM role that grants S3 permission to read from the source and write to the destination. Once configured, new objects are replicated asynchronously without any custom code.

**Why option A is wrong:**
Transfer Acceleration speeds up client uploads over long distances. It is not used or required for replication between buckets.

**Why option C is wrong:**
Object Lock is a retention feature and is not a prerequisite for replication. Enabling it on the destination is not required to copy objects.

**Why option E is wrong:**
A Lambda copy function could work, but it is custom code to build, monitor, and retry. It is not the required configuration for the built-in replication feature and adds unnecessary operational overhead.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> If your goal is to connect AWS architecture with ingestion, processing, and analytics, the complementary track is: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
