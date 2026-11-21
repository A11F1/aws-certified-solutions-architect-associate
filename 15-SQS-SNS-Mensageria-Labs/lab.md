# Guided Lab: Messaging Deep-Dive with SQS Standard vs FIFO, Long Polling, DLQ Redrive and SNS Filter Policies

## Objective

Compare SQS Standard and FIFO behavior (ordering and deduplication), measure the effect of long polling, redrive messages from a DLQ back to the source queue, and use SNS subscription filter policies so each consumer receives only what it needs.

## Scenario

A payments platform needs strictly ordered, deduplicated ledger entries (FIFO), a cheap high-throughput notification queue (Standard), and an alert topic where the `fraud` queue receives only high-severity events. After a bug fix, failed messages in the DLQ must be replayed.

## Services Used

- Amazon SQS (Standard, FIFO, DLQ, redrive)
- Amazon SNS (filter policies)
- AWS CLI v2 (or CloudShell)

## Prerequisites

- AWS account, region `us-east-1`, permissions for SQS and SNS.
- `aws` CLI v2 and `python3`.

## Expected Cost

Free for this scale (1 million SQS requests and 1 million SNS publishes per month are free). Note that long polling lowers cost because it returns fewer empty responses. Avoid: aggressive short-polling loops, which waste requests.

## Step by Step

### Step 1: Create Standard and FIFO queues

Console: SQS > Create queue > FIFO > name `ledger.fifo` (the `.fifo` suffix is mandatory) > enable Content-based deduplication.

```bash
export AWS_REGION=us-east-1
aws sqs create-queue --queue-name ledger.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=false
aws sqs create-queue --queue-name notifications-std
FIFO_URL=$(aws sqs get-queue-url --queue-name ledger.fifo --query QueueUrl --output text)
STD_URL=$(aws sqs get-queue-url --queue-name notifications-std --query QueueUrl --output text)
```

### Step 2: FIFO ordering and deduplication

```bash
for i in 1 2 3; do
  aws sqs send-message --queue-url $FIFO_URL --message-body "entry-$i" \
    --message-group-id account-42 --message-deduplication-id dedup-$i
done
# Send entry-1 again with the same deduplication ID within 5 minutes
aws sqs send-message --queue-url $FIFO_URL --message-body "entry-1" \
  --message-group-id account-42 --message-deduplication-id dedup-1

for i in 1 2 3 4; do
  aws sqs receive-message --queue-url $FIFO_URL --wait-time-seconds 2 \
    --query 'Messages[0].[Body,ReceiptHandle]' --output text | tee /tmp/m.txt
  RH=$(cut -f2 /tmp/m.txt); [ "$RH" != "None" ] && aws sqs delete-message --queue-url $FIFO_URL --receipt-handle "$RH"
done
```

You receive `entry-1`, `entry-2`, `entry-3` in order, and only three messages: the duplicate was accepted by the API (it returns the original `MessageId`) but discarded. Messages in a group are delivered one after another, so a group blocks until its in-flight message is deleted or becomes visible again. FIFO supports 300 messages/s per API action without batching (3,000 with batching), and higher with high-throughput mode.

### Step 3: Long polling vs short polling

```bash
time aws sqs receive-message --queue-url $STD_URL --wait-time-seconds 0    # returns instantly, empty
time aws sqs receive-message --queue-url $STD_URL --wait-time-seconds 20   # waits up to 20 s for a message

# Make long polling the queue default
aws sqs set-queue-attributes --queue-url $STD_URL --attributes ReceiveMessageWaitTimeSeconds=20
```

In another terminal, send a message during the 20-second wait and notice it is delivered immediately, so long polling reduces both empty responses and latency. Maximum wait is 20 seconds.

### Step 4: Create a DLQ and replay it (redrive)

```bash
aws sqs create-queue --queue-name orders-dlq
DLQ_URL=$(aws sqs get-queue-url --queue-name orders-dlq --query QueueUrl --output text)
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url $DLQ_URL --attribute-names QueueArn --query Attributes.QueueArn --output text)

cat > orders-attrs.json <<EOF
{ "RedrivePolicy": "{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"1\"}",
  "VisibilityTimeout": "2" }
EOF
aws sqs create-queue --queue-name orders --attributes file://orders-attrs.json
ORDERS_URL=$(aws sqs get-queue-url --queue-name orders --query QueueUrl --output text)

aws sqs send-message --queue-url $ORDERS_URL --message-body "bad-order"
aws sqs receive-message --queue-url $ORDERS_URL --wait-time-seconds 2   # receive once, do not delete
sleep 4
aws sqs receive-message --queue-url $ORDERS_URL --wait-time-seconds 2   # empty: moved to the DLQ
aws sqs get-queue-attributes --queue-url $DLQ_URL --attribute-names ApproximateNumberOfMessages
```

