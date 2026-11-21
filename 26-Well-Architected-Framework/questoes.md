# Review Questions

## Question 1

A scenario requires the right decision on Well-Architected. Which approach fits best?

A) Apply a generic solution without considering pillars.
B) Use pillars with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of well-architected with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take pillars into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine trade-offs with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of well-architected with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of automacao when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of well-architected with the best balance between requirement, risk, and operations.

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
B) The one that ignores confiabilidade signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of well-architected with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on confiabilidade and cost with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of well-architected with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A startup runs its web application and a MySQL database on a single EC2 instance in one Availability Zone. After an AZ disruption, the application was down for four hours. The business now requires the application to survive the loss of a single AZ with minimal downtime. Which design change best follows the Reliability pillar?

A) Move to a larger instance type with more CPU and memory
B) Take nightly EBS snapshots and copy them to Amazon S3
C) Put the instance behind an Elastic IP address and restart it manually after failures
D) Run the application in an Auto Scaling group across at least two AZs behind an Application Load Balancer, and move the database to Amazon RDS Multi-AZ

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Removing single points of failure by distributing the application tier across AZs and using a Multi-AZ database with automatic failover lets the workload survive an AZ loss with little downtime.

**Why option A is wrong:**
A larger instance scales vertically but remains a single point of failure in one AZ.

**Why option B is wrong:**
Backups help recovery but restoring them takes time, so they do not provide fast failover.

**Why option C is wrong:**
An Elastic IP and manual restarts keep the same single instance in a single AZ, and manual recovery is slow.

</details>

## Question 7

A company runs a steady baseline of 40 EC2 instances 24/7 for a core application. Traffic spikes unpredictably to about 100 instances for a few hours per week. The architect must reduce costs without hurting availability. Which approach follows the Cost Optimization pillar?

A) Cover the baseline with a Compute Savings Plan and use Auto Scaling with On-Demand (or Spot where interruption is tolerable) for the spikes
B) Run all 100 instances on Spot Instances all the time
C) Buy 100 three-year Reserved Instances to cover the peak
D) Keep all workloads On-Demand for flexibility

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Commitments give large discounts for the predictable baseline, while elastic capacity matches variable demand and avoids paying for unused peak capacity.

**Why option B is wrong:**
Spot can be interrupted, which puts the core, always-on application at risk.

**Why option C is wrong:**
Committing to peak capacity means paying for 60 unused instances most of the time.

**Why option D is wrong:**
On-Demand for a steady baseline is the most expensive way to run it.

</details>

## Question 8

An application on Amazon EC2 needs to read objects from an S3 bucket. A developer proposes creating an IAM user and storing its access keys in the application's configuration file. Which alternative best aligns with the Security pillar?

A) Store the access keys in the EC2 user data script
B) Make the bucket public and restrict it by IP address
C) Assign an IAM role to the instance through an instance profile with least-privilege S3 permissions
D) Share one IAM user's keys among all instances, and rotate them yearly

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
IAM roles provide temporary credentials that are rotated automatically, so there are no long-term secrets to leak, and permissions can be limited to the needed bucket and actions.

**Why option A is wrong:**
User data is readable from the instance metadata and still uses long-term keys.

**Why option B is wrong:**
A public bucket exposes data broadly, and IP restrictions are a weak control.

**Why option D is wrong:**
Shared, rarely rotated long-term keys increase the blast radius and violate security best practices.

</details>

## Question 9

A team deploys changes by manually clicking through the AWS console. Environments have drifted apart, and a recent change caused an outage that took hours to roll back. Which practice from the Operational Excellence pillar would help most?

A) Keep a detailed runbook in a wiki listing all console steps
B) Define infrastructure with AWS CloudFormation templates in version control and deploy through a CI/CD pipeline with small, reversible changes
C) Allow only one senior engineer to make changes in the console
D) Take screenshots of the console settings before each change

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Infrastructure as code with automated pipelines makes changes repeatable, reviewable and easy to roll back, which reduces drift and human error.

**Why option A is wrong:**
Documentation still depends on manual steps that people can get wrong, and the runbook can drift from reality.

**Why option C is wrong:**
This creates a bottleneck and a single person who may still make manual errors without improving repeatability.

**Why option D is wrong:**
Screenshots are not a reliable rollback mechanism, and they do not prevent drift.

</details>

## Question 10

A company wants to improve the sustainability of its workloads on AWS. Utilization is low because the fleet is sized for a peak that happens once a month. Which TWO actions align with the Sustainability pillar?

A) Keep provisioning capacity for peak load 24/7 to avoid any risk
B) Use Auto Scaling and right-sizing to match capacity to actual demand
C) Replicate all data to every AWS Region
D) Move suitable workloads to AWS Graviton-based instances, which are more energy efficient
E) Disable S3 lifecycle policies and keep all data in the Standard storage class indefinitely

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, D

**Explanation:**
Matching resources to demand reduces idle capacity and energy use, and more efficient processors lower energy consumption for the same work.

**Why option A is wrong:**
Permanently provisioning for peak keeps idle resources running, which wastes energy.

**Why option C is wrong:**
Replicating everywhere multiplies storage and increases the footprint without a requirement.

**Why option E is wrong:**
Keeping rarely used data in Standard indefinitely wastes storage resources, whereas lifecycle policies reduce them.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> If your goal is to connect AWS architecture with ingestion, processing, and analytics, the complementary track is: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
