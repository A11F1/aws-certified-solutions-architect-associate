# Guided Lab: Private Connectivity Costs and Troubleshooting, VPC Peering, Flow Logs and Reachability Analyzer

## Objective

Connect two VPCs with VPC peering, capture traffic with VPC Flow Logs, diagnose a broken path with VPC Reachability Analyzer, and compare the cost of NAT Gateway, gateway endpoints, and interface endpoints.

## Scenario

An app team in VPC A needs to call an internal service in VPC B. Traffic must stay private, you must prove which packets are accepted or rejected, and the team must understand why S3 access should use a gateway endpoint rather than a NAT Gateway.

## Services Used

- Amazon VPC (peering, route tables, security groups, flow logs)
- Amazon CloudWatch Logs
- VPC Reachability Analyzer
- Amazon EC2 (two t3.micro instances)
- AWS CLI v2

## Prerequisites

- Personal AWS account, non-root IAM identity with VPC, EC2, logs, and IAM permissions, region `us-east-1`.
- Basic VPC knowledge from the module 08 lab. Two non-overlapping CIDRs are required, so this lab uses `10.10.0.0/16` and `10.20.0.0/16`.
- An EC2 key pair `<YOUR_KEY_PAIR>` in us-east-1 and your public IP as `<YOUR_IP>`.

## Expected Cost

Peering connections and flow log configuration are free to create. Same-AZ peering data transfer is free; cross-AZ and cross-region data is billed, which is negligible here. Flow Logs delivered to CloudWatch Logs are billed per GB ingested (about $0.50 per GB), so a short test costs pennies. Reachability Analyzer charges about $0.10 per analysis. Two t3.micro instances cost about $0.02 per hour together. Reference prices to compare (verify on the pricing page): NAT Gateway about $0.045 per hour (about $32 per month) plus $0.045 per GB processed; interface endpoint about $0.01 per hour per AZ plus $0.01 per GB; gateway endpoints for S3 and DynamoDB are free. Do not create a NAT Gateway in this lab; use the numbers for a paper comparison.

## Step by Step

### Step 1: Create two VPCs, subnets, and instances

Console: VPC > Create VPC > "VPC and more" is faster, but the CLI keeps it explicit.

```bash
VPCA=$(aws ec2 create-vpc --cidr-block 10.10.0.0/16 --query Vpc.VpcId --output text)
VPCB=$(aws ec2 create-vpc --cidr-block 10.20.0.0/16 --query Vpc.VpcId --output text)
SUBA=$(aws ec2 create-subnet --vpc-id $VPCA --cidr-block 10.10.1.0/24 --availability-zone us-east-1a --query Subnet.SubnetId --output text)
SUBB=$(aws ec2 create-subnet --vpc-id $VPCB --cidr-block 10.20.1.0/24 --availability-zone us-east-1a --query Subnet.SubnetId --output text)
IGWA=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGWA --vpc-id $VPCA
RTA=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPCA --query 'RouteTables[0].RouteTableId' --output text)
RTB=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPCB --query 'RouteTables[0].RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTA --destination-cidr-block 0.0.0.0/0 --gateway-id $IGWA
aws ec2 modify-subnet-attribute --subnet-id $SUBA --map-public-ip-on-launch
```

Security groups and instances (instance A is reachable for you via public IP, instance B is private and only allows ICMP and TCP 80 from VPC A):

```bash
SGA=$(aws ec2 create-security-group --group-name lab09-a --description a --vpc-id $VPCA --query GroupId --output text)
SGB=$(aws ec2 create-security-group --group-name lab09-b --description b --vpc-id $VPCB --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SGA --protocol tcp --port 22 --cidr <YOUR_IP>/32
aws ec2 authorize-security-group-ingress --group-id $SGB --protocol icmp --port -1 --cidr 10.10.0.0/16
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text)
IA=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $SUBA --security-group-ids $SGA --key-name <YOUR_KEY_PAIR> --query 'Instances[0].InstanceId' --output text)
IB=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro --subnet-id $SUBB --security-group-ids $SGB --no-associate-public-ip-address --query 'Instances[0].InstanceId' --output text)
```

### Step 2: Turn on VPC Flow Logs to CloudWatch Logs

Console: VPC > select `$VPCB` > Flow logs > Create flow log > Filter: All > Destination: CloudWatch Logs > create a log group `/lab09/flowlogs` and an IAM role as the wizard offers.

```bash
aws logs create-log-group --log-group-name /lab09/flowlogs
cat > fl-trust.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"vpc-flow-logs.amazonaws.com"},"Action":"sts:AssumeRole"}]}
JSON
aws iam create-role --role-name lab09-flowlogs --assume-role-policy-document file://fl-trust.json
cat > fl-perm.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["logs:CreateLogStream","logs:PutLogEvents","logs:DescribeLogGroups","logs:DescribeLogStreams"],"Resource":"*"}]}
JSON
aws iam put-role-policy --role-name lab09-flowlogs --policy-name logs --policy-document file://fl-perm.json
aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPCB --traffic-type ALL \
  --log-destination-type cloud-watch-logs --log-group-name /lab09/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::<ACCOUNT_ID>:role/lab09-flowlogs
```

Flow logs take roughly 10 minutes to start producing records and are not real time.

### Step 3: Diagnose the missing route with Reachability Analyzer (before peering)

Console: VPC > Reachability Analyzer > Create and analyze path > source: instance A, destination: instance B, protocol TCP, port 80 > Create.