Now "fix the bug" and redrive. Console: SQS > `orders-dlq` > Start DLQ redrive > Redrive to source queue. CLI:

```bash
aws sqs start-message-move-task --source-arn $DLQ_ARN
aws sqs list-message-move-tasks --source-arn $DLQ_ARN
aws sqs receive-message --queue-url $ORDERS_URL --wait-time-seconds 5    # bad-order is back
```

When no destination is given, messages go back to the queue they originally came from. DLQ for a FIFO queue must also be FIFO, and a Standard queue's DLQ must be Standard.

### Step 5: SNS filter policies

```bash
TOPIC_ARN=$(aws sns create-topic --name alerts --query TopicArn --output text)
aws sqs create-queue --queue-name fraud-queue
aws sqs create-queue --queue-name all-alerts-queue
for q in fraud-queue all-alerts-queue; do
  URL=$(aws sqs get-queue-url --queue-name $q --query QueueUrl --output text)
  ARN=$(aws sqs get-queue-attributes --queue-url $URL --attribute-names QueueArn --query Attributes.QueueArn --output text)
  python3 - "$ARN" "$TOPIC_ARN" > p-$q.json <<'PY'
import json,sys
arn,topic=sys.argv[1],sys.argv[2]
p={"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"sns.amazonaws.com"},
 "Action":"sqs:SendMessage","Resource":arn,"Condition":{"ArnEquals":{"aws:SourceArn":topic}}}]}
print(json.dumps({"Policy":json.dumps(p)}))
PY
  aws sqs set-queue-attributes --queue-url $URL --attributes file://p-$q.json
done

FRAUD_ARN=$(aws sqs get-queue-attributes --queue-url $(aws sqs get-queue-url --queue-name fraud-queue --query QueueUrl --output text) --attribute-names QueueArn --query Attributes.QueueArn --output text)
ALL_ARN=$(aws sqs get-queue-attributes --queue-url $(aws sqs get-queue-url --queue-name all-alerts-queue --query QueueUrl --output text) --attribute-names QueueArn --query Attributes.QueueArn --output text)

aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint $FRAUD_ARN \
  --attributes '{"RawMessageDelivery":"true","FilterPolicy":"{\"severity\":[\"high\"],\"type\":[\"fraud\"]}"}'
aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint $ALL_ARN \
  --attributes RawMessageDelivery=true

aws sns publish --topic-arn $TOPIC_ARN --message "card test" \
  --message-attributes '{"severity":{"DataType":"String","StringValue":"high"},"type":{"DataType":"String","StringValue":"fraud"}}'
aws sns publish --topic-arn $TOPIC_ARN --message "disk 80%" \
  --message-attributes '{"severity":{"DataType":"String","StringValue":"low"},"type":{"DataType":"String","StringValue":"infra"}}'
```

Read both queues: `all-alerts-queue` gets two messages, `fraud-queue` only the first. Messages without attributes do not match an attribute-based filter policy, so they are dropped for that subscriber (a frequent cause of "my messages vanished").

## Validation

- FIFO returned entries in order and silently dropped the duplicate `dedup-1`.
- The empty short poll returned immediately; the long poll waited about 20 seconds.
- After `start-message-move-task`, `bad-order` is back in `orders` and the DLQ count is 0.
- `fraud-queue` has 1 message; `all-alerts-queue` has 2.

## Cleanup

```bash
for sub in $(aws sns list-subscriptions-by-topic --topic-arn $TOPIC_ARN --query 'Subscriptions[].SubscriptionArn' --output text); do
  aws sns unsubscribe --subscription-arn $sub; done
aws sns delete-topic --topic-arn $TOPIC_ARN
for q in fraud-queue all-alerts-queue orders orders-dlq notifications-std ledger.fifo; do
  aws sqs delete-queue --queue-url $(aws sqs get-queue-url --queue-name $q --query QueueUrl --output text); done
```

## Exam Takeaway

Choose FIFO when order and exactly-once processing (deduplication) matter, accepting lower throughput; choose Standard for maximum throughput with at-least-once delivery and best-effort ordering, so consumers must be idempotent. Enable long polling to cut cost and latency. Use DLQs plus redrive to isolate and replay poison messages. Use SNS filter policies instead of building routing code when different subscribers need different subsets of one topic.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
