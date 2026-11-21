# Review Questions

## Question 1

A scenario requires the right decision on VPC. Which approach fits best?

A) Apply a generic solution without considering subnets.
B) Use subnets with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of VPC with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take subnets into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine route tables with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of VPC with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Manual handling is slow, error-prone, and does not scale. Combining automation with observability is what actually reduces operational risk.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 3

What should you do to keep cost under control without losing technical quality?

A) Always pick the cheapest option without validating the requirement.
B) Size based on actual usage and take advantage of NAT Gateway when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of VPC with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Cost is only one constraint. The cheapest option can break availability, performance, or security requirements, so it must be validated against the scenario first.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 4

Which alternative tends to be incorrect on the exam for this topic?
A) The one that meets the dominant requirement with the least complexity.
B) The one that ignores endpoints signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of VPC with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
This is the pattern that is usually correct on the exam. The question asks which alternative tends to be incorrect, so this is the opposite of what is being asked.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 5

Which review approach improves accuracy in this module?

A) Review only definitions without scenario practice.
B) Practice questions focused on SG vs NACL and Transit Gateway with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of VPC with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company runs application servers in private subnets of a VPC. The servers must download operating system patches from the internet, but they must not be reachable from the internet. The company wants a managed solution that does not require patching or scaling infrastructure. What should a solutions architect do?

A) Add a route to an internet gateway in the private subnet route table and assign public IP addresses to the instances
B) Create a NAT gateway in a public subnet and add a default route from the private subnet route table to the NAT gateway
C) Create a NAT gateway in a private subnet and add a default route from the private subnet route table to the NAT gateway
D) Create an egress-only internet gateway and route 0.0.0.0/0 from the private subnets to it

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
A NAT gateway is a managed, highly available (within an AZ) service that lets instances in private subnets initiate outbound connections while blocking unsolicited inbound connections. It must be placed in a public subnet, which has a route to an internet gateway, and the private subnet route table sends 0.0.0.0/0 to it.

**Why option A is wrong:**
Routing to an internet gateway and assigning public IPs makes the instances publicly addressable, so they could be reached from the internet if security groups allow it. That breaks the "not reachable" requirement.

**Why option C is wrong:**
A NAT gateway in a private subnet has no route to an internet gateway, so it cannot forward traffic to the internet. (A private NAT gateway is intended for connecting to other VPCs or on-premises networks.)

**Why option D is wrong:**
An egress-only internet gateway only works for IPv6 traffic. It cannot handle a 0.0.0.0/0 IPv4 route.

</details>

## Question 7

A web tier runs in a public subnet. Security monitoring identified a specific /24 CIDR range sending malicious traffic to the web servers. The security team wants to block this range at the subnet level immediately, while all other internet users remain able to connect on port 443. What should be done?

A) Add a deny rule for the CIDR range to the security group attached to the web servers
B) Add a blackhole route for the CIDR range to the public subnet route table
C) Enable AWS Shield Standard and add the CIDR range to its deny list
D) Add an inbound deny rule for the CIDR range to the subnet's network ACL, with a rule number lower than the allow rule for port 443

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Network ACLs are stateless, subnet-level filters that support both allow and deny rules, evaluated in ascending rule number order. A deny rule for the malicious CIDR with a lower number than the allow rule for port 443 is processed first, so that range is blocked while everyone else is still allowed.

**Why option A is wrong:**
Security groups support only allow rules. You cannot create an explicit deny rule, so you cannot block a specific range while allowing the rest of the internet.

**Why option B is wrong:**
Route tables control where outbound traffic is sent. They do not filter inbound traffic from a source range, and a blackhole route would at most affect return traffic in a way that is not a reliable control.

**Why option C is wrong:**
AWS Shield Standard automatically protects against common network and transport layer DDoS attacks, but it has no customer-managed deny list for specific CIDR ranges.

</details>

## Question 8

