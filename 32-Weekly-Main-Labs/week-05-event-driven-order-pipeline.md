# Week 5 Main Lab: Event-Driven Order Pipeline with SNS Fan-Out, SQS, DLQ and a Fargate Worker

## Objective

Build a decoupled order pipeline: an order event is published once to an SNS topic and fanned out to two SQS queues (billing and shipping), each protected by a dead-letter queue (DLQ). A containerized Fargate worker consumes the shipping queue, an EventBridge schedule feeds the billing queue, and CloudWatch plus Application Auto Scaling scale the workers by queue depth. You will inject a poison message, watch it land in the DLQ, and redrive it after "shipping the fix".

Time: 90 to 120 minutes.

## Modules Covered

- [14 - Decoupling: SQS, SNS and EventBridge](../14-Desacoplamento-SQS-SNS-EventBridge/README.md)
- [15 - SQS and SNS Messaging Labs](../15-SQS-SNS-Mensageria-Labs/lab.md)
- [16 - Containers: ECS, EKS and Fargate](../16-Containers-ECS-EKS-Fargate/README.md)

## Scenario and Architecture

"ShelfCo" receives orders from its storefront. Billing and shipping must each react to every order, at their own pace, without the storefront waiting for either. A slow or broken consumer must never lose orders or block the other consumer. Billing also needs a recurring reconciliation trigger every few minutes (in real life, daily).

```
 Storefront / CLI
      | sns publish
      v
 +-----------------+   fan-out (raw message delivery)
 | SNS  lab-orders |-----------------------------+----------------------------+
 +-----------------+                             |                            |
                                                 v                            v
                                   +---------------------------+  +---------------------------+
 EventBridge rule                  | SQS lab-billing           |  | SQS lab-shipping          |
 rate(5 minutes) ----------------->| visibility 30s            |  | visibility 30s            |
                                   | redrive: 3 receives -> DLQ|  | redrive: 3 receives -> DLQ|
                                   +-------------+-------------+  +-------------+-------------+
                                                 |  (no worker in lab:           |
                                                 |   inspect with CLI)           v
                                                 v                  +-------------------------+
                                   +---------------------------+    | ECS Fargate service     |
                                   | SQS lab-billing-dlq       |    | lab-shipping-worker     |
                                   +---------------------------+    | task role: sqs receive/ |
                                                                    | delete on shipping only |
                                   +---------------------------+    +------------+------------+
                                   | SQS lab-shipping-dlq      |<-- poison msg --+
                                   +---------------------------+
                                                                    CloudWatch alarm
                                                                    ApproximateNumberOfMessagesVisible
                                                                    -> step scaling -> DesiredCount 1..3
```

## Services Used

- Amazon SNS (standard topic, SQS subscriptions, raw message delivery)
- Amazon SQS (standard queues, DLQs, redrive policy, DLQ redrive to source)
- Amazon EventBridge (scheduled rule with SQS target)
- Amazon ECS on AWS Fargate, Amazon CloudWatch Logs and Alarms, Application Auto Scaling
- AWS IAM (execution role and task role)
- AWS CLI v2 (CloudShell works well)

## Prerequisites

- Personal AWS account, region `us-east-1`, administrator-level sandbox permissions.
- A default VPC with public subnets (the Fargate task gets a public IP so it can pull the image and reach SQS without a NAT Gateway or VPC endpoints; this is a lab shortcut).
- Basic familiarity with ECS task definitions.

## Expected Cost

Approximate; confirm on the SNS, SQS, EventBridge and Fargate pricing pages.

- SNS, SQS and EventBridge at this volume: effectively free (inside free tiers or fractions of a cent).
- Fargate bills per vCPU-second and GB-second while tasks run, with a one-minute minimum per task. A 0.25 vCPU / 0.5 GB task costs roughly a couple of cents per hour; up to 3 tasks during the scaling test is still cents. Keep it running no longer than the lab.
- CloudWatch Logs and alarms: pennies (standard alarms have a small monthly price each; delete them).
- Not used on purpose: NAT Gateway and interface VPC endpoints (both bill hourly). Do not add them for this lab, or delete them the same day if you do.

