# Week 2 Main Lab: Highly Available Web Tier with ALB and Auto Scaling

## Objective
Build a self-healing, self-scaling web tier: a launch template, an Application Load Balancer, and an Auto Scaling group spread across two Availability Zones. You will watch traffic spread across instances, scale out under CPU load, terminate an instance and see it replaced, and simulate the loss of an AZ.

Estimated time: 80 to 110 minutes.

## Modules Covered
- [04 - EC2 Compute](../04-Computacao-EC2/README.md)
- [05 - High Availability and Scalability](../05-Alta-Disponibilidade-e-Escalabilidade/README.md)

## Scenario and Architecture
A startup is launching a marketing site. A single EC2 instance went down during a demo. The requirement: no single point of failure, automatic replacement of unhealthy servers, automatic scale-out during traffic spikes, and no direct internet access to the instances (only the load balancer is public).

```
                    Internet
                       |
                 +-----v------+
                 |    ALB     |  SG: 80 from 0.0.0.0/0
                 | (2 subnets)|  Listener HTTP:80
                 +--+------+--+
                    |      |      Target group (health check: GET /)
          +---------+      +----------+
          |                           |
  +-------v--------+         +--------v-------+
  | AZ us-east-1a  |         | AZ us-east-1b  |
  | subnet A       |         | subnet B       |
  | EC2 t3.micro   |         | EC2 t3.micro   |   SG: 80 only from ALB SG
  | page: id + AZ  |         | page: id + AZ  |
  +-------^--------+         +--------^-------+
          |                           |
          +--------- Auto Scaling group ---------+
              min 2 / desired 2 / max 4
              health check type ELB
              target tracking: avg CPU 40%
              built from Launch Template (AL2023, IMDSv2 required)
```

## Services Used
- Amazon EC2 (launch template, AMI from SSM public parameter, security groups, instance profile)
- Elastic Load Balancing (Application Load Balancer, target group)
- Amazon EC2 Auto Scaling (group, target tracking policy)
- Amazon CloudWatch (EC2 and ALB metrics, scaling alarms)
- AWS Systems Manager (Run Command to generate CPU load, no SSH needed)

## Prerequisites
- Completed Week 1 baseline (admin identity with MFA, budget alarm) or equivalent.
- AWS CLI v2, region `us-east-1`, default VPC with default subnets in at least two AZs.
- Your IAM identity can use EC2, ELB, Auto Scaling, IAM (create role), and SSM.

```bash
export AWS_REGION=us-east-1
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=is-default,Values=true \
  --query 'Vpcs[0].VpcId' --output text)

# Pick two default subnets in different AZs
aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=default-for-az,Values=true \
  --query 'Subnets[].[SubnetId,AvailabilityZone]' --output text
export SUBNET_A=<SUBNET_ID_IN_us-east-1a>
export SUBNET_B=<SUBNET_ID_IN_us-east-1b>
```

## Expected Cost
Approximate and region-dependent; confirm on the EC2 and Elastic Load Balancing pricing pages. Rough order of magnitude: 2 to 4 `t3.micro` instances at about 0.01 USD per hour each, plus an ALB at roughly 0.02 to 0.03 USD per hour (hourly charge plus LCU usage), plus public IPv4 charges for the ALB and any instance public IPs. A 2-hour session is typically well under 1 USD. The ALB and the instances bill by the hour until you delete them, so finish the lab and run Cleanup on the same day. `t3` instances run in unlimited CPU credit mode by default, so sustained high CPU during the load test can add a small surplus-credit charge. Free Tier eligibility depends on account age; check your Billing console.

## Step by Step

### Step 1: Security groups
```bash
export ALB_SG=$(aws ec2 create-security-group --group-name lab-w2-alb-sg \
  --description "ALB public HTTP" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

export WEB_SG=$(aws ec2 create-security-group --group-name lab-w2-web-sg \
  --description "Web tier, only from ALB" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $WEB_SG \
  --protocol tcp --port 80 --source-group $ALB_SG
```
The instance security group references the ALB security group, not an IP range. This is the standard layered pattern. No SSH rule exists at all.

