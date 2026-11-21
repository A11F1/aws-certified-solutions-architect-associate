# Guided Lab: Three-Tier Mini Web App (VPC, ALB + Auto Scaling, RDS)

## Objective

Build the classic three-tier layout: an Application Load Balancer in public subnets, an Auto Scaling group of two `t3.micro` web instances, and a small RDS MySQL instance in private subnets that only the web tier can reach. Security groups reference each other instead of CIDR ranges.

## Scenario

A small company needs a highly available web front end with a managed relational database. The web tier must survive the loss of one Availability Zone and replace failed instances automatically. The database must never be reachable from the internet.

```
Internet -> ALB (public subnets, 2 AZs) -> ASG: 2x t3.micro -> RDS MySQL (private subnets)
```

## Services Used

- Amazon VPC (subnets, route tables, internet gateway, security groups)
- Elastic Load Balancing (Application Load Balancer, target group)
- Amazon EC2 and EC2 Auto Scaling (launch template)
- Amazon RDS for MySQL

## Prerequisites

- Personal AWS account, region `us-east-1`, admin-level IAM identity, AWS CLI v2.
- Comfort with CIDR notation.
- A plan to finish and clean up in one sitting.

## Expected Cost

WARNING: this lab creates resources that bill by the hour even when idle. Rough us-east-1 on-demand numbers, which you should confirm on the pricing pages:

- Application Load Balancer: about $0.0225 per hour plus LCU usage (roughly $16 per month if left running).
- RDS `db.t3.micro` plus storage: about $0.017 per hour plus 20 GB storage. Free Tier covers this only for eligible accounts; newer accounts may use the credit-based free plan, so check the Billing console.
- 2x `t3.micro` EC2: about $0.0104 per hour each unless covered by Free Tier.
- Public IPv4 addresses: about $0.005 per hour each (the ALB uses one per subnet, and so does each instance with a public IP).

A 2-hour session should cost well under $1 to a few dollars. Left running for a month it costs real money. To avoid surprises: do NOT add a NAT gateway (about $0.045 per hour plus data), do NOT enable Multi-AZ on RDS, do NOT leave it overnight, and set a budget alert first (Billing > Budgets > Create budget, $5). Cleanup must be done the same day.

## Step by Step

### Step 1: Create the VPC, subnets, and internet gateway

Console: VPC > Create VPC > "VPC and more" can do this in one wizard (choose 2 AZs, 2 public, 2 private, NAT gateways: None, VPC endpoints: None). The CLI version:

```bash
VPC=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames

PUB1=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query Subnet.SubnetId --output text)
PUB2=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --query Subnet.SubnetId --output text)
PRV1=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.11.0/24 --availability-zone us-east-1a --query Subnet.SubnetId --output text)
PRV2=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.12.0/24 --availability-zone us-east-1b --query Subnet.SubnetId --output text)

IGW=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC

RT=$(aws ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $RT --subnet-id $PUB1
aws ec2 associate-route-table --route-table-id $RT --subnet-id $PUB2
aws ec2 modify-subnet-attribute --subnet-id $PUB1 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUB2 --map-public-ip-on-launch
```

A subnet is "public" because its route table has a route to the internet gateway. The private subnets keep the VPC's default local-only route table.

Design note: to avoid NAT gateway cost, the web instances run in the public subnets so they can install packages. In production you would put them in private subnets and use a NAT gateway or VPC endpoints. The security group below still blocks direct internet access to them.

### Step 2: Create the security groups (chained by reference)

```bash
ALB_SG=$(aws ec2 create-security-group --group-name lab-alb-sg --description "ALB" --vpc-id $VPC --query GroupId --output text)
WEB_SG=$(aws ec2 create-security-group --group-name lab-web-sg --description "Web" --vpc-id $VPC --query GroupId --output text)
DB_SG=$(aws ec2 create-security-group --group-name lab-db-sg --description "DB" --vpc-id $VPC --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --source-group $ALB_SG
aws ec2 authorize-security-group-ingress --group-id $DB_SG  --protocol tcp --port 3306 --source-group $WEB_SG
```

### Step 3: Create the RDS instance in private subnets

Start this first because it takes 5 to 10 minutes.

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name lab-db-subnets \
  --db-subnet-group-description "Private subnets for lab" \
  --subnet-ids $PRV1 $PRV2

aws rds create-db-instance \
  --db-instance-identifier lab-mysql \
  --engine mysql \
  --db-instance-class db.t3.micro \
  --allocated-storage 20 \
  --master-username admin \
  --manage-master-user-password \
  --db-subnet-group-name lab-db-subnets \
  --vpc-security-group-ids $DB_SG \
  --no-publicly-accessible \
  --no-multi-az \
  --backup-retention-period 1