## Step by Step

### Step 1: Variables, DLQs and main queues

```bash
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

for q in lab-billing-dlq lab-shipping-dlq; do
  aws sqs create-queue --queue-name $q --attributes MessageRetentionPeriod=1209600
done

BILL_DLQ_ARN=arn:aws:sqs:us-east-1:$ACCOUNT_ID:lab-billing-dlq
SHIP_DLQ_ARN=arn:aws:sqs:us-east-1:$ACCOUNT_ID:lab-shipping-dlq
```

Create the main queues. `RedrivePolicy` is a JSON string inside the attribute map, so use a file:

```bash
for pair in billing:$BILL_DLQ_ARN shipping:$SHIP_DLQ_ARN; do
  name=${pair%%:*}; dlq=${pair#*:}
  cat > q-$name.json <<EOF
{
  "VisibilityTimeout": "30",
  "ReceiveMessageWaitTimeSeconds": "10",
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"$dlq\",\"maxReceiveCount\":\"3\"}"
}
EOF
  aws sqs create-queue --queue-name lab-$name --attributes file://q-$name.json
done

export BILL_URL=$(aws sqs get-queue-url --queue-name lab-billing --query QueueUrl --output text)
export SHIP_URL=$(aws sqs get-queue-url --queue-name lab-shipping --query QueueUrl --output text)
export SHIP_DLQ_URL=$(aws sqs get-queue-url --queue-name lab-shipping-dlq --query QueueUrl --output text)
BILL_ARN=arn:aws:sqs:us-east-1:$ACCOUNT_ID:lab-billing
SHIP_ARN=arn:aws:sqs:us-east-1:$ACCOUNT_ID:lab-shipping
```

`maxReceiveCount=3` means that after a message has been received three times without being deleted, SQS moves it to the DLQ. The 30-second visibility timeout keeps the poison-message experiment quick; in production the timeout should exceed your worst-case processing time. A DLQ for a standard queue must itself be a standard queue.

### Step 2: SNS topic and fan-out subscriptions

```bash
export TOPIC_ARN=$(aws sns create-topic --name lab-orders --query TopicArn --output text)
```

Allow the topic (and, for Step 5, EventBridge) to send to the queues. This one policy per queue is the part people forget:

```bash
cat > policy-billing.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "AllowSNS", "Effect": "Allow", "Principal": { "Service": "sns.amazonaws.com" },
      "Action": "sqs:SendMessage", "Resource": "$BILL_ARN",
      "Condition": { "ArnEquals": { "aws:SourceArn": "$TOPIC_ARN" } } },
    { "Sid": "AllowEventBridge", "Effect": "Allow", "Principal": { "Service": "events.amazonaws.com" },
      "Action": "sqs:SendMessage", "Resource": "$BILL_ARN",
      "Condition": { "ArnEquals": { "aws:SourceArn": "arn:aws:events:us-east-1:$ACCOUNT_ID:rule/lab-billing-reconcile" } } }
  ]
}
EOF
cat > policy-shipping.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "AllowSNS", "Effect": "Allow", "Principal": { "Service": "sns.amazonaws.com" },
      "Action": "sqs:SendMessage", "Resource": "$SHIP_ARN",
      "Condition": { "ArnEquals": { "aws:SourceArn": "$TOPIC_ARN" } } }
  ]
}
EOF
```

The `Policy` attribute also must be passed as a JSON string, so wrap it with `jq`:

```bash
aws sqs set-queue-attributes --queue-url $BILL_URL \
  --attributes "$(jq -n --rawfile p policy-billing.json '{Policy:$p}')"
aws sqs set-queue-attributes --queue-url $SHIP_URL \
  --attributes "$(jq -n --rawfile p policy-shipping.json '{Policy:$p}')"

for arn in $BILL_ARN $SHIP_ARN; do
  aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs \
    --notification-endpoint $arn --attributes RawMessageDelivery=true
done
```

Raw message delivery makes the SQS message body exactly the published text, without the SNS JSON envelope, which keeps the worker simple.

