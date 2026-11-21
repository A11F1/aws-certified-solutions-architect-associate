# Guided Lab: Decoupled Order Pipeline with SNS Fan-Out, SQS, DLQ and EventBridge

## Objective

Build an event-driven order flow where a publisher does not know its consumers: an SNS topic fans out to two SQS queues, a dead-letter queue (DLQ) catches poison messages, and an EventBridge rule routes matching events to a third queue. You also observe the SQS visibility timeout in action.

## Scenario

When an order is placed, the inventory service and the billing service must each receive their own copy. Billing sometimes fails to process a message, so after 2 failed attempts it must be parked in a DLQ. A separate audit queue receives selected events through EventBridge.

## Services Used

- Amazon SNS (topic and subscriptions)
- Amazon SQS (standard queues, DLQ)
- Amazon EventBridge (default bus, rule, target)
- AWS CLI v2 (or CloudShell)

## Prerequisites

- AWS account, region `us-east-1`, permissions for SNS, SQS and EventBridge.
- Shell with `aws` CLI and `python3` (for small JSON edits).

## Expected Cost

Free for practical purposes: SQS and SNS each include 1 million requests per month, and EventBridge custom events cost $1 per million. Nothing here runs hourly. Avoid: tight polling loops on thousands of messages and leftover topics with many subscriptions.

## Step by Step

### Step 1: Create the queues, including a DLQ for billing

Console: SQS > Create queue > Standard > `billing-dlq`; then `billing-queue` with Dead-letter queue enabled, pointing to `billing-dlq`, Maximum receives 2, visibility timeout 30 s; then `inventory-queue` and `audit-queue` with defaults.

```bash
export AWS_REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
for q in billing-dlq inventory-queue audit-queue; do aws sqs create-queue --queue-name $q; done

DLQ_URL=$(aws sqs get-queue-url --queue-name billing-dlq --query QueueUrl --output text)
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url $DLQ_URL --attribute-names QueueArn --query Attributes.QueueArn --output text)

cat > billing-attrs.json <<EOF
{
  "VisibilityTimeout": "30",
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"2\"}"
}
EOF
aws sqs create-queue --queue-name billing-queue --attributes file://billing-attrs.json
```

### Step 2: Create the SNS topic and subscribe the queues

```bash
TOPIC_ARN=$(aws sns create-topic --name order-events --query TopicArn --output text)
for q in billing-queue inventory-queue; do
  URL=$(aws sqs get-queue-url --queue-name $q --query QueueUrl --output text)
  ARN=$(aws sqs get-queue-attributes --queue-url $URL --attribute-names QueueArn --query Attributes.QueueArn --output text)
  # Allow only this topic to send to the queue
  python3 - "$ARN" "$TOPIC_ARN" > policy-$q.json <<'PY'
import json,sys
arn,topic=sys.argv[1],sys.argv[2]
policy={"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"sns.amazonaws.com"},
 "Action":"sqs:SendMessage","Resource":arn,"Condition":{"ArnEquals":{"aws:SourceArn":topic}}}]}
print(json.dumps({"Policy":json.dumps(policy)}))
PY
  aws sqs set-queue-attributes --queue-url $URL --attributes file://policy-$q.json
  aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint $ARN \
    --attributes RawMessageDelivery=true
done
```

Console equivalent: SNS > Topics > `order-events` > Create subscription > Protocol Amazon SQS > choose the queue. Without the queue access policy, delivery silently fails, which is a classic exam trap.

### Step 3: Publish and observe the fan-out

```bash
aws sns publish --topic-arn $TOPIC_ARN --message '{"orderId":"1001","total":42}'
for q in billing-queue inventory-queue; do
  URL=$(aws sqs get-queue-url --queue-name $q --query QueueUrl --output text)
  echo "== $q"; aws sqs receive-message --queue-url $URL --wait-time-seconds 5
done
```

Both queues hold an identical copy. Each consumer scales and fails independently.

### Step 4: Visibility timeout and DLQ experiment

Receive the billing message but never delete it (simulating a crashing consumer):

