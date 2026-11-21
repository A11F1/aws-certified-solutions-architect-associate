# Review Questions

## Question 1

A scenario requires the right decision on introduction. Which approach fits best?

A) Apply a generic solution without considering dominant requirement.
B) Use dominant requirement with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of introduction with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take dominant requirement into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine trade-off with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of introduction with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of baixo esforco operacional when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of introduction with the best balance between requirement, risk, and operations.

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
B) The one that ignores eliminacao de alternativas signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of introduction with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on eliminacao de alternativas and caderno de erros with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of introduction with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A financial services company is preparing for an external audit. The auditors have asked for AWS's own SOC 1, SOC 2, and ISO 27001 compliance reports so they can evaluate the controls AWS operates in its data centers. The security team wants to download these documents on demand without opening a support case. Which AWS service should the team use?

A) AWS Config
B) AWS Trusted Advisor
C) AWS Artifact
D) Amazon Inspector

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
AWS Artifact is the self-service portal for downloading AWS compliance reports (SOC, ISO, PCI attestations) and for accepting agreements such as the BAA. It gives auditors evidence about the controls AWS is responsible for under the shared responsibility model.

**Why option A is wrong:**
AWS Config records and evaluates the configuration of your own resources against rules. It does not provide AWS's third-party audit reports.

**Why option B is wrong:**
Trusted Advisor gives recommendations on cost, performance, security, fault tolerance, and service limits for your account. It does not distribute compliance reports.

**Why option D is wrong:**
Amazon Inspector scans your EC2 instances, container images, and Lambda functions for software vulnerabilities. It says nothing about the controls AWS runs in its facilities.

</details>

## Question 7

A healthcare startup must keep all patient data inside Germany to meet data residency rules. Its users are mostly in Europe, and it wants to prevent engineers from accidentally launching resources in other Regions. Which approach satisfies both requirements with the least ongoing effort?

A) Deploy to the Europe (Frankfurt) Region and apply a Service Control Policy in AWS Organizations that denies actions outside eu-central-1
B) Deploy to any Region and configure a CloudFront geographic restriction that allows only German viewers
C) Deploy to the Europe (Frankfurt) Region and rely on edge locations in Germany to store the primary copy of the data
D) Deploy to any Region and use consolidated billing with cost allocation tags to flag resources created outside Germany

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Data never leaves a Region unless you move it, so choosing Frankfurt keeps the data in Germany. An SCP that denies requests outside the approved Region is a preventive guardrail that applies to every account in the organization, so engineers cannot launch resources elsewhere.

**Why option B is wrong:**
A CloudFront geographic restriction controls which viewers can fetch content. It has no effect on where the origin data is stored or where resources can be created.

**Why option C is wrong:**
Edge locations cache content for low-latency delivery and are not a place to store the primary copy of application data. Durable storage lives in services such as S3 or RDS inside a Region.

**Why option D is wrong:**
Tags and billing views are detective at best and only show up after the resources already exist. They neither enforce data residency nor block launches in other Regions.

</details>

## Question 8

An e-commerce company runs a checkout service on EC2 behind an Application Load Balancer. Leadership wants to prove that the system can recover from failures, so the team schedules regular game days. During these exercises they terminate instances at random, fail over the database, and restore backups into a test environment to verify recovery time. Which Well-Architected Framework pillar does this practice mainly support?

A) Operational Excellence
B) Performance Efficiency
C) Security
D) Reliability

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
The Reliability pillar includes the design principles of testing recovery procedures, automatically recovering from failure, and validating backups. Game days that inject failures and measure recovery are a textbook example.

**Why option A is wrong:**
Operational Excellence focuses on running and monitoring workloads, managing change, and learning from operational events. It overlaps somewhat, but the specific goal of verifying recovery from failure belongs to Reliability.

**Why option B is wrong:**
Performance Efficiency is about selecting the right resource types and sizes and keeping them efficient as demand changes. Failure recovery testing is not its goal.

**Why option C is wrong:**
The Security pillar covers protecting data, systems, and identities. The exercises described target availability and recovery, not confidentiality or access control.

</details>

## Question 9

A media company hosts its application in a single Availability Zone in one Region. After an AZ-level power event caused two hours of downtime, management asks for a design that survives the loss of an entire data center while keeping latency between the web, application, and database tiers low and avoiding the cost of a second Region. What should the architect recommend?

A) Build an active-active deployment in two Regions and synchronize data with custom scripts
B) Spread the tiers across at least two Availability Zones in the same Region
C) Keep a single Availability Zone but move to larger instance types with more redundant hardware
D) Serve the application from CloudFront edge locations so the origin AZ no longer matters

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Availability Zones are physically separate facilities connected by low-latency links inside a Region. Distributing the tiers across multiple AZs protects against a data center failure without the cost and complexity of a second Region.

**Why option A is wrong:**
A second Region would survive an AZ failure, but it adds cross-Region latency, data transfer costs, and operational complexity the requirements specifically want to avoid.

**Why option C is wrong:**
Larger instances do not protect against the loss of the facility. A single AZ remains a single point of failure regardless of instance size.

**Why option D is wrong:**
CloudFront can cache static or cacheable content, but dynamic requests and writes still reach the origin. If the only origin AZ is down, the application still fails.

</details>

## Question 10

A company runs a web application on EC2 instances and stores user uploads in Amazon S3. During a security review, an engineer lists what the company must handle itself under the AWS shared responsibility model. Which TWO of the following are the customer's responsibility?

A) Patching and updating the guest operating system on the EC2 instances
B) Patching the hypervisor that runs the EC2 instances
C) Replacing failed disks in the S3 storage infrastructure
D) Configuring security group rules and S3 bucket policies
E) Maintaining the physical network equipment in the AWS Region

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A, D

**Explanation:**
For EC2 (infrastructure as a service) the customer manages the guest OS, applications, and access configuration such as security groups and bucket policies. AWS is responsible for security of the cloud: hardware, virtualization layer, and facilities.

**Why option B is wrong:**
The hypervisor is part of the virtualization layer, which AWS operates and patches.

**Why option C is wrong:**
S3 is a managed service and AWS maintains the underlying storage hardware and durability. Customers never touch the disks.

**Why option E is wrong:**
Physical networking equipment in data centers and the Regional backbone is operated by AWS as part of the global infrastructure.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> To turn this architecture foundation into practice with pipelines, data, and analytics, you can also follow the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