Test the fan-out:

```bash
aws sns publish --topic-arn $TOPIC_ARN --message '{"orderId":"o-1","total":42.5}'
aws sqs receive-message --queue-url $BILL_URL --wait-time-seconds 5 --query 'Messages[0].Body'
aws sqs receive-message --queue-url $SHIP_URL --wait-time-seconds 5 --query 'Messages[0].Body'
```

One publish, two copies. Purge both queues before continuing so the test copies do not confuse later steps: `aws sqs purge-queue --queue-url $BILL_URL` and the same for `$SHIP_URL` (purge can take up to a minute; there is a limit of one purge per queue per 60 seconds).

### Step 3: IAM roles for the Fargate worker

Two roles, two jobs. The execution role lets ECS pull images and write logs; the task role is what your code uses (least privilege: only the shipping queue).

```bash
cat > ecs-trust.json <<'EOF'
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Principal": { "Service": "ecs-tasks.amazonaws.com" }, "Action": "sts:AssumeRole" }] }
EOF

aws iam create-role --role-name lab-ecs-exec-role --assume-role-policy-document file://ecs-trust.json
aws iam attach-role-policy --role-name lab-ecs-exec-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws iam create-role --role-name lab-shipping-task-role --assume-role-policy-document file://ecs-trust.json
cat > task-policy.json <<EOF
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Action": ["sqs:ReceiveMessage","sqs:DeleteMessage","sqs:GetQueueAttributes"],
  "Resource": "$SHIP_ARN" }] }
EOF
aws iam put-role-policy --role-name lab-shipping-task-role \
  --policy-name shipping-queue-only --policy-document file://task-policy.json
```

### Step 4: Fargate worker (ECS cluster, task definition, service)

The worker is a shell loop in the public AWS CLI image: long-poll the queue, "process" the order (sleep 5 seconds to simulate work), and delete the message. A message containing the marker in `REJECT_MARKER` is treated as a bug and deliberately not deleted, so SQS redelivers it until it reaches the DLQ.

```bash
aws logs create-log-group --log-group-name /ecs/lab-shipping-worker
aws ecs create-cluster --cluster-name lab-orders
```

Write `taskdef.json` (replace `<ACCOUNT_ID>` and `<SHIP_URL>`; `<SHIP_URL>` is the value of `$SHIP_URL`):

```json
{
  "family": "lab-shipping-worker",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/lab-ecs-exec-role",
  "taskRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/lab-shipping-task-role",
  "containerDefinitions": [{
    "name": "worker",
    "image": "public.ecr.aws/aws-cli/aws-cli:latest",
    "essential": true,
    "entryPoint": ["bash", "-c"],
    "command": ["while true; do OUT=$(aws sqs receive-message --queue-url $QUEUE_URL --wait-time-seconds 10 --max-number-of-messages 1 --query 'Messages[0].[ReceiptHandle,Body]' --output text); if [ \"$OUT\" != \"None\" ] && [ -n \"$OUT\" ]; then RH=$(echo \"$OUT\" | cut -f1); BODY=$(echo \"$OUT\" | cut -f2-); case \"$BODY\" in *$REJECT_MARKER*) echo \"ERROR cannot process: $BODY\";; *) echo \"shipping order: $BODY\"; sleep 5; aws sqs delete-message --queue-url $QUEUE_URL --receipt-handle \"$RH\";; esac; fi; done"],
    "environment": [
      { "name": "QUEUE_URL", "value": "<SHIP_URL>" },
      { "name": "AWS_DEFAULT_REGION", "value": "us-east-1" },
      { "name": "REJECT_MARKER", "value": "POISON" }
    ],
    "logConfiguration": { "logDriver": "awslogs", "options": {
      "awslogs-group": "/ecs/lab-shipping-worker",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "worker" } }
  }]
}
```

Register it and run one task in a default-VPC public subnet:

```bash
sed -i.bak "s#<ACCOUNT_ID>#$ACCOUNT_ID#g; s#<SHIP_URL>#$SHIP_URL#" taskdef.json
aws ecs register-task-definition --cli-input-json file://taskdef.json

SUBNET_A=$(aws ec2 describe-subnets --filters Name=default-for-az,Values=true \
  --query 'Subnets[0].SubnetId' --output text)
SG_DEFAULT=$(aws ec2 describe-security-groups --filters Name=group-name,Values=default \
  Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text) \
  --query 'SecurityGroups[0].GroupId' --output text)

aws ecs create-service --cluster lab-orders --service-name lab-shipping-worker \
  --task-definition lab-shipping-worker --desired-count 1 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_A],securityGroups=[$SG_DEFAULT],assignPublicIp=ENABLED}"

aws ecs wait services-stable --cluster lab-orders --services lab-shipping-worker
```

(The default security group has no inbound need here; the task only makes outbound calls.) Console check: ECS > Clusters > lab-orders > Services. Then publish a few orders and read the log:

```bash
for i in 1 2 3; do aws sns publish --topic-arn $TOPIC_ARN --message "{\"orderId\":\"o-$i\"}"; done
aws logs tail /ecs/lab-shipping-worker --since 5m
```

You should see `shipping order: {"orderId":"o-1"}` lines. Billing has no worker, so its messages simply wait (they are retained; this is the point of decoupling).

### Step 5: EventBridge scheduled rule to the billing queue

```bash
aws events put-rule --name lab-billing-reconcile --schedule-expression "rate(5 minutes)"

aws events put-targets --rule lab-billing-reconcile --targets \
  "Id=billing-queue,Arn=$BILL_ARN,Input='{\"type\":\"reconcile\"}'"

# after ~5 minutes:
aws sqs get-queue-attributes --queue-url $BILL_URL \
  --attribute-names ApproximateNumberOfMessages --query Attributes
```

The queue policy you attached in Step 2 already allows this rule's ARN. The same rule structure with `--event-pattern` would react to events from AWS services or custom sources instead of time. Disable it when done testing: `aws events disable-rule --name lab-billing-reconcile`.

### Step 6: Failure experiment, a poison message lands in the DLQ

```bash
aws sns publish --topic-arn $TOPIC_ARN --message 'POISON order with corrupt payload'

# watch the worker fail (it logs ERROR and does not delete)
aws logs tail /ecs/lab-shipping-worker --since 3m | grep -i error

# after roughly 90 seconds (3 receives x 30s visibility):
aws sqs get-queue-attributes --queue-url $SHIP_DLQ_URL \
  --attribute-names ApproximateNumberOfMessages
aws sqs receive-message --queue-url $SHIP_DLQ_URL --visibility-timeout 0 --query 'Messages[0].Body'
```

Expected: the shipping queue keeps serving good orders while the bad one is isolated in `lab-shipping-dlq`. Good messages were never blocked by it. Note the billing queue got its own copy; it has no worker, so it is unaffected.

Now ship the fix and redrive. Register a new task definition revision where the bug no longer exists (the marker no longer matches), update the service, and move the message back to the source queue:

```bash
sed -i.bak 's/"POISON"/"NEVER-MATCHES"/' taskdef.json
aws ecs register-task-definition --cli-input-json file://taskdef.json
aws ecs update-service --cluster lab-orders --service lab-shipping-worker --task-definition lab-shipping-worker
aws ecs wait services-stable --cluster lab-orders --services lab-shipping-worker

aws sqs start-message-move-task \
  --source-arn $SHIP_DLQ_ARN

aws sqs list-message-move-tasks --source-arn $SHIP_DLQ_ARN
aws logs tail /ecs/lab-shipping-worker --since 3m
```

With no `--destination-arn`, SQS returns messages to their original source queue. The log now shows the formerly poisoned order being shipped. Redrive without fixing the cause just sends the message round the loop again; always fix first.

### Step 7: Scale the workers by queue depth (step scaling)

Register the service as a scalable target and add a step scaling policy. Alarm threshold is 50 visible messages. Step bounds are relative to the alarm threshold: 0 to +50 above it adds one task, more than +50 above it adds two.

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/lab-orders/lab-shipping-worker \
  --min-capacity 1 --max-capacity 3

