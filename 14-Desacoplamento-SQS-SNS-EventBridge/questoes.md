# Review Questions

## Question 1

A scenario requires the right decision on decoupling. Which approach fits best?

A) Apply a generic solution without considering SQS.
B) Use SQS with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of decoupling with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take SQS into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine SNS with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of decoupling with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of EventBridge when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of decoupling with the best balance between requirement, risk, and operations.

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
B) The one that ignores DLQ signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of decoupling with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on FIFO and idempotency with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of decoupling with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

An e-commerce company publishes an "order placed" event from its checkout service. The inventory, billing, and analytics services must each process every order independently, at their own pace, and a slow or failing consumer must not affect the others. The company also expects to add more consumers over the next year without changing the checkout code. Which architecture meets these requirements with the least operational effort?

A) Publish each order to a single SQS standard queue and have all three services poll that queue
B) Publish each order to an SNS topic and subscribe the three services' HTTPS endpoints directly
C) Publish each order to an SNS topic and subscribe a separate SQS queue for each service
D) Have the checkout service call each downstream service's API synchronously

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
The SNS-to-SQS fan-out pattern delivers a copy of every message to each subscribed queue. Each service consumes from its own durable queue, so it can process at its own speed and a failure in one consumer does not affect the others. New consumers are added by subscribing another queue to the topic, with no change to the publisher.

**Why option A is wrong:**
With one shared queue, each message is delivered to only one consumer and then deleted. The three services would compete for orders instead of each receiving every order.

**Why option B is wrong:**
Direct HTTPS subscriptions push to the endpoints, so a slow or unavailable service depends on SNS retry policies and loses the buffering a queue provides. There is also no per-service backlog to absorb spikes.

**Why option D is wrong:**
Synchronous calls tightly couple checkout to every downstream service. Any slow or failing service increases checkout latency or causes failures, and each new consumer requires a code change.

</details>

## Question 7

A payments company processes account transactions through a queue. Transactions for the same account must be processed in the exact order they were submitted, and a transaction must never be processed twice even if the producer retries a send. Different accounts can be processed in parallel. Which solution should the solutions architect choose?

A) An SQS FIFO queue, using the account ID as the message group ID and enabling deduplication
B) An SQS standard queue with a long visibility timeout so messages are handled one at a time
C) An SQS standard queue, with the application adding sequence numbers and discarding duplicates
D) An SNS standard topic with a Lambda subscription for each account

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
FIFO queues guarantee ordering within a message group and provide exactly-once processing through deduplication. Using the account ID as the message group ID keeps each account's transactions in order while allowing different accounts to be processed in parallel.

**Why option B is wrong:**
A visibility timeout only hides a message from other consumers while it is being processed. It does not provide ordering, and standard queues offer at-least-once delivery, so duplicates can occur.

**Why option C is wrong:**
Standard queues provide best-effort ordering, so messages can arrive out of order. Rebuilding ordering and deduplication in the application adds significant complexity and risk compared with a native feature.

**Why option D is wrong:**
Standard SNS topics do not guarantee ordering or exactly-once delivery, and creating a subscription per account does not scale and does not solve the problem.

</details>

## Question 8

A video platform has workers on EC2 instances that read messages from an SQS standard queue. Each message takes about 8 minutes to process. The team notices that the same message is often processed by two different workers, wasting compute. The queue still has its default settings. What should the architect do to fix this?

A) Enable long polling by setting ReceiveMessageWaitTimeSeconds to 20 seconds
B) Configure a delivery delay of 10 minutes on the queue
C) Increase the message retention period to 14 days
D) Increase the visibility timeout to be longer than the maximum processing time

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
The default visibility timeout is 30 seconds. After that, a message that has not been deleted becomes visible again and another worker receives it. Setting the visibility timeout above the 8-minute processing time, or extending it with ChangeMessageVisibility, prevents the duplicate processing.

**Why option A is wrong:**
Long polling reduces empty responses and API costs when the queue is empty. It does not change when an in-flight message becomes visible again.

**Why option B is wrong:**
A delivery delay postpones when a new message first becomes visible to consumers. It has no effect on a message that is already being processed and reappears after the visibility timeout.

**Why option C is wrong:**
Retention controls how long unprocessed messages are kept in the queue. It does not affect how long a received message stays hidden from other consumers.

</details>

## Question 9

A company wants to react to events from several sources: an AWS service (an S3 object creation), a custom application, and a third-party SaaS provider that is a supported EventBridge partner. Events must be routed to different targets based on fields in the event body, such as "priority = high". The team wants to avoid writing and running custom polling code. Which service should they use?

A) Amazon SNS with subscription filter policies, with the SaaS provider publishing directly to the topic
B) Amazon EventBridge with rules that use event patterns, using a partner event source for the SaaS events
C) Amazon SQS with a Lambda function that polls the SaaS provider's API and routes the messages
D) Amazon CloudWatch Logs with subscription filters on each source

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
EventBridge natively receives events from AWS services, custom applications, and SaaS partner event sources. Rules with event patterns can match on fields in the event body and route to different targets, all without custom code.

**Why option A is wrong:**
SNS filter policies can filter on message attributes or payload, but SNS has no native integration with SaaS partner event sources. The provider would need custom integration to publish to the topic.

**Why option C is wrong:**
This requires building and operating polling and routing code, which the company explicitly wants to avoid. SQS also has no content-based routing to multiple targets.

**Why option D is wrong:**
CloudWatch Logs subscription filters work on log data, not on events from services or SaaS partners, and they are not an event routing service.

</details>

## Question 10

A company's order-processing consumers read from an SQS standard queue. Occasionally a malformed message causes the consumer to fail every time it is processed, which keeps cycling through the queue and wastes compute. After 5 failed receives, the company wants the message set aside for inspection, and once a code fix is deployed it wants to reprocess those messages without writing custom tooling. Which TWO actions should the architect take?

A) Create a standard SQS dead-letter queue and set a redrive policy with maxReceiveCount of 5 on the source queue
B) Set the source queue's visibility timeout to 0 so failed messages retry immediately
C) Create a FIFO queue as the dead-letter queue for the standard source queue
D) Use the dead-letter queue redrive feature to move messages back to the source queue after the fix
E) Configure the consumer to delete any message that fails processing

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A, D

**Explanation:**
A redrive policy with maxReceiveCount of 5 moves a message to the dead-letter queue after five failed receives, isolating the poison message. DLQ redrive can then send those messages back to the source queue, or another destination, once the bug is fixed, without custom code.

**Why option B is wrong:**
A visibility timeout of 0 makes failed messages instantly available again, which makes the retry loop tighter and wastes even more compute.

**Why option C is wrong:**
A dead-letter queue must be the same type as its source queue. A standard queue cannot use a FIFO queue as its DLQ.

**Why option E is wrong:**
Deleting failed messages discards the data permanently, so there is nothing left to inspect or reprocess after the fix.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> When you want to take this topic into data, lakehouse, and hands-on project scenarios, use as your next step: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
