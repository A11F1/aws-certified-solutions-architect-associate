# Review Questions

## Question 1

A scenario requires the right decision on hybrid connectivity. Which approach fits best?

A) Apply a generic solution without considering Direct Connect.
B) Use Direct Connect with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hybrid connectivity with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take Direct Connect into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine VPN with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hybrid connectivity with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of Transit Gateway when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hybrid connectivity with the best balance between requirement, risk, and operations.

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
B) The one that ignores BGP signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hybrid connectivity with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on BGP and PrivateLink with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of hybrid connectivity with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company connects its data center to AWS with a single 1 Gbps AWS Direct Connect connection. Management is concerned about a Direct Connect outage. They need a backup path that is encrypted, can be provisioned within hours, and costs far less than a second dedicated circuit. What should the solutions architect recommend?

A) Order a second Direct Connect connection at a different location
B) Create an AWS Site-to-Site VPN connection over the internet to the same virtual private gateway or Transit Gateway as a failover path
C) Create a VPC peering connection between the data center network and the VPC
D) Add an internet gateway and have the on-premises servers reach private IPs over the internet

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
A Site-to-Site VPN can be set up in minutes to hours, is IPsec-encrypted and is inexpensive. With BGP, the VPN works as a backup that is used when the Direct Connect path fails.

**Why option A is wrong:**
A second Direct Connect circuit improves resilience, but it takes weeks to provision and is expensive, which violates the hours and low-cost requirements.

**Why option C is wrong:**
VPC peering connects two VPCs and cannot connect an on-premises network.

**Why option D is wrong:**
An internet gateway does not give private connectivity to on-premises networks, and private IPs are not routable over the internet.

</details>

## Question 7

A company has 50 VPCs across 12 accounts in one Region and also an on-premises network connected via Direct Connect. Teams need any-to-any routing between VPCs and the on-premises network, and the network team wants to avoid managing hundreds of individual connections. What is the best solution?

A) Create a full mesh of VPC peering connections and rely on transitive routing through the peers
B) Create a virtual private gateway in each VPC and connect them to each other
C) Publish each VPC as an AWS PrivateLink endpoint service
D) Create an AWS Transit Gateway, attach the VPCs and the Direct Connect gateway, and manage routes in its route tables

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
A Transit Gateway is a regional hub that connects many VPCs and on-premises networks with a hub-and-spoke model. Attachments are managed in one place and route tables control which networks can talk to each other.

**Why option A is wrong:**
VPC peering is non-transitive, and a full mesh of 50 VPCs needs 1,225 peerings, which is exactly the complexity to avoid.

**Why option B is wrong:**
A virtual private gateway connects one VPC to an on-premises network and cannot interconnect VPCs.

**Why option C is wrong:**
PrivateLink gives one-way access to specific services, not general any-to-any routing between networks.

</details>

## Question 8

A SaaS provider hosts an application behind a Network Load Balancer. Hundreds of customers, many using overlapping 10.0.0.0/16 CIDR ranges in their own VPCs, must access it privately without traffic traversing the internet. The provider does not want customers to reach anything else in its VPC. Which solution should the provider choose?

A) Expose the NLB as a VPC endpoint service with AWS PrivateLink and have customers create interface endpoints
B) Create VPC peering connections with each customer VPC
C) Attach the provider and customer VPCs to a shared Transit Gateway
D) Put the application behind a public Application Load Balancer and ask customers to allowlist the provider's IPs

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
PrivateLink exposes a single service through interface endpoints in the consumer VPC. It works with overlapping CIDRs, traffic stays on the AWS network and the consumer can only reach that service.

**Why option B is wrong:**
Peering requires non-overlapping CIDR blocks and opens network-level routing between the whole VPCs.

**Why option C is wrong:**
Transit Gateway also cannot route between overlapping CIDRs and gives broad network connectivity instead of access to a single service.

**Why option D is wrong:**
A public ALB sends traffic over the internet, which violates the private-access requirement.

</details>

## Question 9

On-premises applications connect to a VPC through an AWS Direct Connect private virtual interface. The applications must upload data to Amazon S3 using private IP addresses, without sending traffic over the public internet. Which solution meets this requirement?

A) Create an S3 gateway endpoint in the VPC and route on-premises traffic to it
B) Deploy a NAT gateway in the VPC and send S3 traffic from on-premises through it
C) Create an S3 interface endpoint (AWS PrivateLink) in the VPC and have on-premises systems connect to its private IP addresses or endpoint-specific DNS names
D) Attach an internet gateway to the VPC and restrict the bucket policy to the VPC's public IP

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
S3 interface endpoints have private IP addresses (ENIs) in the VPC, so they are reachable from on-premises over Direct Connect or VPN. Applications can use the endpoint-specific DNS names to reach S3 privately.

**Why option A is wrong:**
Gateway endpoints work through route table entries and cannot be used from on-premises networks, even over Direct Connect or VPN.

**Why option B is wrong:**
A NAT gateway allows private instances in a VPC to initiate outbound connections to the internet, so S3 traffic would still use public endpoints, and it does not accept traffic from on-premises.

**Why option D is wrong:**
An internet gateway and public IP restrictions send traffic over the internet, which breaks the private connectivity requirement.

</details>

## Question 10

A company has a hybrid environment. EC2 instances in a VPC must resolve names in corp.example.com hosted on on-premises DNS servers, and on-premises servers must resolve records in a Route 53 private hosted zone associated with the VPC. A Direct Connect link already exists between the networks. Which TWO actions are required?

A) Create a Route 53 Resolver outbound endpoint and a forwarding rule for corp.example.com that targets the on-premises DNS servers
B) Manually copy every private hosted zone record into the on-premises DNS servers
C) Enable the DNS hostnames option on the VPC so on-premises servers resolve its private zone
D) Create a Route 53 public hosted zone for the same domain and delegate it to the on-premises servers
E) Create a Route 53 Resolver inbound endpoint and configure the on-premises DNS servers to forward queries for the private zone to it

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A, E

**Explanation:**
An outbound endpoint with a forwarding rule sends queries from the VPC to the on-premises DNS. An inbound endpoint gives on-premises resolvers IP addresses in the VPC to forward queries to, so they can resolve private hosted zone names.

**Why option B is wrong:**
Copying records manually creates a maintenance burden and quickly becomes stale, and Resolver endpoints exist to avoid this.

**Why option C is wrong:**
The DNS hostnames setting controls whether instances get DNS names. It does not make a VPC resolver reachable from on-premises.

**Why option D is wrong:**
A public hosted zone exposes records to the internet and cannot hold the private resolution path that was requested.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> To consolidate architecture with real data projects on AWS, keep the complementary track on your radar: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