### Step 2: Instance role for Session Manager and Run Command
```bash
cat > ec2-trust.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
JSON
aws iam create-role --role-name lab-w2-web-role --assume-role-policy-document file://ec2-trust.json
aws iam attach-role-policy --role-name lab-w2-web-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam create-instance-profile --instance-profile-name lab-w2-web-profile
aws iam add-role-to-instance-profile --instance-profile-name lab-w2-web-profile \
  --role-name lab-w2-web-role
```
Wait about 10 seconds for IAM propagation before using the profile.

### Step 3: Write the user data
The page reads the instance ID and AZ from the metadata service using IMDSv2 (token required).

```bash
cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
IID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
cat > /var/www/html/index.html <<HTML
<html><body style="font-family:sans-serif">
<h1>Week 2 web tier</h1>
<p>Instance: $IID</p>
<p>Availability Zone: $AZ</p>
</body></html>
HTML
systemctl enable --now httpd
EOF
export USERDATA_B64=$(base64 < userdata.sh | tr -d '\n')
```

### Step 4: Create the launch template
The AMI is resolved at launch time from the public SSM parameter for the latest Amazon Linux 2023 image, so you do not hard-code an AMI ID.

```bash
cat > lt-data.json <<JSON
{
  "ImageId": "resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64",
  "InstanceType": "t3.micro",
  "SecurityGroupIds": ["${WEB_SG}"],
  "IamInstanceProfile": {"Name": "lab-w2-web-profile"},
  "MetadataOptions": {"HttpTokens": "required", "HttpEndpoint": "enabled"},
  "UserData": "${USERDATA_B64}",
  "TagSpecifications": [{
    "ResourceType": "instance",
    "Tags": [{"Key": "Name", "Value": "lab-w2-web"}]
  }]
}
JSON

aws ec2 create-launch-template --launch-template-name lab-w2-lt \
  --launch-template-data file://lt-data.json
```
Console equivalent: **EC2 > Launch Templates > Create launch template**. Look at the "Advanced details" section to see the instance profile, metadata options, and user data fields you just set.

### Step 5: Create the ALB, target group, and listener
```bash
export ALB_ARN=$(aws elbv2 create-load-balancer --name lab-w2-alb \
  --subnets $SUBNET_A $SUBNET_B --security-groups $ALB_SG \
  --scheme internet-facing --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

export TG_ARN=$(aws elbv2 create-target-group --name lab-w2-tg \
  --protocol HTTP --port 80 --vpc-id $VPC_ID --target-type instance \
  --health-check-path / --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 2 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

export ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)
echo $ALB_DNS
```
Shorten deregistration delay for a faster lab (default is 300 seconds):

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

### Step 6: Create the Auto Scaling group across two AZs
Default-VPC subnets assign public IPs by default, which lets the instances reach the package repository without a NAT Gateway. Inbound is still only allowed from the ALB. (Production would use private subnets plus NAT or VPC endpoints.)

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name lab-w2-asg \
  --launch-template LaunchTemplateName=lab-w2-lt,Version='$Latest' \
  --min-size 2 --max-size 4 --desired-capacity 2 \
  --vpc-zone-identifier "${SUBNET_A},${SUBNET_B}" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB --health-check-grace-period 120 \
  --tags Key=Name,Value=lab-w2-web,PropagateAtLaunch=true
```
`--health-check-type ELB` is the important part: the ASG now replaces instances that fail the load balancer health check, not only instances whose EC2 status check fails.

Wait for healthy targets (about 2 to 4 minutes):

```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State]' --output text
```

### Step 7: Verify load distribution
```bash
for i in $(seq 1 10); do curl -s http://$ALB_DNS | grep -E "Instance|Zone" | tr '\n' ' '; echo; done
```
You should see two different instance IDs and both AZs. Round-robin is the default ALB routing algorithm for HTTP; the exact order is not guaranteed.

### Step 8: Add a target tracking scaling policy
```bash
cat > tt-policy.json <<'JSON'
{
  "TargetValue": 40.0,
  "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
  "DisableScaleIn": false
}
JSON

aws autoscaling put-scaling-policy \
  --auto-scaling-group-name lab-w2-asg \
  --policy-name lab-w2-cpu-tt \
  --policy-type TargetTrackingScaling \
  --estimated-instance-warmup 120 \
  --target-tracking-configuration file://tt-policy.json