A company has 40 VPCs across several accounts and a data center connected through AWS Direct Connect. Applications in any VPC must be able to communicate with any other VPC and with the data center. The network team is struggling with the complexity of managing individual connections and wants a single, scalable hub. Which solution should be used?

A) Create an AWS Transit Gateway, attach all VPCs and the Direct Connect connection (through a transit virtual interface and Direct Connect gateway), and manage routing from its route tables
B) Create VPC peering connections between every pair of VPCs and route on-premises traffic through one peered VPC
C) Create a virtual private gateway in each VPC and connect each one to the data center with its own Site-to-Site VPN
D) Expose each application as an AWS PrivateLink endpoint service and create interface endpoints in every other VPC

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Transit Gateway acts as a regional hub-and-spoke router. VPCs and on-premises networks attach to it once, and its route tables control which attachments can reach each other. This removes the need for a mesh of point-to-point connections.

**Why option B is wrong:**
VPC peering is not transitive, so traffic cannot flow through a peered VPC to the data center or to a third VPC. A full mesh for 40 VPCs requires 780 peering connections, which is the complexity the team wants to avoid.

**Why option C is wrong:**
A VPN connection per VPC multiplies tunnels to manage, does not provide VPC-to-VPC transit routing, and runs over the internet instead of using the existing Direct Connect link.

**Why option D is wrong:**
PrivateLink exposes specific services one way to consumers. It is not a general-purpose routing solution for bidirectional communication between networks, and it would require an endpoint per service per VPC.

</details>

## Question 9

A company is migrating a data analytics platform to AWS. Its data center needs to transfer about 5 TB of data per day to AWS, requires consistent network latency, and the security policy prohibits sending this traffic over the public internet. The company can wait several weeks for provisioning. Which solution meets these requirements?

A) An AWS Site-to-Site VPN connection with two tunnels
B) AWS Client VPN with certificate-based authentication
C) An AWS Direct Connect dedicated connection to the nearest Direct Connect location
D) VPC peering between the on-premises network and the VPC

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Direct Connect provides a dedicated private network connection from the data center to AWS that bypasses the public internet, with consistent latency and throughput up to 10 Gbps or 100 Gbps per connection. Provisioning can take weeks, which the company can accept.

**Why option A is wrong:**
Site-to-Site VPN runs over the public internet, which the policy forbids, and its performance varies. Each tunnel is also limited to about 1.25 Gbps.

**Why option B is wrong:**
Client VPN is a managed service for individual remote users connecting to AWS resources. It is not meant for connecting a data center and still uses the internet.

**Why option D is wrong:**
VPC peering connects two VPCs. It cannot be established with an on-premises network.

</details>

## Question 10

Application servers in private subnets must read and write objects in Amazon S3 and items in Amazon DynamoDB. Traffic must not traverse the internet, and the company wants to avoid the hourly and data processing charges of a NAT gateway and keep costs as low as possible. Which solution meets these requirements?

A) Create gateway VPC endpoints for S3 and DynamoDB and associate them with the private subnet route tables
B) Create a NAT gateway in each Availability Zone and restrict outbound traffic with security groups
C) Create interface VPC endpoints for S3 and DynamoDB in each Availability Zone
D) Create a VPC peering connection to the AWS-managed VPC that hosts S3 and DynamoDB

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Gateway endpoints for S3 and DynamoDB add prefix-list routes to the chosen route tables, sending traffic to those services over the AWS network. They have no hourly or data processing charge, which makes them the lowest-cost private option.

**Why option B is wrong:**
A NAT gateway sends traffic out through an internet gateway and incurs hourly and per-GB processing charges, which is exactly what the company wants to avoid.

**Why option C is wrong:**
Interface endpoints (AWS PrivateLink) do work privately, and are available for S3, but they cost per endpoint-hour per AZ plus per-GB data processing. They are more expensive than gateway endpoints for same-Region access.

**Why option D is wrong:**
AWS services such as S3 and DynamoDB are not hosted in a VPC you can peer with. Peering is not an available way to reach them.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> To turn this architecture foundation into practice with pipelines, data, and analytics, you can also follow the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