POLICY_ARN=$(aws application-autoscaling put-scaling-policy \
  --policy-name lab-shipping-scale-out \
  --service-namespace ecs --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/lab-orders/lab-shipping-worker \
  --policy-type StepScaling \
  --step-scaling-policy-configuration '{
    "AdjustmentType": "ChangeInCapacity",
    "Cooldown": 120,
    "MetricAggregationType": "Maximum",
    "StepAdjustments": [
      { "MetricIntervalLowerBound": 0,  "MetricIntervalUpperBound": 50, "ScalingAdjustment": 1 },
      { "MetricIntervalLowerBound": 50, "ScalingAdjustment": 2 }
    ]}' --query PolicyARN --output text)

aws cloudwatch put-metric-alarm --alarm-name lab-shipping-backlog-high \
  --namespace AWS/SQS --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=lab-shipping \
  --statistic Maximum --period 60 --evaluation-periods 1 \
  --threshold 50 --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions $POLICY_ARN
```

Generate a backlog (the worker takes about 5 seconds per message, so 120 orders is about 10 minutes for one task):

```bash
for i in $(seq 1 120); do
  aws sns publish --topic-arn $TOPIC_ARN --message "{\"orderId\":\"bulk-$i\"}" > /dev/null
done

aws cloudwatch describe-alarms --alarm-names lab-shipping-backlog-high --query 'MetricAlarms[0].StateValue'
aws ecs describe-services --cluster lab-orders --services lab-shipping-worker \
  --query 'services[0].[desiredCount,runningCount]'
```

SQS metrics are published at about one-minute granularity, so allow a few minutes for the alarm to enter `ALARM` and for desired count to grow. This lab only configures scale-out; a matching scale-in policy (alarm on a low backlog) is a stretch goal. Production tip: AWS documents a target tracking approach using a custom "backlog per task" metric, which is simpler to tune than step scaling; step scaling is shown here because the exam asks about alarm-driven scaling.

### Step 8: FIFO vs Standard decision note

| Question | Standard | FIFO |
|---|---|---|
| Ordering | Best-effort | Strict per message group (`MessageGroupId`) |
| Duplicates | At-least-once, duplicates possible, so consumers must be idempotent | Exactly-once processing within a 5-minute deduplication window |
| Throughput | Nearly unlimited | Much lower (default and high-throughput modes, check current quotas) |
| Name | any | must end in `.fifo` |
| SNS fan-out | Standard topic to standard queues | FIFO topic to FIFO queues only |
| DLQ | Standard DLQ | FIFO DLQ |

For ShelfCo, orders are independent, so Standard queues plus idempotent consumers (keyed on `orderId`) are right. You would pick FIFO if, for example, "update address" and "cancel" events for one order must apply in sequence: use `orderId` as the message group.

### Step 9: What would you change for production

- Run Fargate tasks in private subnets with a NAT Gateway or VPC endpoints for SQS, ECR and CloudWatch Logs (accept the hourly cost), and pin the image by digest from your own ECR repository.
- Replace the shell loop with real code that batches (up to 10 messages), extends visibility timeouts for long jobs, and is idempotent.
- Encrypt topic and queues with KMS, set a CloudWatch alarm on the DLQ depth that pages someone, and set DLQ retention longer than the source queue.
- Use target tracking on backlog per task, plus scale-in, Fargate Spot for tolerant workers, and ECS deployment circuit breakers.
- Evaluate Lambda with an SQS event source mapping if the work is short and bursty; it is less code than a container service.

## Validation

- One `sns publish` produced one message in each of `lab-billing` and `lab-shipping`.
- Worker logs show shipped orders; the shipping task role cannot touch the billing queue (optionally prove with `aws iam simulate-principal-policy --policy-source-arn <TASK_ROLE_ARN> --action-names sqs:ReceiveMessage --resource-arns $BILL_ARN`, which should return `implicitDeny`).
- The poison message appeared in `lab-shipping-dlq` after three failed receives, and `list-message-move-tasks` shows the redrive completed, after which the worker logged it as shipped.
- The EventBridge rule delivered a `{"type":"reconcile"}` message to `lab-billing` roughly every 5 minutes.
- The backlog alarm entered `ALARM` and the service desired count rose above 1 (up to 3).

## Cleanup

Delete in dependency order. Fargate tasks bill while running, so do the ECS part first.

```bash
# 1. Scaling, alarm, ECS
aws application-autoscaling delete-scaling-policy --policy-name lab-shipping-scale-out \
  --service-namespace ecs --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/lab-orders/lab-shipping-worker
