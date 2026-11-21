# Guided Lab: VPC Peering and an Interface Endpoint (PrivateLink), Plus a Hybrid Connectivity Walkthrough

## Objective

Connect two VPCs with VPC peering and test routing, create an interface VPC endpoint so a private instance reaches an AWS service without internet, and compare Transit Gateway, Direct Connect and Site-to-Site VPN in a no-create walkthrough.

## Scenario

An app VPC (`10.0.0.0/16`) must talk privately to a shared-services VPC (`10.1.0.0/16`) and call AWS Systems Manager privately, with no NAT Gateway and no public IPs.

## Services Used

- Amazon VPC (VPC peering, route tables, security groups)
- Interface VPC endpoints (AWS PrivateLink)
- Amazon EC2, AWS Systems Manager Session Manager

## Prerequisites

- AWS CLI v2, Region `us-east-1`.
- IAM permissions for VPC, EC2, SSM and IAM instance profiles.
- An instance profile with `AmazonSSMManagedInstanceCore` (create role `lab-ssm-role` and profile `lab-ssm-profile` if you have none).
- Latest Amazon Linux 2023 AMI ID (`aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text`).

## Expected Cost

Two `t3.micro` instances are Free Tier eligible for new accounts (about 0.01 USD/h each otherwise). VPC peering itself is free (cross-AZ data is charged at normal rates). Interface endpoints cost about 0.01 USD per AZ-hour plus data processing, so create them in one subnet and delete them within the hour. Do NOT create a NAT Gateway (about 0.045 USD/h plus data), and do NOT create Transit Gateway attachments (about 0.05 USD/h each) or Site-to-Site VPN connections (about 0.05 USD/h) unless you delete them immediately. Direct Connect is walkthrough only.

## Step by Step

### Step 1: Create two VPCs with non-overlapping CIDRs

Console: VPC > Create VPC > "VPC and more" is fine, but use two separate VPCs: `app-vpc` 10.0.0.0/16 and `shared-vpc` 10.1.0.0/16, one private subnet each, no NAT Gateway, no internet gateway.

```bash
APP=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
SHARED=$(aws ec2 create-vpc --cidr-block 10.1.0.0/16 --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $APP --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $SHARED --enable-dns-hostnames
APP_SN=$(aws ec2 create-subnet --vpc-id $APP --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query Subnet.SubnetId --output text)
SH_SN=$(aws ec2 create-subnet --vpc-id $SHARED --cidr-block 10.1.1.0/24 --availability-zone us-east-1a --query Subnet.SubnetId --output text)
```

Peering cannot work with overlapping CIDRs; try 10.0.0.0/16 for both in your head and note why it would fail.

### Step 2: Create and accept the peering connection

Console: VPC > Peering connections > Create peering connection > requester `app-vpc`, accepter `shared-vpc` > then Actions > Accept request.

```bash
PCX=$(aws ec2 create-vpc-peering-connection --vpc-id $APP --peer-vpc-id $SHARED --query VpcPeeringConnection.VpcPeeringConnectionId --output text)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PCX
```

### Step 3: Add routes on both sides

Console: Route tables > select each VPC's route table > Edit routes > add the other VPC's CIDR with target the peering connection.

```bash
APP_RT=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$APP --query "RouteTables[0].RouteTableId" --output text)
SH_RT=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$SHARED --query "RouteTables[0].RouteTableId" --output text)
aws ec2 create-route --route-table-id $APP_RT --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PCX
aws ec2 create-route --route-table-id $SH_RT --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id $PCX
```

### Step 4: Launch test instances and endpoints for Session Manager

Security group `lab-sg` in each VPC: allow ICMP from the other VPC's CIDR and all traffic from itself (for endpoint HTTPS use TCP 443 from 10.0.0.0/16).

```bash
SG_APP=$(aws ec2 create-security-group --group-name lab-sg --description lab --vpc-id $APP --query GroupId --output text)
SG_SH=$(aws ec2 create-security-group --group-name lab-sg --description lab --vpc-id $SHARED --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_APP --protocol icmp --port -1 --cidr 10.1.0.0/16
aws ec2 authorize-security-group-ingress --group-id $SG_SH  --protocol icmp --port -1 --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id $SG_APP --protocol tcp --port 443 --cidr 10.0.0.0/16
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3.micro --subnet-id $APP_SN --security-group-ids $SG_APP --iam-instance-profile Name=lab-ssm-profile --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=app}]'
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3.micro --subnet-id $SH_SN --security-group-ids $SG_SH --iam-instance-profile Name=lab-ssm-profile --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=shared}]'
```