```bash
BILLING_URL=$(aws sqs get-queue-url --queue-name billing-queue --query QueueUrl --output text)
aws sqs receive-message --queue-url $BILLING_URL --visibility-timeout 5 --wait-time-seconds 2   # 1st failure
sleep 7
aws sqs receive-message --queue-url $BILLING_URL --visibility-timeout 5 --wait-time-seconds 2   # 2nd failure
sleep 7
aws sqs receive-message --queue-url $BILLING_URL --wait-time-seconds 5     # empty: moved to the DLQ
aws sqs receive-message --queue-url $DLQ_URL --wait-time-seconds 5          # the poison message is here
```

While the message is invisible it cannot be received by another consumer; if not deleted before the timeout, it reappears. After more receives than `maxReceiveCount` it is moved to the DLQ. (The message you consumed in Step 3 may need a fresh `sns publish` if you already received it there.)

### Step 5: Route events with EventBridge

```bash
aws events put-rule --name big-orders --event-pattern '{
  "source": ["lab.orders"],
  "detail-type": ["OrderPlaced"],
  "detail": { "total": [{ "numeric": [">", 100] }] }
}'

AUDIT_URL=$(aws sqs get-queue-url --queue-name audit-queue --query QueueUrl --output text)
AUDIT_ARN=$(aws sqs get-queue-attributes --queue-url $AUDIT_URL --attribute-names QueueArn --query Attributes.QueueArn --output text)

python3 - "$AUDIT_ARN" "$ACCOUNT_ID" > audit-policy.json <<'PY'
import json,sys
arn,acct=sys.argv[1],sys.argv[2]
p={"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"events.amazonaws.com"},
 "Action":"sqs:SendMessage","Resource":arn,
 "Condition":{"ArnEquals":{"aws:SourceArn":f"arn:aws:events:us-east-1:{acct}:rule/big-orders"}}}]}
print(json.dumps({"Policy":json.dumps(p)}))
PY
aws sqs set-queue-attributes --queue-url $AUDIT_URL --attributes file://audit-policy.json
aws events put-targets --rule big-orders --targets "Id=audit,Arn=$AUDIT_ARN"

# One event that matches, one that does not
aws events put-events --entries '[
 {"Source":"lab.orders","DetailType":"OrderPlaced","Detail":"{\"orderId\":\"2001\",\"total\":250}"},
 {"Source":"lab.orders","DetailType":"OrderPlaced","Detail":"{\"orderId\":\"2002\",\"total\":30}"}]'
aws sqs receive-message --queue-url $AUDIT_URL --wait-time-seconds 10
```

Only order 2001 should arrive in the audit queue. EventBridge filters on content, whereas SNS filters only on message attributes (or payload if configured).

## Validation

- `sns publish` produced one message in each of the billing and inventory queues.
- A message received twice without deletion lands in `billing-dlq` (check `ApproximateNumberOfMessages` on the DLQ).
- The audit queue contains order 2001 and not 2002.
- `aws sns list-subscriptions-by-topic --topic-arn $TOPIC_ARN` shows both queues as confirmed subscriptions.

## Cleanup

```bash
aws events remove-targets --rule big-orders --ids audit
aws events delete-rule --name big-orders
for sub in $(aws sns list-subscriptions-by-topic --topic-arn $TOPIC_ARN --query 'Subscriptions[].SubscriptionArn' --output text); do
  aws sns unsubscribe --subscription-arn $sub; done
aws sns delete-topic --topic-arn $TOPIC_ARN
for q in billing-queue inventory-queue audit-queue billing-dlq; do
  aws sqs delete-queue --queue-url $(aws sqs get-queue-url --queue-name $q --query QueueUrl --output text); done
```

## Exam Takeaway

Use SNS fan-out (topic to multiple SQS queues) when one event must be processed independently by several consumers, and SQS between producer and consumer to absorb spikes and survive consumer failure. Tune the visibility timeout to exceed processing time, and attach a DLQ (with a sensible `maxReceiveCount`) so poison messages do not loop forever. Pick EventBridge when you need rich content-based routing, many AWS or SaaS event sources, or schedules; pick SNS for simple high-throughput pub/sub.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