```
Target tracking creates and manages the CloudWatch alarms for you (look in **CloudWatch > Alarms**, names start with `TargetTracking-`). Remember that scale-out alarms react relatively quickly while scale-in is deliberately slower (it needs about 15 one-minute datapoints below target).

### Step 9: Load test and watch scale-out
Generate CPU load on the running instances without SSH, by using Run Command on instances tagged by the ASG. The load runs for 10 minutes and stops by itself.

```bash
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Name,Values=lab-w2-web" \
  --parameters 'commands=["for i in $(seq $(nproc)); do timeout 600 sh -c \"while :; do :; done\" & done; sleep 1"]' \
  --comment "w2 cpu load"
```

If the command reports instances as not registered with SSM, wait a minute or two (the SSM agent needs time to register) or check that the role has `AmazonSSMManagedInstanceCore`.

Watch the group (repeat every minute; expect scale-out after roughly 3 to 7 minutes):

```bash
aws autoscaling describe-scaling-activities --auto-scaling-group-name lab-w2-asg \
  --max-items 5 --query 'Activities[].[StartTime,StatusCode,Description]' --output text
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-w2-asg \
  --query 'AutoScalingGroups[0].[DesiredCapacity,length(Instances)]' --output text
```

Console: **EC2 > Auto Scaling groups > lab-w2-asg > Activity** and **Monitoring**. Desired capacity should rise above 2 (maximum 4). New instances do not carry the load loop, so average CPU falls back as capacity grows.

### Step 10: Failure experiment A, terminate an instance and watch self-healing
```bash
INSTANCE_ID=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-w2-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)

aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# Meanwhile the site must keep answering
for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code}\n" http://$ALB_DNS; sleep 2; done

aws autoscaling describe-scaling-activities --auto-scaling-group-name lab-w2-asg --max-items 3
```
Expected: HTTP 200 throughout (the other instance serves traffic), and a new activity "Launching a new EC2 instance" because the group fell below desired capacity. Self-healing restores the count.

### Step 11: Failure experiment B, simulate loss of one AZ
This is a simulation, not a real AZ outage: you remove subnet B from the group so the ASG must run everything in AZ A. First, stop scaling from interfering by setting a fixed size.

```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name lab-w2-asg \
  --min-size 2 --desired-capacity 2

aws autoscaling update-auto-scaling-group --auto-scaling-group-name lab-w2-asg \
  --vpc-zone-identifier "${SUBNET_A}"
```

Watch it rebalance (a few minutes):

```bash
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-w2-asg \
  --query 'AutoScalingGroups[0].Instances[].[InstanceId,AvailabilityZone,LifecycleState]' --output table
for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code} " http://$ALB_DNS; sleep 3; done; echo
```
Expected: instances in the removed AZ are terminated and replaced in AZ A, the group still has 2 instances, and requests keep succeeding (brief errors are possible if the ALB has no healthy target at a moment, which is what you would investigate). Restore both AZs:

```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name lab-w2-asg \
  --vpc-zone-identifier "${SUBNET_A},${SUBNET_B}"
```
The group rebalances back across both AZs over the next minutes.

### Step 12: Pricing models comparison
Fill the "this web tier" column with your own judgment.

| Model | How it works | Typical savings (approximate; verify on AWS pricing pages) | Interruption risk | Fit for this web tier |
|---|---|---|---|---|
| On-Demand | Pay per second/hour, no commitment | Baseline (0%) | None | Good for the lab and unpredictable load |
| Reserved Instances / Savings Plans | 1 or 3 year commitment to usage (Compute Savings Plans flexible across instance families, Regions, Lambda, Fargate; EC2 Instance Savings Plans tied to a family and Region) | Up to roughly 66% to 72% depending on plan and term | None | Cover the steady baseline (the 2 always-on instances) |
| Spot Instances | Spare capacity at deep discount; AWS can reclaim with a 2-minute notice | Up to roughly 90% | Yes | Good for the burst instances above the baseline in an ASG mixed instances policy, not for the only copy of state |
| Dedicated Hosts / Capacity Reservations | Reserve physical server or capacity in an AZ | Varies | None | Only for licensing or guaranteed capacity needs |

A common production design: Savings Plan for the baseline, On-Demand plus Spot (mixed instances policy with several instance types) for the elastic portion.

## Validation
- `curl http://$ALB_DNS` returns a page with an instance ID and AZ, and repeated requests show both AZs.
- Targets in the target group are `healthy`; the instance security group has no rule open to `0.0.0.0/0` (try `curl http://<instance-public-ip>` and expect a timeout).
- Scaling activity history shows a scale-out during the load test and a replacement after termination.
- During both failure experiments, the ALB kept returning HTTP 200 (allowing for short blips).
- The ASG has `HealthCheckType: ELB` (`aws autoscaling describe-auto-scaling-groups`).