```bash
PATH_ID=$(aws ec2 create-network-insights-path --source $IA --destination $IB \
  --protocol tcp --destination-port 80 --query NetworkInsightsPath.NetworkInsightsPathId --output text)
AN=$(aws ec2 start-network-insights-analysis --network-insights-path-id $PATH_ID \
  --query NetworkInsightsAnalysis.NetworkInsightsAnalysisId --output text)
aws ec2 describe-network-insights-analysis --network-insights-analysis-ids $AN \
  --query 'NetworkInsightsAnalyses[0].[Status,NetworkPathFound,Explanations]'
```

Expected: `NetworkPathFound: false`, with an explanation such as no route to the destination or no interconnect between the VPCs. This is the failure experiment: you get a precise reason without sending any packets.

### Step 4: Create and accept the peering connection, add routes

Console: VPC > Peering connections > Create > requester `lab09 A`, accepter `lab09 B` (same account and region) > Actions > Accept request.

```bash
PCX=$(aws ec2 create-vpc-peering-connection --vpc-id $VPCA --peer-vpc-id $VPCB \
  --query VpcPeeringConnection.VpcPeeringConnectionId --output text)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PCX
aws ec2 create-route --route-table-id $RTA --destination-cidr-block 10.20.0.0/16 --vpc-peering-connection-id $PCX
aws ec2 create-route --route-table-id $RTB --destination-cidr-block 10.10.0.0/16 --vpc-peering-connection-id $PCX
aws ec2 authorize-security-group-ingress --group-id $SGB --protocol tcp --port 80 --cidr 10.10.0.0/16
```

Both route tables need a route; peering does nothing until you add them. Rerun the analysis from Step 3. The result should now be `NetworkPathFound: true` and list the hops through `pcx-...`. Peering is not transitive: if VPC B were also peered with a third VPC C, A could not reach C through B.

### Step 5: Generate traffic and read the flow logs

SSH to instance A, then:

```bash
ping -c 3 <PRIVATE_IP_OF_B>
curl -m 3 http://<PRIVATE_IP_OF_B>:80   # no web server running, so connection is rejected or times out
curl -m 3 http://<PRIVATE_IP_OF_B>:8080 # port not allowed by SG, expect a timeout
```

After a few minutes, query CloudWatch Logs:

```bash
aws logs filter-log-events --log-group-name /lab09/flowlogs --filter-pattern "REJECT" --limit 10
aws logs filter-log-events --log-group-name /lab09/flowlogs --filter-pattern "ACCEPT" --limit 10
```

Each default-format record has fields in this order: version, account-id, interface-id, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes, start, end, action, log-status. Port 8080 shows `REJECT` (blocked by the security group), ICMP and port 80 show `ACCEPT`. A flow log record shows what happened (accepted or rejected), but not which rule decided it.

### Step 6: Cost comparison worksheet (no resources to create)

Imagine a private subnet pulling 500 GB per month from S3 and DynamoDB:

- NAT Gateway: about $32 hourly cost plus about $22.50 data processing, roughly $55 per month per AZ, plus public egress charges for non-AWS traffic.
- S3 and DynamoDB gateway endpoints: $0.
- Interface endpoint for a service such as Secrets Manager: about $7.30 per month per AZ plus $0.01 per GB.

Try it in the console: VPC > Endpoints > Create endpoint, pick `com.amazonaws.us-east-1.s3`, note the Gateway versus Interface types offered, and cancel without creating.

## Validation

- Reachability Analyzer reports `NetworkPathFound: false` before peering and `true` after routes are added.
- ICMP from A to B succeeds only after the peering routes exist in both tables.
- Flow logs show `REJECT` for the blocked port and `ACCEPT` for allowed traffic.
- You can state which endpoint type is free and which options bill hourly.

## Cleanup

```bash
aws ec2 terminate-instances --instance-ids $IA $IB
aws ec2 wait instance-terminated --instance-ids $IA $IB
aws ec2 delete-flow-logs --flow-log-ids <FLOW_LOG_ID>
aws logs delete-log-group --log-group-name /lab09/flowlogs
aws iam delete-role-policy --role-name lab09-flowlogs --policy-name logs
aws iam delete-role --role-name lab09-flowlogs
aws ec2 delete-network-insights-analysis --network-insights-analysis-id $AN
aws ec2 delete-network-insights-path --network-insights-path-id $PATH_ID
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PCX
aws ec2 delete-security-group --group-id $SGA
aws ec2 delete-security-group --group-id $SGB
aws ec2 detach-internet-gateway --internet-gateway-id $IGWA --vpc-id $VPCA
aws ec2 delete-internet-gateway --internet-gateway-id $IGWA
aws ec2 delete-subnet --subnet-id $SUBA
aws ec2 delete-subnet --subnet-id $SUBB
aws ec2 delete-vpc --vpc-id $VPCA
aws ec2 delete-vpc --vpc-id $VPCB
```

If a VPC delete fails, check for leftover ENIs, endpoints, or custom route table associations.

## Exam Takeaway

Peering needs non-overlapping CIDRs, routes in both VPCs, and is not transitive; for many VPCs or transitive routing, choose Transit Gateway, and for exposing a single service to many consumers, choose PrivateLink. For cost, S3 and DynamoDB should use free gateway endpoints instead of a NAT Gateway, while other AWS services use interface endpoints (PrivateLink) that bill hourly and per GB. For troubleshooting, Reachability Analyzer answers configuration-level "why can't A reach B" questions without traffic, while Flow Logs show actual accepted and rejected traffic after the fact but never capture packet contents.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
