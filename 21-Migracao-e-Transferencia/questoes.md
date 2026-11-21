# Review Questions

## Question 1

A scenario requires the right decision on migration. Which approach fits best?

A) Apply a generic solution without considering DMS.
B) Use DMS with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of migration with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take DMS into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine SCT with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of migration with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of DataSync when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of migration with the best balance between requirement, risk, and operations.

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
B) The one that ignores Snowball signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of migration with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on Snowball and cutover with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of migration with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A media company must move 80 TB of archived footage from its data center to Amazon S3 within two weeks. The office has a 100 Mbps internet link that is already used by daily operations, and a Direct Connect link cannot be provisioned in that time. Which approach meets the deadline?

A) Use AWS DataSync over the existing internet connection
B) Enable S3 Transfer Acceleration and upload with multipart uploads
C) Order AWS Snowball Edge Storage Optimized devices, load the data locally, and ship them back to AWS
D) Order an AWS Direct Connect connection and transfer the data over it

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Transferring 80 TB over 100 Mbps would take roughly 70 days even with the link fully dedicated. Snowball Edge devices hold tens of terabytes each, are shipped to the site, and the data is imported into S3 after the device returns. Option C is the standard answer when data volume is large and bandwidth is limited.

**Why option A is wrong:**
DataSync is efficient and reliable, but it is still limited by the 100 Mbps link, so the transfer would take far longer than two weeks.

**Why option B is wrong:**
Transfer Acceleration optimizes the path to S3 over long distances, but it cannot exceed the capacity of the local internet link, which is the bottleneck here.

**Why option D is wrong:**
Direct Connect takes weeks to months to provision, and the scenario already rules it out. Even once available, a new circuit is not required for a one-time bulk move.

</details>

## Question 7

A company keeps 5 TB of shared data on an on-premises NFS server and wants a nightly incremental copy to Amazon EFS. The transfer must be encrypted in transit, verify data integrity, and require minimal scripting. Which solution is the most appropriate?

A) Use AWS DataSync with an on-premises agent and a scheduled task from the NFS location to EFS
B) Write a nightly rsync cron job that copies data to an EC2 instance mounting EFS
C) Order an AWS Snowball Edge device every night
D) Set up AWS Transfer Family with an SFTP endpoint and have the NFS server push files to it

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
DataSync is built for online transfers between on-premises storage such as NFS and AWS storage including EFS, S3, and FSx. It encrypts data in transit, validates integrity, copies only changed data, and supports task scheduling. Option A satisfies the requirements without custom scripts.

**Why option B is wrong:**
A hand-built rsync job on an EC2 instance needs scripting, monitoring, and the instance itself, and offers no built-in integrity verification or managed retry logic.

**Why option C is wrong:**
Snowball Edge is for offline bulk transfers. Shipping a device every night for incremental changes is impractical and slow.

**Why option D is wrong:**
Transfer Family provides managed SFTP, FTPS, and FTP endpoints into S3 or EFS for external partners. It is a server, not a scheduled NFS synchronization tool, and the NFS server would need custom scripting to push files.

</details>

## Question 8

A company wants to move a 3 TB Oracle database to Amazon Aurora PostgreSQL to reduce licensing costs. The application can only tolerate a few minutes of downtime at cutover. Stored procedures and schema objects must also be converted. Which combination should the architect use?

A) AWS DMS alone, because it converts schema objects and code automatically
B) Export the Oracle database to a dump file and import it into Aurora PostgreSQL with pg_restore
C) Copy the Oracle data files to S3 using Snowball Edge and restore them into Aurora
D) AWS Schema Conversion Tool to convert the schema and code, then AWS DMS with ongoing change data capture

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Heterogeneous migrations such as Oracle to PostgreSQL need two steps: the Schema Conversion Tool (or DMS Schema Conversion) translates schemas and code objects, and DMS migrates the data and uses change data capture to keep the target in sync. The cutover then takes only minutes. Option D covers both needs.

**Why option A is wrong:**
DMS migrates data but does not fully convert schemas and procedural code between different engines. The conversion tooling is required first.

**Why option B is wrong:**
Oracle dump files are in a proprietary format that PostgreSQL's pg_restore cannot read, and an offline export and import would cause hours of downtime.

**Why option C is wrong:**
Oracle data files are engine-specific and cannot be restored into Aurora PostgreSQL. A physical copy does not change the database engine.

</details>

## Question 9

Several business partners upload files to the company every day using SFTP clients and existing credentials. The company is retiring its self-managed SFTP servers, wants the files to land in Amazon S3, and does not want partners to change their clients or workflows. Which solution meets these requirements with the least operational effort?

A) Run SFTP software on an EC2 instance with an EBS volume and a script that copies files to S3
B) Create an AWS Transfer Family SFTP server with an S3 bucket as the storage backend
C) Have partners upload with S3 presigned URLs generated by a Lambda function
D) Use AWS DataSync to receive the files from the partners over SFTP

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
AWS Transfer Family provides a fully managed SFTP endpoint that stores files directly in S3 and supports existing identity providers or service-managed users. Partners keep using the same protocol and clients. Option B retires the servers without client changes.

**Why option A is wrong:**
This replicates the existing self-managed model: the company would still patch, scale, and monitor the server and maintain the copy script.

**Why option C is wrong:**
Presigned URLs use HTTPS, so every partner would have to rewrite their upload process, which violates the requirement.

**Why option D is wrong:**
DataSync moves data between storage systems using an agent. It does not act as an SFTP server that accepts uploads from external clients.

</details>

## Question 10

A company plans to rehost 200 on-premises VMware virtual machines on EC2 with no application changes. The migration team wants continuous block-level replication so that each cutover requires only a short downtime, plus the ability to run non-disruptive test launches before cutover. Which service should be used?

A) AWS Application Migration Service
B) Export each VM as an OVA file and import it using VM Import/Export
C) AWS Database Migration Service
D) Copy the VM disk files to Snowball Edge and ship them to AWS

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
AWS Application Migration Service (MGN) installs an agent on the source servers and continuously replicates their disks to a staging area in AWS. You can launch test instances without disrupting the source, then perform a cutover with minimal downtime. Option A is the recommended lift-and-shift service.

**Why option B is wrong:**
Exporting and importing images is a manual, per-VM process that is a point-in-time copy. The source must stay frozen or changes are lost, causing long downtime, and it does not scale to 200 servers.

**Why option C is wrong:**
DMS migrates databases, not entire servers and operating systems.

**Why option D is wrong:**
Shipping disk files offline gives no continuous replication, so any changes after the copy would be lost or require long outages, and the files would still need conversion into bootable images.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> If your goal is to connect AWS architecture with ingestion, processing, and analytics, the complementary track is: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
