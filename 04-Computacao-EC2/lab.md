# Guided Lab: Launch a Secure EC2 Web Server with an Instance Profile and Session Manager

## Objective
Launch a Free Tier EC2 instance running a web server via user data, give it S3 access through an instance profile (no keys on disk), connect without SSH using Session Manager, and observe stop/terminate and storage behavior.

## Scenario
A small web server must serve a page, read from S3, and be administrable without opening port 22 or storing credentials.

## Services Used
- Amazon EC2, EBS, Security Groups
- IAM role and instance profile
- AWS Systems Manager Session Manager
- Amazon S3 (read access only)

## Prerequisites
- Module 01 setup, region `us-east-1`, default VPC.
- AWS CLI v2. The Session Manager plugin is needed only for `aws ssm start-session`; the console works without it.

## Expected Cost
Within Free Tier (750 hours/month of an eligible instance type, 30 GB EBS) or about $0.01/hour otherwise. Avoid leaving the instance running, leaving an unattached Elastic IP (charged), and creating NAT Gateways or interface VPC endpoints (hourly charges). Public IPv4 addresses are billed hourly outside the Free Tier.

## Step by Step

### Step 1: Create the instance role and profile
```bash
cat > ec2-trust.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
JSON
aws iam create-role --role-name lab-ec2-role --assume-role-policy-document file://ec2-trust.json
aws iam attach-role-policy --role-name lab-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name lab-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam create-instance-profile --instance-profile-name lab-ec2-profile
aws iam add-role-to-instance-profile --instance-profile-name lab-ec2-profile --role-name lab-ec2-role
```
(The console creates the instance profile automatically when you create an EC2 role.)

### Step 2: Create a security group (HTTP only, no SSH)
```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SG=$(aws ec2 create-security-group --group-name lab-web-sg --description "lab web" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 80 --cidr 0.0.0.0/0
```
Console: **EC2** > **Security Groups**. There is no inbound rule for port 22.

### Step 3: Launch the instance with user data
```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)
cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
echo "<h1>Hello from $(hostname -f)</h1>" > /var/www/html/index.html
systemctl enable --now httpd
EOF
aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --iam-instance-profile Name=lab-ec2-profile --security-group-ids $SG \
  --user-data file://userdata.sh \
  --metadata-options HttpTokens=required \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-web}]'
```
Console equivalent: **Launch instance** > Amazon Linux 2023 > an instance type marked Free Tier eligible (`t2.micro` or `t3.micro`) > proceed without a key pair > select the SG > Advanced details: IAM instance profile, Metadata version "V2 only", and User data. `HttpTokens=required` enforces IMDSv2.

### Step 4: Browse and connect without SSH
```bash
aws ec2 describe-instances --filters Name=tag:Name,Values=lab-web Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].[InstanceId,PublicIpAddress]' --output text
```
Open `http://<PUBLIC_IP>`. Then **Connect** > **Session Manager** (allow a couple of minutes for the SSM agent to register), or run `aws ssm start-session --target <INSTANCE_ID>`.

### Step 5: Prove credentials come from the role
Inside the session:
```bash
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/
aws s3 ls     # works, with no access keys configured
```
Then, from your workstation, detach `AmazonS3ReadOnlyAccess` from the role and retry `aws s3 ls` in the session after a minute or two: it fails.

### Step 6: Stop versus terminate
```bash
aws ec2 stop-instances --instance-ids <INSTANCE_ID>
aws ec2 wait instance-stopped --instance-ids <INSTANCE_ID>
aws ec2 start-instances --instance-ids <INSTANCE_ID>
aws ec2 describe-instances --instance-ids <INSTANCE_ID> \
  --query 'Reservations[].Instances[].[PublicIpAddress,BlockDeviceMappings]'
```
The public IP changes after a stop/start (no Elastic IP), while the EBS root volume data persists. The root volume has `DeleteOnTermination: true` by default, so terminating deletes it.

## Validation
- The web page shows the instance hostname.
- You have a shell through Session Manager while port 22 is closed.
- `aws s3 ls` works with no stored keys and fails once the policy is detached.

## Cleanup
```bash
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
aws ec2 wait instance-terminated --instance-ids <INSTANCE_ID>
aws ec2 delete-security-group --group-id $SG
aws iam remove-role-from-instance-profile --instance-profile-name lab-ec2-profile --role-name lab-ec2-role
aws iam delete-instance-profile --instance-profile-name lab-ec2-profile
aws iam detach-role-policy --role-name lab-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam detach-role-policy --role-name lab-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name lab-ec2-role
```
If you already detached the S3 policy in Step 5, that detach reports an error you can ignore.

## Exam Takeaway
Use an IAM role (instance profile) instead of embedded access keys, and Session Manager instead of SSH plus key pairs. Security groups are stateful allow-only rules. EBS root volumes survive a stop but are deleted on termination by default; instance store data is lost on stop. For purchasing: On-Demand for unpredictable short workloads, Savings Plans or Reserved Instances for steady usage, Spot for fault-tolerant interruptible work.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
