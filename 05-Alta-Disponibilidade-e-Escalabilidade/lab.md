# Guided Lab: Highly Available Web Tier with an ALB and Auto Scaling Group

## Objective
Deploy an Application Load Balancer in front of an Auto Scaling group spread across two Availability Zones, then terminate an instance and watch the group self-heal.

## Scenario
A stateless web page must survive the loss of an instance or an AZ and add capacity under CPU load.

## Services Used
- Elastic Load Balancing (Application Load Balancer, target group)
- EC2 Auto Scaling (launch template, Auto Scaling group, target tracking policy)
- Amazon EC2, Security Groups

## Prerequisites
- Module 01 setup, `us-east-1`, default VPC with default subnets in at least two AZs.
- AWS CLI v2.

## Expected Cost
An ALB costs roughly $0.0225 per hour plus LCU charges, so a one-hour lab is a few cents, plus two `t3.micro` instances (Free Tier hours or about $0.01/hour each). Delete the ALB and ASG right after the lab; an idle load balancer keeps billing. No NAT Gateway is used because the instances run in default public subnets.

## Step by Step

### Step 1: Gather network info and create security groups
```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC Name=availability-zone,Values=us-east-1a,us-east-1b \
  --query 'Subnets[].SubnetId' --output text | tr '\t' ',')
ALB_SG=$(aws ec2 create-security-group --group-name lab-alb-sg --description alb --vpc-id $VPC --query GroupId --output text)
WEB_SG=$(aws ec2 create-security-group --group-name lab-web-sg --description web --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --source-group $ALB_SG
```
The web security group accepts traffic only from the ALB's security group.

### Step 2: Create the launch template
```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)
cat > ud.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
echo "<h1>$ID in $AZ</h1>" > /var/www/html/index.html
systemctl enable --now httpd
EOF
aws ec2 create-launch-template --launch-template-name lab-lt --launch-template-data \
 "{\"ImageId\":\"$AMI\",\"InstanceType\":\"t3.micro\",\"SecurityGroupIds\":[\"$WEB_SG\"],\"UserData\":\"$(base64 < ud.sh | tr -d '\n')\"}"
```
Console: **EC2** > **Launch Templates** > **Create launch template** with the same values.

### Step 3: Create the target group and ALB
```bash
TG=$(aws elbv2 create-target-group --name lab-tg --protocol HTTP --port 80 --vpc-id $VPC \
  --health-check-path / --query 'TargetGroups[0].TargetGroupArn' --output text)
ALB=$(aws elbv2 create-load-balancer --name lab-alb --subnets ${SUBNETS//,/ } \
  --security-groups $ALB_SG --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 create-listener --load-balancer-arn $ALB --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG
```
(`${SUBNETS//,/ }` is bash/zsh syntax; in zsh use `${=SUBNETS//,/ }` or list the subnet IDs by hand.)

### Step 4: Create the Auto Scaling group
```bash
aws autoscaling create-auto-scaling-group --auto-scaling-group-name lab-asg \
  --launch-template LaunchTemplateName=lab-lt,Version='$Latest' \
  --min-size 2 --max-size 4 --desired-capacity 2 \
  --vpc-zone-identifier "$SUBNETS" \
  --target-group-arns $TG --health-check-type ELB --health-check-grace-period 120
aws autoscaling put-scaling-policy --auto-scaling-group-name lab-asg \
  --policy-name cpu50 --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":50.0}'
```
`--health-check-type ELB` lets the group replace instances that fail the load balancer health check, not only EC2 status checks.

### Step 5: Test load balancing
```bash
DNS=$(aws elbv2 describe-load-balancers --names lab-alb --query 'LoadBalancers[0].DNSName' --output text)
aws elbv2 describe-target-health --target-group-arn $TG   # wait for both targets to be healthy
for i in 1 2 3 4 5 6; do curl -s http://$DNS; done
```
Responses alternate between instance IDs and AZs.

### Step 6: Failure experiment
Run a request loop in one terminal (`while true; do curl -s -m 2 http://$DNS; sleep 1; done`), then terminate one instance:
```bash
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
aws autoscaling describe-scaling-activities --auto-scaling-group-name lab-asg --max-items 3
```
The ALB stops routing to the terminated instance, the site stays up on the survivor, and the ASG launches a replacement to restore the desired capacity of 2.

## Validation
- The ALB DNS name returns pages from two different AZs.
- After the termination, the activity history shows a new launch and the group returns to 2 healthy instances.
- The target group shows both targets `healthy`.

## Cleanup
Delete in dependency order (ASG first, then the load balancer, then the target group, template, and security groups):
```bash
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name lab-asg --force-delete
aws elbv2 delete-load-balancer --load-balancer-arn $ALB
aws elbv2 wait load-balancers-deleted --load-balancer-arns $ALB
aws elbv2 delete-target-group --target-group-arn $TG
aws ec2 delete-launch-template --launch-template-name lab-lt
# once the instances are gone (a few minutes):
aws ec2 delete-security-group --group-id $WEB_SG
aws ec2 delete-security-group --group-id $ALB_SG
```
Confirm in the console that no load balancers or instances remain.

## Exam Takeaway
High availability means multiple AZs plus a load balancer plus an Auto Scaling group that replaces failed capacity. ELB health checks on the ASG catch application-level failures that EC2 status checks miss. Chain security groups (the web tier accepts only the ALB's SG) instead of using IP ranges. Use an ALB for HTTP/HTTPS layer-7 routing and an NLB for TCP/UDP, extreme performance, or static IPs; target tracking is the simplest scaling policy. Keep instances stateless so scaling and replacement are safe.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
