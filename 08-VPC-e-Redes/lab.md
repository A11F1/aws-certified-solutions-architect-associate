# Guided Lab: Build a Custom VPC with Public and Private Subnets, Security Groups vs NACLs, and an S3 Gateway Endpoint

## Objective

Build a VPC from scratch, make one subnet public and one private, compare stateful security groups with stateless NACLs, and give the private subnet access to S3 through a gateway endpoint with no NAT Gateway.

## Scenario

You are designing the network for a small web app. The web server sits in a public subnet, a backend instance sits in a private subnet with no internet route, and the backend must read from S3 without traversing the internet.

## Services Used

- Amazon VPC (subnets, route tables, internet gateway, security groups, network ACLs, gateway endpoint)
- Amazon EC2 (two t3.micro or t2.micro instances)
- SSH through the public instance as a jump host
- Amazon S3, AWS IAM
- AWS CLI v2

## Prerequisites

- Personal AWS account, non-root IAM identity with VPC, EC2, IAM, and S3 permissions, region `us-east-1`.
- AWS CLI v2 and an SSH client.
- An EC2 key pair `<YOUR_KEY_PAIR>` in us-east-1.
- An S3 bucket `<YOUR_BUCKET_NAME>` with one small test file (`aws s3 cp hello.txt s3://<YOUR_BUCKET_NAME>/hello.txt`).

## Expected Cost

A VPC, subnets, route tables, internet gateway, security groups, NACLs, and gateway endpoints are free. Two small instances cost roughly $0.02 per hour together (and are Free Tier eligible for 750 hours per month on new accounts), so a one-hour lab costs pennies. A public IPv4 address is billed at about $0.005 per hour per address. Do not create a NAT Gateway in this lab: it costs about $0.045 per hour plus data processing, which is the common surprise bill. Stop or terminate instances when finished.

## Step by Step

### Step 1: Create the VPC and subnets

Console: VPC > Your VPCs > Create VPC > VPC only > name `lab08-vpc`, IPv4 CIDR `10.0.0.0/16`. Then Subnets > Create subnet: `lab08-public` `10.0.1.0/24` in us-east-1a and `lab08-private` `10.0.2.0/24` in us-east-1a.

```bash
VPC=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab08-vpc}]' \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames

PUB=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab08-public}]' \
  --query Subnet.SubnetId --output text)
PRIV=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.2.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab08-private}]' \
  --query Subnet.SubnetId --output text)
aws ec2 modify-subnet-attribute --subnet-id $PUB --map-public-ip-on-launch
```

Note that AWS reserves 5 IPs in every subnet, so a /24 gives 251 usable addresses.

### Step 2: Internet gateway and route tables

What makes a subnet "public" is a route to an internet gateway, not its name.

Console: VPC > Internet gateways > Create > attach to `lab08-vpc`. Route tables > Create `lab08-public-rt`, add route `0.0.0.0/0` to the IGW, and associate with `lab08-public`. The private subnet stays on the main route table (local route only).

```bash
IGW=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC

PUBRT=$(aws ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $PUBRT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $PUBRT --subnet-id $PUB

PRIVRT=$(aws ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
aws ec2 associate-route-table --route-table-id $PRIVRT --subnet-id $PRIV
```

### Step 3: Security groups and an instance profile for S3 access

```bash
WEBSG=$(aws ec2 create-security-group --group-name lab08-web --description "web" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $WEBSG --protocol tcp --port 80 --cidr 0.0.0.0/0

APPSG=$(aws ec2 create-security-group --group-name lab08-app --description "app" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $APPSG --protocol tcp --port 8080 --source-group $WEBSG
```

The app group allows traffic only from members of the web group, a reference by security group rather than CIDR. Security groups only have allow rules and are stateful.

Create an instance profile so the instances can read the S3 bucket:

```bash
cat > ec2-trust.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
JSON
aws iam create-role --role-name lab08-ec2-role --assume-role-policy-document file://ec2-trust.json
aws iam attach-role-policy --role-name lab08-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam create-instance-profile --instance-profile-name lab08-ec2-profile
aws iam add-role-to-instance-profile --instance-profile-name lab08-ec2-profile --role-name lab08-ec2-role
```

### Step 4: Launch the instances

Console: EC2 > Launch instance, use Amazon Linux 2023, t3.micro, network `lab08-vpc`, subnet as listed, matching security group, and IAM instance profile `lab08-ec2-profile`. Public instance: auto-assign public IP enabled. Private instance: disabled.

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)
PUBI=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $PUB \
  --key-name <YOUR_KEY_PAIR> --security-group-ids $WEBSG --iam-instance-profile Name=lab08-ec2-profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab08-public}]' \
  --query 'Instances[0].InstanceId' --output text)
PRIVI=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $PRIV \
  --key-name <YOUR_KEY_PAIR> --security-group-ids $APPSG --iam-instance-profile Name=lab08-ec2-profile --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab08-private}]' \
  --query 'Instances[0].InstanceId' --output text)
