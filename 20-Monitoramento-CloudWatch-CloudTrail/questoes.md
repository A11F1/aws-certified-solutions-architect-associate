# Review Questions

## Question 1

A scenario requires the right decision on monitoring. Which approach fits best?

A) Apply a generic solution without considering CloudWatch.
B) Use CloudWatch with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of monitoring with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take CloudWatch into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine CloudTrail with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of monitoring with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of alarmes when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of monitoring with the best balance between requirement, risk, and operations.

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
B) The one that ignores logs signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of monitoring with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on logs and MTTR with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of monitoring with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company uses AWS Organizations with 40 accounts. Auditors ask who deleted an S3 bucket in one of the accounts 14 months ago, and policy requires that API activity records be kept for 7 years and be provably unaltered. What should the architect have configured to meet this requirement?

A) Rely on the CloudTrail Event history in each account's console
B) Create an organization trail that delivers to a central S3 bucket with log file integrity validation enabled
C) Create CloudWatch Logs metric filters on every account's default log groups
D) Enable AWS Config in each account and use the configuration timeline

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
An organization trail records management events for all member accounts and writes them to an S3 bucket where retention can be controlled for as long as needed, such as 7 years. Log file integrity validation produces digest files that prove the logs were not modified or deleted. Option B satisfies both the retention and tamper-evidence requirements.

**Why option A is wrong:**
Event history only covers the last 90 days of management events per account and offers no integrity validation, so a 14-month-old deletion would be unavailable.

**Why option C is wrong:**
Metric filters work on logs that are already in CloudWatch Logs and do not capture API activity by themselves. There are no default log groups that record API calls, so this does not create an audit trail.

**Why option D is wrong:**
AWS Config tracks resource configuration state over time. It is not designed to be the authoritative record of which principal made each API call, which is what CloudTrail provides.

</details>

## Question 7

A team runs a memory-intensive Java application on EC2 and wants to be alerted when memory utilization exceeds 85 percent for 10 minutes. They checked the EC2 console and cannot find a memory metric. What is the correct way to implement this alert?

A) Enable detailed monitoring on the instances and alarm on the MemoryUtilization metric
B) Create a CloudWatch alarm on the default EC2 MemoryUtilization metric
C) Create a CloudTrail trail and alarm on events that show high memory usage
D) Install the CloudWatch agent to publish memory as a custom metric, then create an alarm that notifies an SNS topic

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
EC2 sends hypervisor-level metrics such as CPU, network, and disk I/O, but it cannot see inside the guest operating system, so memory is not reported by default. The CloudWatch agent collects in-guest metrics like memory and publishes them as custom metrics, which an alarm can evaluate and send to SNS. Option D is the supported approach.

**Why option A is wrong:**
Detailed monitoring only increases the frequency of the standard metrics to 1 minute. It does not add a memory metric.

**Why option B is wrong:**
There is no default EC2 MemoryUtilization metric, so there is nothing to alarm on until the agent is installed.

**Why option C is wrong:**
CloudTrail records API calls, not operating system resource usage. It cannot report memory pressure on an instance.

</details>

## Question 8

A single EC2 instance hosts a legacy licensed application that is tied to its instance ID and private IP address. The instance occasionally becomes unreachable due to underlying hardware problems on the host. The operations team wants automatic recovery with the same instance ID and IP address and as little effort as possible. What should the architect do?

A) Create a CloudWatch alarm on the StatusCheckFailed_System metric with the EC2 recover action
B) Place the instance in an Auto Scaling group with a minimum and maximum size of 1
C) Create a Route 53 health check and fail over to a standby instance
D) Send system logs to CloudWatch Logs and trigger a Lambda function that launches a replacement instance

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
A CloudWatch alarm on StatusCheckFailed_System can automatically recover the instance by moving it to new hardware. The recovered instance keeps its instance ID, private IP addresses, Elastic IP addresses, and metadata. Option A fits the licensing constraint with almost no operational work.

**Why option B is wrong:**
An Auto Scaling group terminates an unhealthy instance and launches a new one with a new instance ID and typically a new private IP, which would break the licensed application.

**Why option C is wrong:**
A standby instance is a different instance with its own ID and IP, and it adds cost and configuration. It does not meet the requirement to keep the same instance identity.

**Why option D is wrong:**
Launching a replacement creates a new instance ID, and the custom Lambda logic adds operational burden. The built-in recover action already handles this scenario.

</details>

## Question 9

A security team must detect any security group that allows SSH from 0.0.0.0/0 and automatically remove the offending rule within minutes. They also need a history of which security groups were non-compliant over time. Which solution meets these requirements with the least custom code?

A) Create a CloudWatch alarm on EC2 CPU utilization and notify the security team
B) Review the AWS Trusted Advisor security checks every week and fix issues manually
C) Use an AWS Config managed rule for restricted SSH with an automatic remediation action that runs an SSM Automation document
D) Enable VPC Flow Logs and query them with Athena to find SSH traffic from the internet

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
AWS Config continuously evaluates resource configurations against rules and keeps a compliance history. The managed rule for restricted SSH flags security groups open to the world, and remediation actions can invoke SSM Automation documents to fix them automatically. Option C delivers detection, remediation, and history without custom code.

**Why option A is wrong:**
CPU utilization says nothing about security group configuration, so the alarm would never detect an open SSH rule.

**Why option B is wrong:**
Trusted Advisor can flag open ports, but weekly manual review does not remove the rule within minutes and gives no automatic remediation.

**Why option D is wrong:**
Flow logs show traffic that actually occurred, not the configuration of the security group, and analyzing them with Athena is a manual, after-the-fact process.

</details>

## Question 10

A company runs a web application on 30 EC2 instances. Operations wants to centralize application logs, search them across all instances, and be alerted when more than 10 ERROR lines appear within 5 minutes. Which TWO actions should the architect take?

A) Enable CloudTrail data events for EC2 to capture the application log lines
B) Install the CloudWatch agent on the instances to send application logs to a CloudWatch Logs log group
C) Enable AWS X-Ray tracing so that X-Ray counts ERROR lines in the log files
D) Create a metric filter on the log group that counts ERROR lines, and create a CloudWatch alarm on the resulting metric
E) Copy the log files to S3 with a nightly cron job and query them with Athena the next day

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, D

**Explanation:**
The CloudWatch agent streams application log files to a central CloudWatch Logs log group, where they can be searched with Logs Insights. A metric filter turns matching ERROR lines into a CloudWatch metric, and an alarm on that metric sends the notification. Options B and D together provide centralization and near real-time alerting.

**Why option A is wrong:**
CloudTrail data events record API activity on resources such as S3 objects and Lambda invocations. They do not capture the contents of application log files on an EC2 instance.

**Why option C is wrong:**
X-Ray traces requests across application components and does not parse or count log lines in files.

**Why option E is wrong:**
A nightly copy provides only next-day visibility, so the 5-minute alerting requirement cannot be met.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> When you want to take this topic into data, lakehouse, and hands-on project scenarios, use as your next step: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