```

`--manage-master-user-password` stores the password in AWS Secrets Manager so no secret ever appears in your shell history. A secret in Secrets Manager adds a small monthly charge (about $0.40 per month, prorated); it is deleted along with the instance. Console: RDS > Create database > Standard create > MySQL > Free tier template, "Don't connect to an EC2 compute resource", Public access: No.

### Step 4: Create the launch template

```bash
cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>Hello from $AZ</h1>" > /var/www/html/index.html
systemctl enable --now httpd
EOF

aws ec2 create-launch-template --launch-template-name lab-web-lt \
  --launch-template-data "{
    \"ImageId\": \"resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64\",
    \"InstanceType\": \"t3.micro\",
    \"SecurityGroupIds\": [\"$WEB_SG\"],
    \"UserData\": \"$(base64 < userdata.sh | tr -d '\n')\"
  }"
```

### Step 5: Create the ALB, target group, and listener

```bash
TG=$(aws elbv2 create-target-group --name lab-web-tg --protocol HTTP --port 80 \
  --vpc-id $VPC --health-check-path / --query 'TargetGroups[0].TargetGroupArn' --output text)

ALB=$(aws elbv2 create-load-balancer --name lab-alb --type application \
  --subnets $PUB1 $PUB2 --security-groups $ALB_SG \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG
```

### Step 6: Create the Auto Scaling group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name lab-web-asg \
  --launch-template LaunchTemplateName=lab-web-lt,Version='$Latest' \
  --min-size 2 --max-size 2 --desired-capacity 2 \
  --vpc-zone-identifier "$PUB1,$PUB2" \
  --target-group-arns $TG \
  --health-check-type ELB --health-check-grace-period 120
```

`--health-check-type ELB` makes the group replace instances that fail the load balancer's health check, not just EC2 status checks.

### Step 7: Test it

```bash
DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB --query 'LoadBalancers[0].DNSName' --output text)
for i in 1 2 3 4 5 6; do curl -s http://$DNS; done
```

Wait 2 to 4 minutes for targets to become healthy.

## Validation

- `aws elbv2 describe-target-health --target-group-arn $TG` shows 2 targets `healthy`, one per AZ.
- Repeated `curl` calls show "Hello from us-east-1a" and "us-east-1b".
- Resilience test: terminate one instance (`aws ec2 terminate-instances --instance-ids <INSTANCE_ID>`) and watch the ASG launch a replacement while the ALB keeps answering.
- `aws rds describe-db-instances --db-instance-identifier lab-mysql --query 'DBInstances[0].[DBInstanceStatus,PubliclyAccessible,Endpoint.Address]'` shows `available` and `false`. The endpoint resolves only to a private IP and cannot be reached from your laptop.

## Cleanup

Do this the same day, in this order (dependencies block deletion otherwise):

```bash
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name lab-web-asg --force-delete
aws rds delete-db-instance --db-instance-identifier lab-mysql \
  --skip-final-snapshot --delete-automated-backups
aws elbv2 delete-load-balancer --load-balancer-arn $ALB
# wait ~1 minute for the ALB and instances to go away
aws elbv2 delete-target-group --target-group-arn $TG
aws ec2 delete-launch-template --launch-template-name lab-web-lt
# wait until the DB is deleted, then:
aws rds delete-db-subnet-group --db-subnet-group-name lab-db-subnets
aws ec2 delete-security-group --group-id $DB_SG
aws ec2 delete-security-group --group-id $WEB_SG
aws ec2 delete-security-group --group-id $ALB_SG
aws ec2 delete-subnet --subnet-id $PUB1; aws ec2 delete-subnet --subnet-id $PUB2
aws ec2 delete-subnet --subnet-id $PRV1; aws ec2 delete-subnet --subnet-id $PRV2
aws ec2 delete-route-table --route-table-id $RT
aws ec2 detach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC
aws ec2 delete-internet-gateway --internet-gateway-id $IGW
aws ec2 delete-vpc --vpc-id $VPC
```

Then confirm in the console that no load balancers, RDS instances, EC2 instances, or Elastic IPs remain, and check Billing the next day.

## Exam Takeaway

Know the standard tiering pattern: ALB in public subnets, application tier in an Auto Scaling group across at least two AZs, database in private subnets with no public access. Allow traffic by referencing security groups (ALB SG -> web SG -> DB SG) instead of IP ranges. Use ELB health checks on the ASG so unhealthy instances are replaced. Note what this lab deliberately skipped, because the exam asks for it: RDS Multi-AZ for database high availability (a standby in another AZ with automatic failover), read replicas for read scaling (not HA), and NAT gateways or VPC endpoints when private instances need outbound access.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
