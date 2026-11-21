# Review Questions

## Question 1

A scenario requires the right decision on glossary. Which approach fits best?

A) Apply a generic solution without considering idempotency.
B) Use idempotency with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of glossary with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take idempotency into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine blast radius with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of glossary with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of RTO when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of glossary with the best balance between requirement, risk, and operations.

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
B) The one that ignores RPO signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of glossary with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on RPO and throttling with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of glossary with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company's disaster recovery policy states that after an outage the payment system must be back in service within 4 hours, and no more than 15 minutes of transactions may be lost. The current plan takes a snapshot of the database once every night.

Which statement correctly interprets the policy and the plan?

A) The RTO is 15 minutes and the RPO is 4 hours, so nightly snapshots meet the requirement
B) The RPO is 15 minutes and the RTO is 4 hours, so nightly snapshots do not meet the RPO
C) The RPO and the RTO are both 4 hours, so nightly snapshots are enough as long as restores finish in time
D) The RPO is 15 minutes and the RTO is 4 hours, so nightly snapshots meet the RPO but not the RTO

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
RPO (recovery point objective) is the maximum acceptable amount of data loss measured in time, and RTO (recovery time objective) is the maximum acceptable downtime. Nightly snapshots can lose up to 24 hours of data, so they violate a 15-minute RPO regardless of how fast a restore is.

**Why option A is wrong:**
This swaps the two terms. Data loss tolerance is the RPO and downtime tolerance is the RTO, and even with the swapped values a nightly snapshot would not fit.

**Why option C is wrong:**
The two values are different: 4 hours applies to downtime, and 15 minutes applies to data loss. Treating both as 4 hours ignores the data-loss limit.

**Why option D is wrong:**
The terms are assigned correctly, but the conclusion is wrong. A nightly snapshot can lose up to a day of data, so it fails the 15-minute RPO.

</details>

## Question 7

An e-commerce site is busy for two hours every evening and nearly idle the rest of the day. The architecture team wants compute capacity to be added automatically as load rises and removed when it falls, so that the company pays only for what it uses.

Which cloud characteristic does this describe?

A) High availability
B) Durability
C) Elasticity
D) Vertical scalability achieved by manually resizing instances

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Elasticity is the ability to acquire and release resources automatically in response to demand, for example with Auto Scaling. It addresses both the capacity needs of the peak and the cost of the idle time.

**Why option A is wrong:**
High availability is about keeping the application operating despite component or Availability Zone failures. It says nothing about matching capacity to load.

**Why option B is wrong:**
Durability describes how unlikely it is that stored data is lost, such as the 11 nines of S3. It does not relate to compute capacity.

**Why option D is wrong:**
Manually resizing instances is vertical scaling and usually requires a stop and start. It is not automatic and cannot release capacity quickly when demand falls, so it is not elasticity.

</details>

## Question 8

An application runs on EC2 instances in an Auto Scaling group across two Availability Zones behind an Application Load Balancer. When one zone fails, the load balancer needs about a minute of health checks to stop routing to it, and some in-flight requests fail. Management asks whether the system is fault tolerant.

Which answer is MOST accurate?

A) It is highly available but not fully fault tolerant, because users can still see a brief disruption when a component fails
B) It is fault tolerant, because the instances are spread over multiple Availability Zones
C) It is neither highly available nor fault tolerant, because high availability requires multiple Regions
D) It is elastic but not highly available, because the Auto Scaling group has to replace the failed capacity

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
High availability means the system recovers quickly and keeps serving after a failure, which this design does through redundancy and automated routing. Fault tolerance is stricter: the workload continues with no interruption at all, which usually needs extra standing capacity and no dependence on detection time.

**Why option B is wrong:**
Using multiple Availability Zones provides redundancy, but redundancy alone does not mean no interruption. The failed requests show that the system is not fully fault tolerant.

**Why option C is wrong:**
High availability does not need multiple Regions. A Multi-AZ design is the standard way to achieve it within a Region.

**Why option D is wrong:**
Elasticity concerns scaling with demand, not surviving failure. The remaining zone continues to serve traffic, so the system is highly available.

</details>

## Question 9

A web application stores each user's shopping cart in the memory of the EC2 instance that handled the login. After an Auto Scaling scale-in event terminates instances, users lose their carts and are signed out. The team wants scale-in events to stop affecting users.

What should the architect do?

A) Enable sticky sessions on the Application Load Balancer
B) Increase the Auto Scaling cooldown period
C) Move to a larger instance type so that fewer instances are needed
D) Store session and cart data in an external store such as Amazon ElastiCache or DynamoDB so the web tier becomes stateless

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
A stateless web tier keeps no user data locally, so any instance can serve any request and instances can be added or terminated freely. Externalizing session state to a shared store is the standard way to achieve this.

**Why option A is wrong:**
Sticky sessions pin a user to one instance, but the data is still on that instance. When it is terminated, the session is lost anyway.

**Why option B is wrong:**
A longer cooldown only delays the next scaling activity. Terminated instances still take their in-memory sessions with them.

**Why option C is wrong:**
Fewer, larger instances reduce the number of terminations but not their impact, and an instance failure would still lose all the carts on it.

</details>

## Question 10

A German company must keep customer data within Germany for regulatory reasons. It also wants the application to keep running if one data center fails.

Which design meets both requirements?

A) Deploy to two Availability Zones, one in the Frankfurt Region and one in the Ireland Region
B) Deploy everything in a single Availability Zone and rely on frequent backups to Amazon S3
C) Deploy across multiple Availability Zones in the Europe (Frankfurt) Region
D) Store the data in CloudFront edge locations in Europe so that it stays close to the users

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
A Region is a geographic area that contains multiple, isolated Availability Zones, each made up of one or more data centers. Using several Availability Zones in the Frankfurt Region survives the loss of a data center while keeping the data inside Germany.

**Why option A is wrong:**
Availability Zones belong to a single Region, so this is really two Regions. Ireland is outside Germany, which violates the data residency requirement.

**Why option B is wrong:**
A single Availability Zone is a single point of failure. Restoring from backups causes an outage rather than keeping the application running.

**Why option D is wrong:**
Edge locations cache content for delivery. They are not a primary data store, and they do not guarantee that data stays in Germany.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> To consolidate architecture with real data projects on AWS, keep the complementary track on your radar: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
