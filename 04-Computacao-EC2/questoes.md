# Review Questions

## Question 1

A scenario requires the right decision on EC2. Which approach fits best?

A) Apply a generic solution without considering instance types.
B) Use instance types with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of EC2 with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take instance types into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine Auto Scaling with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of EC2 with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of load balancers when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of EC2 with the best balance between requirement, risk, and operations.

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
B) The one that ignores EBS signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of EC2 with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on placement groups and Spot/Savings with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of EC2 with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A video company renders thumbnails by pulling jobs from an SQS queue. Each job is stateless and idempotent, takes under five minutes, and can be retried if a worker disappears. The workload is bursty and the company wants the lowest possible compute cost. Which EC2 purchasing option fits best?

A) Spot Instances in an Auto Scaling group using multiple instance types
B) On-Demand Instances sized for peak load
C) Three-year Standard Reserved Instances for the peak fleet size
D) Dedicated Hosts to guarantee capacity

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Spot Instances offer large discounts, up to about 90 percent versus On-Demand, and are ideal for fault-tolerant, stateless work. Diversifying across instance types and AZs reduces the chance of interruption, and the queue lets interrupted jobs be retried.

**Why option B is wrong:**
On-Demand works but costs significantly more, and sizing for peak means paying for idle capacity during quiet periods.

**Why option C is wrong:**
A three-year commitment for peak capacity wastes money on a bursty workload and locks the company into a specific configuration.

**Why option D is wrong:**
Dedicated Hosts are for licensing or compliance needs tied to physical servers and are among the most expensive options. No such requirement exists here.

</details>

## Question 7

A company runs a steady, 24/7 fleet of EC2 instances. Over the next year it plans to migrate part of the fleet from m5 instances to Graviton-based m7g instances and to move some workloads from us-east-1 to eu-west-1. It wants a one-year commitment that keeps discounts through these changes. What should the company purchase?

A) An EC2 Instance Savings Plan for the m5 family in us-east-1
B) Spot Instances for the whole fleet
C) A Compute Savings Plan
D) On-Demand Capacity Reservations for the current instances

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Compute Savings Plans apply automatically to EC2 usage regardless of instance family, size, Region, OS, or tenancy, and also cover Fargate and Lambda. That flexibility fits a fleet that will change families and Regions.

**Why option A is wrong:**
An EC2 Instance Savings Plan is tied to a specific instance family in a specific Region. Moving to m7g or to eu-west-1 would leave the commitment unused.

**Why option B is wrong:**
Spot can be interrupted and is unsuitable for a steady fleet that is presumably serving always-on workloads.

**Why option D is wrong:**
Capacity Reservations guarantee capacity in an AZ but give no discount by themselves, and they are tied to a specific instance type and AZ.

</details>

## Question 8

A self-managed database on EC2 must sustain a consistent 40,000 IOPS with single-digit millisecond latency, and the data must survive instance stops and replacements. Which storage choice should the architect use?

A) General Purpose SSD (gp3) volume with maximum provisioned IOPS
B) Instance store volumes attached to the instance
C) Throughput Optimized HDD (st1) volumes striped together
D) Provisioned IOPS SSD (io2) volume

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
io2 volumes are designed for I/O-intensive databases and can provision far more IOPS than gp3, with consistent low latency and higher durability. The data persists independently of the instance.

**Why option A is wrong:**
A gp3 volume tops out at 16,000 IOPS, which is below the 40,000 required.

**Why option B is wrong:**
Instance store offers very high IOPS, but the data is lost when the instance is stopped, terminated, or the underlying host fails. The data must persist.

**Why option C is wrong:**
st1 is optimized for sequential, throughput-heavy workloads such as logs and big data. It performs poorly for small random I/O and is not intended for databases.

</details>

## Question 9

A research team runs a tightly coupled HPC simulation on 16 EC2 instances. The nodes exchange large amounts of data constantly, so the team needs the lowest possible network latency and highest packet-per-second performance between instances. Which placement strategy should it use?

A) A spread placement group
B) A cluster placement group
C) A partition placement group
D) An Auto Scaling group that spreads instances across three Availability Zones with no placement group

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
A cluster placement group packs instances close together inside one Availability Zone, providing low-latency, high-throughput networking, particularly with enhanced networking. That suits tightly coupled HPC workloads.

**Why option A is wrong:**
Spread groups place each instance on distinct hardware to reduce correlated failures, which improves resilience but not network performance.

**Why option C is wrong:**
Partition groups separate sets of instances onto different racks for large distributed systems such as Hadoop or Cassandra. They do not optimize inter-node latency.

**Why option D is wrong:**
Spreading across AZs increases latency between nodes, the opposite of what the simulation needs.

</details>

## Question 10

A team of developers uses EC2 instances as workstations with large in-memory datasets. Loading the data takes about 40 minutes. At night the instances should not incur compute charges, but each morning the developers want to resume exactly where they stopped, with their in-memory state intact. Which action should they take at the end of the day?

A) Hibernate the instances, using encrypted EBS root volumes on a supported instance type
B) Stop the instances normally
C) Terminate the instances and relaunch them from an AMI each morning
D) Reboot the instances and leave them running overnight

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Hibernation saves the contents of RAM to the encrypted EBS root volume and stops the instance. Compute charges stop, and on start the in-memory state is restored. The instance must be on a supported type, have an encrypted root volume, and have enough root volume space to store the RAM contents.

**Why option B is wrong:**
A normal stop discards the RAM contents, so the 40-minute data load would need to be repeated each morning.

**Why option C is wrong:**
Terminating destroys the instance and its memory. Relaunching from an AMI would also lose the in-memory dataset and any unsaved data.

**Why option D is wrong:**
A reboot clears memory and the instance keeps running, so compute charges continue overnight.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> When you want to take this topic into data, lakehouse, and hands-on project scenarios, use as your next step: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