aws application-autoscaling deregister-scalable-target \
  --service-namespace ecs --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/lab-orders/lab-shipping-worker
aws cloudwatch delete-alarms --alarm-names lab-shipping-backlog-high
aws ecs update-service --cluster lab-orders --service lab-shipping-worker --desired-count 0
aws ecs delete-service --cluster lab-orders --service lab-shipping-worker --force
aws ecs delete-cluster --cluster lab-orders
for td in $(aws ecs list-task-definitions --family-prefix lab-shipping-worker --query 'taskDefinitionArns[]' --output text); do
  aws ecs deregister-task-definition --task-definition $td > /dev/null
done
aws logs delete-log-group --log-group-name /ecs/lab-shipping-worker

# 2. EventBridge
aws events remove-targets --rule lab-billing-reconcile --ids billing-queue
aws events delete-rule --name lab-billing-reconcile

# 3. SNS subscriptions and topic (deleting the topic removes its subscriptions)
aws sns delete-topic --topic-arn $TOPIC_ARN

# 4. SQS queues
for u in $BILL_URL $SHIP_URL $SHIP_DLQ_URL \
         $(aws sqs get-queue-url --queue-name lab-billing-dlq --query QueueUrl --output text); do
  aws sqs delete-queue --queue-url $u
done

# 5. IAM
aws iam delete-role-policy --role-name lab-shipping-task-role --policy-name shipping-queue-only
aws iam delete-role --role-name lab-shipping-task-role
aws iam detach-role-policy --role-name lab-ecs-exec-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name lab-ecs-exec-role
```

Task definitions are only deregistered (they become INACTIVE and cost nothing). Confirm in the ECS console that no tasks are running.

## Exam Takeaway

- SNS plus SQS fan-out decouples one producer from many consumers, each with its own buffer, retry behavior and failure isolation. The SQS queue policy must allow the SNS topic. (Domain 2: Design Resilient Architectures.)
- Standard queues give at-least-once delivery, so consumers must be idempotent. FIFO gives ordering and deduplication at lower throughput. (Domain 2, Domain 3.)
- Visibility timeout, `maxReceiveCount` and a DLQ isolate poison messages; DLQ redrive returns messages after the fix; alarm on DLQ depth. (Domain 2.)
- Scale consumers on queue depth (`ApproximateNumberOfMessagesVisible`) via CloudWatch alarms and Application Auto Scaling; this is the classic "decouple and scale independently" answer. (Domain 2, Domain 3.)
- Fargate removes server management: task execution role pulls images/logs, the task role is the application's identity. Least privilege per task. (Domain 1: Secure Architectures; Domain 3.)
- EventBridge handles schedules and event routing from AWS services and SaaS; SNS is pub/sub fan-out; SQS is a buffer. Choose by purpose. (Domain 2, Domain 3.)
- Avoid hourly-billed networking (NAT Gateway, interface endpoints) unless the security requirement justifies it, and clean up Fargate tasks. (Domain 4: Cost-Optimized Architectures.)

## Stretch Goals

1. Add a scale-in step policy with a low-backlog alarm (`LessThanOrEqualToThreshold`), and observe the service returning to 1 task.
2. Add an SNS subscription filter policy so that only orders with `"priority":"express"` (published as a message attribute) reach a third queue.
3. Rebuild the pipeline with FIFO: `lab-orders.fifo` topic, two `.fifo` queues, `MessageGroupId=orderId`, and test deduplication with a repeated `MessageDeduplicationId`.
4. Replace the Fargate worker with a Lambda function using an SQS event source mapping, and compare operational effort and cost.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
