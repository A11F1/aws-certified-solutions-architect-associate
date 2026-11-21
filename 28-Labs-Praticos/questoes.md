# Review Questions

## Question 1

A scenario requires the right decision on hands-on labs. Which approach fits best?

A) Apply a generic solution without considering teardown.
B) Use teardown with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hands-on labs with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take teardown into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine runbook with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hands-on labs with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of failure testing when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hands-on labs with the best balance between requirement, risk, and operations.

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
B) The one that ignores baseline security signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hands-on labs with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on baseline security and DR lab with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hands-on labs with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

An EC2 instance in a private subnet must download operating system patches from the internet. It must not be reachable from the internet. The VPC already has an internet gateway attached, and a public subnet exists in the same Availability Zone.

What should a solutions architect do?

A) Add a route to the internet gateway in the private subnet's route table and keep the security group's inbound rules closed
B) Create an egress-only internet gateway and route all outbound IPv4 traffic from the private subnet to it
C) Edit the instance's security group to allow outbound traffic to 0.0.0.0/0
D) Create a NAT gateway in the public subnet and add a 0.0.0.0/0 route to it in the private subnet's route table

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
A NAT gateway in a public subnet lets instances in private subnets start outbound connections to the internet while blocking connections initiated from the internet. The private subnet's route table needs a default route that points to the NAT gateway.

**Why option A is wrong:**
A route to the internet gateway makes the subnet public. Without a public IP address the instance still could not use it, and giving it one would expose the instance, which the requirement forbids.

**Why option B is wrong:**
An egress-only internet gateway works only for IPv6 traffic. It cannot provide outbound IPv4 access.

**Why option C is wrong:**
Security groups allow all outbound traffic by default, so this changes nothing. The problem is a missing route, not a filtering rule.

</details>

## Question 7

A company stores application logs in Amazon S3. The logs are read often during the first 30 days, rarely afterward, and must be kept for 7 years. Retrieval within several hours is acceptable for logs older than one year.

Which lifecycle configuration is the MOST cost-effective and valid?

A) Transition to S3 Standard-IA after 30 days, transition to S3 Glacier Flexible Retrieval after 365 days, and expire after 7 years
B) Transition to S3 Glacier Deep Archive on day 1 and expire after 7 years
C) Enable versioning with MFA delete and keep all objects in S3 Standard for 7 years
D) Transition to S3 One Zone-IA after 7 days, then to S3 Glacier Flexible Retrieval after 365 days, and expire after 7 years

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
The data is hot for 30 days, so S3 Standard fits that period. Standard-IA is cheaper for infrequently accessed data, and Glacier Flexible Retrieval is cheaper still for archives that tolerate retrieval in minutes to hours. An expiration action deletes the objects after 7 years.

**Why option B is wrong:**
Deep Archive retrievals take hours, which does not work for logs that are read often in the first 30 days. Day-1 archiving also makes recent logs impractical to use.

**Why option C is wrong:**
Versioning and MFA delete protect against accidental deletion, but they do not reduce cost. Keeping seven years in Standard is the most expensive option.

**Why option D is wrong:**
Lifecycle rules cannot transition objects from S3 Standard to Standard-IA or One Zone-IA until they are at least 30 days old, so a transition at 7 days is not valid. One Zone-IA would also store the only copy of the logs in a single Availability Zone.

</details>

## Question 8

An operations team wants an alarm when memory utilization on a fleet of Amazon Linux EC2 instances exceeds 80 percent, and wants to be notified by email. They find no memory metric in the EC2 console.

What should the team do?

A) Create an alarm on the MemoryUtilization metric in the AWS/EC2 namespace and send it to an Amazon SNS topic
B) Enable detailed monitoring on the instances to publish memory metrics every minute
C) Install the Amazon CloudWatch agent on the instances to publish memory metrics, create an alarm on that metric, and notify an Amazon SNS topic with an email subscription
D) Create an AWS CloudTrail trail and filter the events for high memory usage

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
EC2 publishes CPU, network, and disk I/O metrics from the hypervisor level, but it cannot see inside the operating system. The CloudWatch agent collects in-guest metrics such as memory as custom metrics. A CloudWatch alarm can then notify an SNS topic that has an email subscription.

**Why option A is wrong:**
The AWS/EC2 namespace has no memory metric because the hypervisor cannot observe guest memory. The alarm would have no data to evaluate.

**Why option B is wrong:**
Detailed monitoring only changes the frequency of the existing EC2 metrics from 5 minutes to 1 minute. It does not add memory metrics.

**Why option D is wrong:**
CloudTrail records API calls and account activity. It does not report resource utilization.

</details>

## Question 9

A web application on EC2 cannot connect to an Amazon RDS for MySQL instance in the same VPC. The application's security group is sg-app, and the database's security group is sg-db, which currently has no inbound rules. Security best practices require that the database not be exposed to the internet.

What is the BEST way to fix this?

A) Add an outbound rule for port 3306 to sg-app and leave sg-db unchanged
B) Add an inbound rule to sg-db that allows TCP port 3306 with sg-app as the source
C) Set the database to publicly accessible and allow 0.0.0.0/0 on port 3306 in sg-db
D) Add inbound and outbound rules for ephemeral ports 1024-65535 to sg-app

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Security groups deny inbound traffic by default. Allowing port 3306 from the application's security group lets only those instances reach the database, even as their IP addresses change.

**Why option A is wrong:**
Outbound traffic from sg-app is already allowed by default, and security groups are stateful. The blocking problem is on the database side, where no inbound rule exists.

**Why option C is wrong:**
This opens the database to the entire internet, which violates the stated requirement.

**Why option D is wrong:**
Ephemeral port rules are a network ACL concern for stateless filtering. Security groups are stateful, so return traffic is allowed automatically, and the missing inbound rule on sg-db would still block the connection.

</details>

## Question 10

A developer's application on an EC2 instance needs to read objects from a single S3 bucket. The developer currently stores long-term IAM user access keys in a configuration file on the instance. A security review flags this practice.

What is the MOST secure fix?

A) Put the access keys in the instance user data script so they are applied at launch
B) Make the bucket public so the application does not need credentials
C) Create one IAM user per instance and rotate its keys manually with a cron job
D) Create an IAM role with a policy that allows s3:GetObject on that bucket and attach it to the instance through an instance profile

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
An instance profile delivers temporary credentials through the instance metadata service, and the AWS SDKs rotate them automatically. No long-term keys are stored anywhere, and the policy follows least privilege.

**Why option A is wrong:**
User data is readable by anyone with access to the instance and is visible through the metadata service. The keys remain long-term credentials, so the problem is only moved.

**Why option B is wrong:**
A public bucket exposes the data to everyone on the internet, which is a much bigger risk than the one being fixed.

**Why option C is wrong:**
Per-instance IAM users still rely on long-term keys on disk, and manual rotation is error-prone and adds operational work. Roles avoid keys entirely.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
