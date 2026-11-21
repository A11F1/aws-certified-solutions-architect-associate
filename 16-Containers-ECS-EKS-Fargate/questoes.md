# Review Questions

## Question 1

A scenario requires the right decision on containers. Which approach fits best?

A) Apply a generic solution without considering ECS.
B) Use ECS with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of containers with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take ECS into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine EKS with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of containers with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of Fargate when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of containers with the best balance between requirement, risk, and operations.

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
B) The one that ignores task definition signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of containers with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on task definition and service autoscaling with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of containers with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A startup is moving a set of stateless microservices, packaged as Docker containers, to AWS. They need an Application Load Balancer in front, automatic scaling, and as little infrastructure management as possible. The team does not want to patch, size, or scale any EC2 instances. Which solution meets these requirements?

A) Amazon ECS with the EC2 launch type and an Auto Scaling group of container instances
B) Amazon ECS with the Fargate launch type behind an Application Load Balancer
C) Amazon EKS with self-managed EC2 worker nodes
D) AWS Lambda functions invoked by the load balancer, one per microservice

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Fargate is a serverless compute engine for containers, so there are no instances to patch, size, or scale. ECS services on Fargate integrate directly with an Application Load Balancer and can scale the task count automatically.

**Why option A is wrong:**
With the EC2 launch type the team still owns the container instances: AMI patching, capacity planning, and Auto Scaling group management.

**Why option C is wrong:**
Self-managed worker nodes in EKS mean even more operational work, including node patching and upgrades, plus the Kubernetes control-plane concepts the team does not need.

**Why option D is wrong:**
Lambda can run container images, but it is a different programming model with a 15-minute execution limit and is not a drop-in way to run long-lived container services behind a load balancer.

</details>

## Question 7

A company already runs its applications on Kubernetes on premises, with Helm charts and custom operators. It wants to move to AWS while keeping the Kubernetes API for portability and avoiding the operational burden of running its own control plane. Which service should the architect recommend?

A) Amazon ECS with the Fargate launch type, since it is fully managed
B) Kubernetes installed with kops on a fleet of EC2 instances
C) AWS App Runner with a container image from Amazon ECR
D) Amazon EKS, which provides a managed Kubernetes control plane

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Amazon EKS runs a managed, conformant Kubernetes control plane across multiple Availability Zones. Existing Helm charts, operators, and kubectl workflows carry over, and the company avoids operating etcd and the API servers.

**Why option A is wrong:**
ECS is an AWS-specific orchestrator with its own API and task definitions. Moving to it would mean rewriting manifests and abandoning the Kubernetes tooling the company wants to keep.

**Why option B is wrong:**
Running Kubernetes yourself on EC2 keeps compatibility but puts the control plane, upgrades, and high availability back on the team, which is exactly the burden they want to avoid.

**Why option C is wrong:**
App Runner is a simplified service for web apps and APIs. It does not expose the Kubernetes API, Helm, or custom operators.

</details>

## Question 8

Several services share one ECS cluster that runs on EC2 container instances. The orders service must read and write one DynamoDB table, while the reports service must only read an S3 bucket. The security team requires least privilege for each service and no long-term credentials in images or environment variables. What should the architect do?

A) Create a separate IAM role for each task definition and assign it as the task role
B) Attach one IAM role with all permissions to the EC2 instance profile that all tasks share
C) Store IAM user access keys for each service in environment variables of the task definition
D) Add the DynamoDB and S3 permissions to the ECS task execution role and use it for all tasks

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
ECS task roles supply temporary credentials scoped to a single task definition. Each service can therefore get only the permissions it needs, with credentials delivered automatically and rotated by AWS.

**Why option B is wrong:**
An instance profile is shared by every task on that instance, so each container would have the combined permissions of all services, which violates least privilege.

**Why option C is wrong:**
Long-term access keys in environment variables are exactly what the security team prohibits, and they are hard to rotate and easy to leak.

**Why option D is wrong:**
The task execution role is used by the ECS agent to pull images from ECR, write logs, and fetch secrets. It is not meant for application permissions, and sharing it would again give every task the same access.

</details>

## Question 9

A media company runs containerized transcoding workers as Fargate tasks across multiple Availability Zones. All workers must read and write the same set of working files through a POSIX-compliant file system, and the data must persist after tasks stop. Which storage choice meets this?

A) An Amazon EBS volume attached to every task
B) The ephemeral storage that comes with each Fargate task
C) An Amazon EFS file system mounted into the task definition
D) An Amazon S3 bucket mounted as a local POSIX file system by each task

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Amazon EFS is a shared, elastic, POSIX-compliant file system that spans multiple Availability Zones, and Fargate tasks can mount it directly. Data persists independently of the task lifecycle and is visible to all tasks at once.

**Why option A is wrong:**
An EBS volume lives in a single Availability Zone and attaches to one task at a time, so it cannot be shared by workers across zones.

**Why option B is wrong:**
Fargate ephemeral storage is private to each task and is deleted when the task stops, so it is neither shared nor persistent.

**Why option D is wrong:**
S3 is object storage, not a POSIX file system. Tasks would need to use the S3 API, and Fargate does not natively mount an S3 bucket as a file system for this use case.

</details>

## Question 10

A company runs an ECS cluster. An always-on order API has a steady baseline load around the clock, and a nightly image-processing batch job is fault tolerant and can be restarted if interrupted. The company wants to cut compute costs without risking the availability of the API. Which TWO actions should the architect take?

A) Run the order API on Spot capacity because it is the largest workload
B) Purchase a Compute Savings Plan to cover the steady baseline usage
C) Buy Standard Reserved Instances and expect them to discount Fargate task usage
D) Run the interruption-tolerant nightly batch on Spot capacity, such as Fargate Spot
E) Move all workloads to Dedicated Hosts to get volume discounts

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, D

**Explanation:**
A Compute Savings Plan gives a discount on predictable baseline usage and applies to EC2, Fargate, and Lambda. Spot capacity offers deep discounts for workloads that tolerate interruption, which fits the restartable nightly batch.

**Why option A is wrong:**
Spot capacity can be reclaimed with a short notice. Running the always-on API only on Spot risks availability, which the company wants to protect.

**Why option C is wrong:**
Reserved Instances apply to EC2 instances, not to Fargate tasks. Savings Plans are the discount model that covers Fargate.

**Why option E is wrong:**
Dedicated Hosts are for licensing or compliance needs and are generally more expensive. They do not provide volume discounts for container workloads.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> If your goal is to connect AWS architecture with ingestion, processing, and analytics, the complementary track is: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