## Cleanup
Delete in dependency order, and do it today. The ALB and instances bill hourly.

```bash
# 1. Auto Scaling group (terminates its instances)
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name lab-w2-asg --force-delete
# wait until instances are gone
aws ec2 describe-instances --filters Name=tag:Name,Values=lab-w2-web \
  Name=instance-state-name,Values=pending,running,shutting-down \
  --query 'Reservations[].Instances[].InstanceId' --output text

# 2. Load balancer, listener goes with it, then the target group
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
sleep 60
aws elbv2 delete-target-group --target-group-arn $TG_ARN

# 3. Launch template
aws ec2 delete-launch-template --launch-template-name lab-w2-lt

# 4. Security groups (instance SG first, because it references the ALB SG)
aws ec2 delete-security-group --group-id $WEB_SG
aws ec2 delete-security-group --group-id $ALB_SG

# 5. IAM
aws iam remove-role-from-instance-profile --instance-profile-name lab-w2-web-profile \
  --role-name lab-w2-web-role
aws iam delete-instance-profile --instance-profile-name lab-w2-web-profile
aws iam detach-role-policy --role-name lab-w2-web-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name lab-w2-web-role
```
If a security group delete says a dependency exists, wait a minute for network interfaces to release and retry. Scaling alarms created by target tracking are removed when the policy and group are deleted; confirm in **CloudWatch > Alarms**.

### What Would You Change for Production
- Put instances in private subnets, with NAT Gateways (one per AZ) or VPC endpoints for outbound access, and keep only the ALB in public subnets.
- Terminate HTTPS on the ALB with an ACM certificate, redirect HTTP to HTTPS, and attach AWS WAF if needed.
- Bake a custom AMI (or use a deployment pipeline) instead of installing software in user data on every launch; use instance refresh for rolling updates.
- Use a mixed instances policy (On-Demand base plus Spot), an ALB request-count-per-target policy alongside CPU, and a minimum of two instances across two or more AZs.
- Add CloudWatch alarms and notifications, access logs on the ALB, and make the data tier (RDS Multi-AZ, DynamoDB) highly available too. A web tier alone is not a highly available application.

## Exam Takeaway
- Domain 2, Design Resilient Architectures: multi-AZ ASG plus ALB is the canonical pattern; ELB health checks on the ASG replace unhealthy instances; the ALB stops routing to unhealthy targets; stateless instances are what make scaling and replacement safe.
- Know the load balancer types: ALB for HTTP/HTTPS (layer 7, path/host routing), NLB for TCP/UDP and static IPs (layer 4), GWLB for appliances. Security group chaining (instance SG allows only the ALB SG) is a common answer.
- Scaling policies: target tracking (simplest, keep a metric near a value), step, scheduled, and predictive. Cooldown and warm-up exist to avoid thrashing.
- Domain 3, High-Performing Architectures: choose instance family by workload; use scaling to match demand.
- Domain 4, Cost-Optimized Architectures: Savings Plans for steady baseline, Spot for fault-tolerant or flexible workloads, On-Demand for spiky or short-term unknowns; right-size and scale in when idle.

## Stretch Goals
1. Add a second scaling policy using `ALBRequestCountPerTarget` (requires a `ResourceLabel` built from the ALB and target group ARN suffixes) and compare behavior.
2. Convert the group to a mixed instances policy with an On-Demand base of 2 and Spot for additional capacity, using two or three instance types.
3. Add a scheduled action that raises `min-size` before a known traffic window, then lowers it afterward.
4. Add an HTTPS listener with an ACM certificate on a domain you own and redirect HTTP to HTTPS.


---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