```

The private instance has no internet route, so reach it through the public instance as a jump host. Add SSH from the web group to the app group, and pass `--key-name <YOUR_KEY_PAIR>` to both `run-instances` commands (also allow SSH from `<YOUR_IP>/32` on the web group):

```bash
aws ec2 authorize-security-group-ingress --group-id $APPSG --protocol tcp --port 22 --source-group $WEBSG
aws ec2 authorize-security-group-ingress --group-id $WEBSG --protocol tcp --port 22 --cidr <YOUR_IP>/32
ssh -A -i <YOUR_KEY_PAIR>.pem ec2-user@<PUBLIC_IP>
ssh ec2-user@<PRIVATE_IP>      # from the public instance
```

(Session Manager would also work, but from a subnet without internet it needs three billed interface endpoints, which this lab avoids.)

### Step 5: Add an S3 gateway endpoint

Console: VPC > Endpoints > Create endpoint > service `com.amazonaws.us-east-1.s3` type Gateway > VPC `lab08-vpc` > select the private route table `lab08-private-rt`.

```bash
aws ec2 create-vpc-endpoint --vpc-id $VPC --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway --route-table-ids $PRIVRT
aws ec2 describe-route-tables --route-table-ids $PRIVRT --query 'RouteTables[0].Routes'
```

The route table gets a new entry targeting a prefix list (`pl-...`) with the `vpce-...` target. Gateway endpoints are free and work only for S3 and DynamoDB. From the private instance, `aws s3 ls s3://<YOUR_BUCKET_NAME> --region us-east-1` now works with no internet route, using the instance profile credentials. To prove the endpoint is the reason, remove it (`aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <VPCE_ID>`), rerun the command and watch it hang, then recreate the endpoint.

### Step 6: Experiment with NACLs (stateless) versus security groups (stateful)

Create a custom NACL and associate it with the public subnet. A new custom NACL denies everything.

```bash
NACL=$(aws ec2 create-network-acl --vpc-id $VPC --query NetworkAcl.NetworkAclId --output text)
aws ec2 create-network-acl-entry --network-acl-id $NACL --ingress --rule-number 100 \
  --protocol tcp --port-range From=80,To=80 --cidr-block 0.0.0.0/0 --rule-action allow
ASSOC=$(aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=$PUB \
  --query 'NetworkAcls[0].Associations[?SubnetId==`'$PUB'`].NetworkAclAssociationId' --output text)
aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $NACL
```

Install a web server on the public instance and curl its public IP:

```bash
# on the instance
sudo dnf install -y httpd && sudo systemctl enable --now httpd
echo "lab08" | sudo tee /var/www/html/index.html
# from your laptop
curl -m 5 http://<PUBLIC_IP>/
```

It times out: the inbound rule allows port 80, but there is no outbound rule, so the response (to an ephemeral port) is dropped because NACLs are stateless. Fix it by allowing outbound ephemeral ports:

```bash
aws ec2 create-network-acl-entry --network-acl-id $NACL --egress --rule-number 100 \
  --protocol tcp --port-range From=1024,To=65535 --cidr-block 0.0.0.0/0 --rule-action allow
curl -m 5 http://<PUBLIC_IP>/
```

Now it works. Add a deny rule with a lower number (for example `--ingress --rule-number 50 ... --rule-action deny --cidr-block <YOUR_IP>/32`) to see that NACLs evaluate rules in number order, first match wins, and that you can block a single IP, which a security group cannot do.

## Validation

- The public instance serves the page over HTTP; the private instance has no public IP.
- `describe-route-tables` shows `0.0.0.0/0` to the IGW on the public table, only a local route plus the S3 prefix-list route on the private table.
- Requests time out with only an inbound NACL rule, and succeed after adding the outbound ephemeral rule.
- S3 listing works from inside the VPC via the endpoint route.

## Cleanup

Delete in dependency order:

```bash
aws ec2 terminate-instances --instance-ids $PUBI $PRIVI
aws ec2 wait instance-terminated --instance-ids $PUBI $PRIVI
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <VPCE_ID>
aws ec2 replace-network-acl-association --association-id <NACL_ASSOC_ID> --network-acl-id <DEFAULT_NACL_ID>
aws ec2 delete-network-acl --network-acl-id $NACL
aws ec2 delete-security-group --group-id $APPSG
aws ec2 delete-security-group --group-id $WEBSG
aws ec2 disassociate-route-table --association-id <PUBRT_ASSOC_ID>
aws ec2 delete-route-table --route-table-id $PUBRT
aws ec2 disassociate-route-table --association-id <PRIVRT_ASSOC_ID>
aws ec2 delete-route-table --route-table-id $PRIVRT
aws ec2 detach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC
aws ec2 delete-internet-gateway --internet-gateway-id $IGW
aws ec2 delete-subnet --subnet-id $PUB
aws ec2 delete-subnet --subnet-id $PRIV
aws ec2 delete-vpc --vpc-id $VPC
aws iam remove-role-from-instance-profile --instance-profile-name lab08-ec2-profile --role-name lab08-ec2-role
aws iam delete-instance-profile --instance-profile-name lab08-ec2-profile
aws iam detach-role-policy --role-name lab08-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name lab08-ec2-role
```

Delete the gateway endpoint (above) before the route tables.

## Exam Takeaway

A subnet is public only if its route table has a route to an internet gateway and its instances have public IPs. Security groups are stateful, allow-only, and attach to ENIs; NACLs are stateless, support allow and deny, are evaluated in rule order, and attach to subnets, so they are the tool for blocking specific IPs. For private subnets that only need S3 or DynamoDB, a gateway endpoint is free and keeps traffic on the AWS network, whereas a NAT Gateway costs hourly plus per-GB and should be chosen only when instances need general internet egress.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
