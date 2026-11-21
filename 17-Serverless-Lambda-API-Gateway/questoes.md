# Review Questions

## Question 1

A scenario requires the right decision on serverless. Which approach fits best?

A) Apply a generic solution without considering Lambda.
B) Use Lambda with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of serverless with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take Lambda into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine API Gateway with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of serverless with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of Step Functions when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of serverless with the best balance between requirement, risk, and operations.

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
B) The one that ignores cold start signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of serverless with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on cold start and throttling with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of serverless with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A web application uses API Gateway and Lambda functions that query an Amazon RDS for MySQL database. During traffic spikes, thousands of concurrent Lambda executions open new connections, and the database starts rejecting them with "too many connections" errors. The team wants to fix this with minimal code changes. What should the architect do?

A) Increase the memory allocated to the Lambda functions
B) Configure provisioned concurrency on the Lambda functions
C) Enable Multi-AZ on the RDS instance
D) Place Amazon RDS Proxy between the Lambda functions and the database

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
RDS Proxy pools and shares database connections, so many concurrent Lambda executions multiplex over a smaller number of connections to the database. Applications typically only need to change the endpoint they connect to.

**Why option A is wrong:**
More memory also gives a function more CPU, but each concurrent execution still opens its own connection. The connection count does not go down.

**Why option B is wrong:**
Provisioned concurrency keeps execution environments initialized to reduce cold starts. It does not pool connections, and it can even keep more connections open.

**Why option C is wrong:**
Multi-AZ provides a standby for high availability and failover. The standby does not serve traffic, so the connection limit of the primary stays the same.

</details>

## Question 7

A latency-sensitive API runs on API Gateway and a Java Lambda function. Users report that after quiet periods the first requests take 4 to 6 seconds, but the target is under 500 ms at the 99th percentile. Traffic is steady during business hours with about 50 concurrent requests. Which solution best addresses the issue?

A) Configure provisioned concurrency for the function, sized to the expected concurrency
B) Set reserved concurrency on the function to 50
C) Increase the function timeout to 60 seconds
D) Schedule an EventBridge rule to invoke the function every 5 minutes

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
The delays are cold starts, which are especially long for Java. Provisioned concurrency keeps the requested number of execution environments initialized and ready, so requests up to that level avoid cold starts.

**Why option B is wrong:**
Reserved concurrency guarantees and limits the maximum concurrency for the function, but it does not pre-initialize environments, so cold starts still happen.

**Why option C is wrong:**
The timeout is the maximum run time of an invocation. A longer timeout does not make the function start faster.

**Why option D is wrong:**
A periodic ping keeps at most one environment warm. It does nothing for the other concurrent requests, which still hit cold starts, and it is an unreliable workaround.

</details>

## Question 8

A company exposes a REST API through Amazon API Gateway to paying partners. The Gold tier is allowed 1 million requests per month and 100 requests per second. The Silver tier is allowed 100,000 requests per month and 10 requests per second. Partners are identified by an API key. How should the architect enforce these limits?

A) Use an AWS WAF rate-based rule with a different limit for each tier
B) Use Amazon Cognito user pool scopes to define the tiers
C) Create API Gateway usage plans for each tier and associate each partner's API key
D) Set a different reserved concurrency on the backend Lambda function for each tier

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Usage plans in API Gateway define throttle (rate and burst) and quota limits per period, and are associated with API keys. This lets each partner tier be given its own monthly quota and request rate.

**Why option A is wrong:**
WAF rate-based rules count requests per IP address or other aggregation keys over a short window. They do not provide per-customer monthly quotas tied to API keys.

**Why option B is wrong:**
Cognito scopes handle authorization, meaning what a caller is allowed to access. They do not throttle or meter request volume.

**Why option D is wrong:**
Reserved concurrency applies to a function as a whole, not to individual customers or tiers, and it limits concurrent executions rather than enforcing monthly quotas.

</details>

## Question 9

A company stores uploaded video files in S3. Each upload must be transcoded by a custom containerized tool that takes up to 45 minutes and needs 16 GB of memory. Uploads arrive unpredictably, from none for hours to dozens at once. The company wants to avoid paying for idle servers. Which design should the architect choose?

A) An S3 event notification invokes a Lambda function with the timeout set to 60 minutes
B) An S3 event triggers EventBridge, which starts an ECS task on Fargate that runs the container to completion
C) An S3 event invokes a Lambda function that invokes itself every 14 minutes until the job finishes
D) A fleet of EC2 instances polls S3 continuously for new objects

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Fargate tasks have no 15-minute limit, can be sized with enough memory, and are billed only while they run. Triggering them from S3 through EventBridge gives an event-driven, scale-to-zero design with no idle servers.

**Why option A is wrong:**
The maximum Lambda timeout is 15 minutes, so a 60-minute setting is not possible and a 45-minute job cannot complete in a single invocation.

**Why option C is wrong:**
Chaining self-invocations to get around the limit is fragile and hard to manage. A transcoding process cannot be reliably paused and resumed across invocations.

**Why option D is wrong:**
An always-on EC2 fleet polling for work incurs idle cost during quiet hours and adds scaling and patching overhead.

</details>

## Question 10

A Lambda function is attached to a VPC so that it can reach a private RDS database. It must also call a third-party HTTPS API on the internet and read from DynamoDB, and the company wants the DynamoDB traffic to stay on the AWS network. The function currently times out when calling the external API. Which TWO actions should the architect take?

A) Enable the function's public IP address setting so it can reach the internet directly
B) Place the function in private subnets and route 0.0.0.0/0 to a NAT gateway in a public subnet
C) Place the function in a public subnet with a route to an internet gateway
D) Add a gateway VPC endpoint for DynamoDB and update the route tables of the function's subnets
E) Place the NAT gateway in a private subnet with no route to an internet gateway

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, D

**Explanation:**
Lambda functions in a VPC use private ENIs, so internet access requires a NAT gateway in a public subnet reached from the function's private subnets. A gateway VPC endpoint for DynamoDB keeps that traffic on the AWS network and avoids sending it through the NAT gateway.

**Why option A is wrong:**
Lambda functions attached to a VPC do not get public IP addresses, and there is no such setting that would give them one.

**Why option C is wrong:**
Even in a public subnet, a VPC-attached function only has a private IP, so an internet gateway alone does not give it internet access.

**Why option E is wrong:**
A NAT gateway must be in a public subnet with a route to an internet gateway. Without that route it cannot reach the internet.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> To turn this architecture foundation into practice with pipelines, data, and analytics, you can also follow the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
