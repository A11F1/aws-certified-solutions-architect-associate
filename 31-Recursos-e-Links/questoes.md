# Review Questions

## Question 1

A scenario requires the right decision on resources. Which approach fits best?

A) Apply a generic solution without considering official documentation.
B) Use official documentation with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resources with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take official documentation into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine annotated practice exams with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resources with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of curadoria when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resources with the best balance between requirement, risk, and operations.

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
B) The one that ignores escopo SAA-C03 signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resources with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on curadoria and escopo SAA-C03 with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of resources with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

An exam question asks for the solution with the LEAST operational overhead to process images uploaded to Amazon S3 at unpredictable times. Option 1 uses an EC2 Auto Scaling group with a custom polling script. Option 2 uses S3 event notifications to invoke AWS Lambda. Both would work technically.

Which test-taking approach is BEST?

A) Choose the option that uses more services, because it is probably more resilient
B) Choose the first option that works, because the exam rewards speed
C) Choose the lowest-cost option and ignore the operational wording
D) Treat the phrase about operational overhead as a hard constraint and choose the managed, event-driven option that satisfies every requirement

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
SAA-C03 questions usually have several technically valid answers, and qualifiers such as least operational overhead, most cost-effective, or highest availability decide between them. The managed, serverless option removes server management, so it fits the stated constraint.

**Why option A is wrong:**
More services usually mean more complexity and more to operate, which works against the overhead requirement. Extra components are a common sign of a distractor.

**Why option B is wrong:**
Several options may work, so stopping at the first one risks missing the qualifier that makes another option the best answer.

**Why option C is wrong:**
Cost is only one possible qualifier. This stem asks about operational overhead, and a cheaper option that needs servers managed would not satisfy it.

</details>

## Question 7

A candidate preparing for the AWS Certified Solutions Architect - Associate (SAA-C03) exam wants to know the four exam domains and how heavily each one is weighted so that study time can be allocated accordingly.

Which resource should the candidate consult?

A) The official AWS exam guide for the certification
B) The AWS Well-Architected Tool in the console
C) AWS Trusted Advisor in the console
D) The AWS Pricing Calculator

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
The official exam guide defines the domains (Design Secure Architectures, Design Resilient Architectures, Design High-Performing Architectures, and Design Cost-Optimized Architectures), their weightings, and the in-scope services and tasks. It is the authoritative source for planning study time.

**Why option B is wrong:**
The Well-Architected Tool reviews real workloads against the framework's pillars. It does not describe the exam's content or weightings.

**Why option C is wrong:**
Trusted Advisor inspects an AWS account and recommends improvements for cost, security, and performance. It has no information about the certification exam.

**Why option D is wrong:**
The Pricing Calculator estimates the cost of AWS services for a workload. It does not provide any exam information.

</details>

## Question 8

A team wants to review its production workload against AWS best practices, record answers to structured questions for each pillar, and receive a prioritized list of high-risk issues together with improvement guidance that can be tracked over time.

Which AWS resource should the team use?

A) AWS Trusted Advisor, because it reviews the workload's architecture design against the framework's pillars
B) Amazon Inspector, because it audits the design of the workload
C) The AWS Well-Architected Tool, by defining the workload and answering its pillar-based questions
D) AWS Config, because it scores the architecture on the six pillars

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
The AWS Well-Architected Tool guides a structured review of a workload against the Well-Architected Framework pillars. It identifies high and medium risks, suggests improvement items, and keeps milestones so progress can be tracked.

**Why option A is wrong:**
Trusted Advisor runs automated checks on account-level configuration and usage, covering areas such as cost, security, and service quotas. It does not walk a team through a question-based architecture review.

**Why option B is wrong:**
Amazon Inspector scans workloads such as EC2 instances, container images, and Lambda functions for software vulnerabilities and network exposure. It does not assess architecture design.

**Why option D is wrong:**
AWS Config records resource configurations and evaluates them against rules. It does not provide a pillar-based architecture review or scoring.

</details>

## Question 9

During the exam, a candidate reads a question that asks for a relational database that fails over automatically with minimal administration. The options are Amazon DynamoDB global tables, Amazon RDS in a Multi-AZ deployment, MySQL on EC2 with manual replication, and Amazon Redshift.

Which technique BEST narrows the choices to the right answer?

A) Eliminate options whose service names are unfamiliar and choose from what remains
B) Identify the hard constraints in the stem (relational, automatic failover, minimal administration) and eliminate every option that violates one
C) Choose the option with the most advanced-sounding features, since the exam favors them
D) Choose the option that offers the highest scalability, because scalability is always the priority

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
The stem's explicit constraints rule out options one by one. DynamoDB is not relational, MySQL on EC2 with manual replication is neither automatic nor low-administration, and Redshift is a data warehouse, so RDS Multi-AZ remains.

**Why option A is wrong:**
Unfamiliar names are not wrong answers by definition, and this approach would not apply the requirements. The correct choice could be a service the candidate happens to know less well.

**Why option C is wrong:**
Feature richness is not a criterion. The exam rewards the option that fits the stated requirements, and a more advanced service can be the wrong workload fit.

**Why option D is wrong:**
Scalability is not mentioned in the stem. Optimizing for an attribute the question does not request ignores the actual constraints.

</details>

## Question 10

A candidate practices labs in a personal AWS account and wants to be warned if spending approaches a small monthly limit, so that forgotten resources do not create a surprise bill.

What should the candidate do?

A) Create an AWS Budgets cost budget with email alerts at chosen thresholds, and delete lab resources after each session
B) Use the AWS Pricing Calculator, which automatically stops resources when the estimate is exceeded
C) Check AWS Cost Explorer once at the end of the month to review the spending
D) Enable AWS Shield Advanced to block charges from unexpected usage

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
AWS Budgets can send alerts when actual or forecasted costs cross a threshold, which gives an early warning. Cleaning up resources after each lab remains necessary because a budget alert notifies but does not stop resources by itself.

**Why option B is wrong:**
The Pricing Calculator only produces estimates before deployment. It is not connected to the account and cannot monitor or stop anything.

**Why option C is wrong:**
Cost Explorer is useful for analysis, but a single check at the end of the month comes after the charges have already been incurred. It does not alert in advance.

**Why option D is wrong:**
Shield Advanced is a paid DDoS protection service and has a significant monthly fee. It does not control or limit spending.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> When you want to take this topic into data, lakehouse, and hands-on project scenarios, use as your next step: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
