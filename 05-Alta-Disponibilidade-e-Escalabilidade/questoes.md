# Review Questions

## Question 1

A scenario requires the right decision on high availability. Which approach fits best?

A) Apply a generic solution without considering Multi-AZ.
B) Use Multi-AZ with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of high availability with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take Multi-AZ into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine health checks with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of high availability with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of horizontal scalability when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of high availability with the best balance between requirement, risk, and operations.

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
B) The one that ignores stateless signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of high availability with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on stateless and ASG with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of high availability with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A payments company exposes a TCP-based API that is consumed by partner banks. The partners can only allow traffic from a small list of fixed IP addresses in their firewalls. The service must handle millions of requests per second with very low latency. Which load balancing solution should the architect choose?

A) Application Load Balancer with Elastic IP addresses attached to it
B) Classic Load Balancer with a fixed DNS name
C) Network Load Balancer with one Elastic IP address per Availability Zone
D) Application Load Balancer with a Route 53 alias record

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
A Network Load Balancer operates at layer 4, handles millions of requests per second with very low latency, and supports one static Elastic IP per AZ. Those fixed addresses can be given to partners for allow-listing.

**Why option A is wrong:**
You cannot attach Elastic IPs to an Application Load Balancer. Its IP addresses change over time, and it is designed for HTTP and HTTPS routing.

**Why option B is wrong:**
The Classic Load Balancer is a previous-generation service with no static IP support and is not intended for this scale or latency.

**Why option D is wrong:**
A Route 53 alias record points a DNS name at the ALB, but the ALB's underlying IPs still change, so partners cannot allow-list them.

</details>

## Question 7

A web tier runs in an Auto Scaling group behind an ALB. Traffic is unpredictable, with sudden peaks during promotions. The operations team wants the fleet to keep average CPU utilization around 50 percent without maintaining multiple alarms and step thresholds. Which scaling policy should it use?

A) Target tracking scaling on average CPU utilization
B) Simple scaling with a fixed cooldown period
C) Scheduled scaling based on last month's traffic
D) Manual scaling by adjusting desired capacity during incidents

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Target tracking works like a thermostat. You choose a metric and target value, and Auto Scaling creates and manages the CloudWatch alarms and adjusts capacity to keep the metric near the target.

**Why option B is wrong:**
Simple scaling requires you to define alarms and fixed adjustments, and it waits out a cooldown before reacting again, which is slow for sudden peaks.

**Why option C is wrong:**
Scheduled scaling suits predictable patterns. Unpredictable promotion peaks would not match a schedule.

**Why option D is wrong:**
Manual changes depend on someone noticing the problem and acting, which contradicts the goal of low operational effort.

</details>

## Question 8

A company runs a MySQL database on Amazon RDS. Heavy reporting queries run during business hours and slow down the transactional workload. The company also needs automatic failover if the primary database instance or its Availability Zone fails. Which combination addresses both needs?

A) Enable Multi-AZ and point the reporting tool at the standby instance
B) Increase the instance size and remain in a single AZ
C) Place an Application Load Balancer in front of the database to distribute queries
D) Enable Multi-AZ for failover and create a read replica for the reporting queries

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Multi-AZ keeps a synchronous standby in another AZ and fails over automatically for high availability. Read replicas use asynchronous replication and serve read traffic, so reporting can be moved off the primary.

**Why option A is wrong:**
In a standard Multi-AZ instance deployment the standby cannot be read from. It exists only for failover.

**Why option B is wrong:**
A bigger instance may help performance temporarily but gives no failover and leaves the reporting load on the primary.

**Why option C is wrong:**
An ALB routes HTTP and HTTPS traffic and cannot load balance database connections. Even if it could, it would not provide replication.

</details>

## Question 9

A company runs its primary application stack in us-east-1 and keeps a scaled-down standby stack in us-west-2. It wants DNS to send all users to us-east-1 normally, and automatically redirect them to us-west-2 only if the primary endpoint becomes unhealthy. Which Route 53 configuration meets this?

A) Weighted routing with 100 for the primary and 0 for the standby and no health checks
B) Failover routing with a health check on the primary record
C) Latency-based routing across both Regions
D) Geolocation routing that maps every country to the primary Region

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Failover routing supports active-passive designs. Route 53 returns the primary record while its health check passes and switches to the secondary record when the check fails.

**Why option A is wrong:**
Without a health check Route 53 has no way to detect the failure, so it keeps returning the primary record.

**Why option C is wrong:**
Latency routing sends users to the Region with the lowest latency, so some users would be sent to us-west-2 even when the primary is healthy.

**Why option D is wrong:**
Geolocation routing is based on the user's location and does not by itself react to endpoint health, so it does not provide failover to the standby.

</details>

## Question 10

An image-processing application uses EC2 workers that read jobs from an SQS queue. Workers currently run in a single Availability Zone, and a recent AZ disruption left a large backlog unprocessed. The company wants the worker fleet to survive an AZ failure and to scale with the number of waiting jobs. Which TWO changes should the architect make?

A) Replace the workers with a single larger instance in the same AZ
B) Run the workers in an Auto Scaling group that spans at least two Availability Zones
C) Scale the group on the CPU utilization of the web servers that submit jobs
D) Configure target tracking on a backlog-per-instance metric derived from the queue length
E) Assign an Elastic IP address to each worker so jobs can find them

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, D

**Explanation:**
Spreading the Auto Scaling group across multiple AZs lets it keep and replace capacity if one AZ fails. Scaling on queue backlog per instance, a common pattern built from the SQS ApproximateNumberOfMessagesVisible metric and the group size, matches capacity to the actual work waiting.

**Why option A is wrong:**
A single larger instance in one AZ is still a single point of failure and does not scale with the queue.

**Why option C is wrong:**
The web servers' CPU does not reflect how many jobs are waiting for processing, so scaling decisions would not track the real workload.

**Why option E is wrong:**
Workers pull messages from SQS, so nothing needs to reach them by a fixed address. Elastic IPs add management overhead and do nothing for availability or scaling.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> If your goal is to connect AWS architecture with ingestion, processing, and analytics, the complementary track is: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