Because the private instances have no internet path, Session Manager needs three interface endpoints in the app VPC: `ssm`, `ssmmessages`, `ec2messages`. Create them (console: VPC > Endpoints > Create endpoint > AWS services, enable Private DNS):

```bash
for S in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint --vpc-id $APP --vpc-endpoint-type Interface \
    --service-name com.amazonaws.us-east-1.$S --subnet-ids $APP_SN \
    --security-group-ids $SG_APP --private-dns-enabled
done
```

For the shared instance, create the same three endpoints in `$SHARED` if you want to open a shell there, or just test one direction from `app`.

### Step 5: Test routing

Console: Systems Manager > Session Manager > Start session on `app` (wait a few minutes until it shows as managed).

```bash
ping -c 3 <SHARED_INSTANCE_PRIVATE_IP>      # works through the peering
```

Failure experiment: delete the route in the app route table and ping again (it times out), then restore it.

```bash
aws ec2 delete-route --route-table-id $APP_RT --destination-cidr-block 10.1.0.0/16
aws ec2 create-route --route-table-id $APP_RT --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PCX
```

Transitivity check: peering is not transitive. A third VPC peered to `shared-vpc` could not reach `app-vpc` through it.

### Step 6: Walkthrough only - Transit Gateway, VPN and Direct Connect (do not create)

| Option | Use when | Typical latency/bandwidth | Cost model | Setup time |
|---|---|---|---|---|
| VPC peering | A few VPCs, simple 1:1, no transitive routing | AWS backbone, no bandwidth cap | Free; data charged | Minutes |
| Transit Gateway | Many VPCs/on-prem, hub-and-spoke, transitive routing | Up to 50 Gbps per VPC attachment | Per attachment-hour plus per GB | Minutes |
| PrivateLink | Expose one service privately, even with overlapping CIDRs | Per endpoint | Per endpoint-hour plus per GB | Minutes |
| Site-to-Site VPN | Quick encrypted link over the internet | Up to ~1.25 Gbps per tunnel | Per connection-hour plus data out | Minutes to hours |
| Direct Connect | Consistent latency, high bandwidth, private link | 1/10/100 Gbps dedicated | Port-hour plus data out | Weeks |

Console tour (read only): VPC > Transit gateways, VPC > Site-to-Site VPN connections > Create (stop at the form), Direct Connect > Connections. Note that Direct Connect is not encrypted by default; run a VPN over it, or use MACsec on supported ports, if encryption is required. A Direct Connect gateway plus a Transit Gateway lets one connection reach many VPCs.

## Validation

- Peering status is `active` and ping across VPCs succeeds with the routes in place.
- Ping fails after deleting the route and succeeds after restoring it.
- Session Manager reaches the private instance with no NAT Gateway and no public IP.
- `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$APP` lists three `available` endpoints.

## Cleanup

```bash
aws ec2 terminate-instances --instance-ids <APP_INSTANCE_ID> <SHARED_INSTANCE_ID>
aws ec2 wait instance-terminated --instance-ids <APP_INSTANCE_ID> <SHARED_INSTANCE_ID>
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <ENDPOINT_ID_1> <ENDPOINT_ID_2> <ENDPOINT_ID_3>
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PCX
aws ec2 delete-security-group --group-id $SG_APP
aws ec2 delete-security-group --group-id $SG_SH
aws ec2 delete-subnet --subnet-id $APP_SN
aws ec2 delete-subnet --subnet-id $SH_SN
aws ec2 delete-vpc --vpc-id $APP
aws ec2 delete-vpc --vpc-id $SHARED
```

Endpoint network interfaces take a minute to disappear; retry security group and subnet deletion if they report dependencies. Remove `lab-ssm-role` and `lab-ssm-profile` if you created them only for this lab.

## Exam Takeaway

Pick by scale and requirement: VPC peering for a few VPCs (non-overlapping CIDRs, no transitive routing, route tables on both sides), Transit Gateway for hub-and-spoke at scale, PrivateLink to expose one service privately (works even with overlapping CIDRs), Site-to-Site VPN for fast encrypted setup over the internet, and Direct Connect for consistent bandwidth and latency at the cost of weeks of lead time. Interface endpoints (ENI, PrivateLink, hourly cost) differ from gateway endpoints (S3 and DynamoDB only, free).

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
