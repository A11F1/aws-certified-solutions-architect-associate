# Review Questions

## Question 1

A scenario requires the right decision on governance and cost. Which approach fits best?

A) Apply a generic solution without considering Organizations.
B) Use Organizations with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of governance and cost with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take Organizations into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine SCP with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of governance and cost with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of Cost Explorer when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of governance and cost with the best balance between requirement, risk, and operations.

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
B) The one that ignores rightsizing signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of governance and cost with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on Savings Plans and tags with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of governance and cost with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company runs 60 AWS accounts in AWS Organizations, grouped into OUs. The security team must guarantee that no one in any member account, including account administrators, can stop or delete the AWS CloudTrail trails that the organization uses for auditing. The solution must be enforced centrally with the least ongoing effort. What should the solutions architect do?

A) Deploy an IAM policy that denies cloudtrail:StopLogging and cloudtrail:DeleteTrail to every IAM user and role in each member account
B) Attach a service control policy (SCP) that denies those actions to the management account only
C) Attach a service control policy (SCP) that denies cloudtrail:StopLogging and cloudtrail:DeleteTrail to the root of the organization or the relevant OUs
D) Create an AWS Config rule in each account that detects a stopped trail and sends an SNS notification

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
SCPs set the maximum permissions for member accounts, and an explicit deny in an SCP cannot be overridden by any IAM policy in the account, even for administrators. Attaching the SCP at the root or OU level applies it to all current and future accounts below it.

**Why option A is wrong:**
IAM policies are managed inside each account, so an account administrator can simply detach or edit them. It also requires deploying and maintaining the policy in 60 accounts.

**Why option B is wrong:**
SCPs never restrict the management account, and attaching one there would not affect the member accounts you want to protect.

**Why option D is wrong:**
An AWS Config rule with a notification only detects the problem after logging has already been stopped. It does not prevent the action.

</details>

## Question 7

A company has 25 accounts in one organization with all features enabled. The data platform account purchased a one-year Compute Savings Plan sized for its Amazon EC2 usage, but a project ended and part of the commitment is now unused. Other accounts in the organization are running steady EC2 and Fargate workloads on On-Demand pricing. What is the simplest way to benefit from the unused commitment?

A) Do nothing special, because Savings Plans discounts are shared across accounts in the organization by default and are applied to eligible usage in the other accounts
B) Transfer the Savings Plan to another account using the Billing console
C) Sell the unused commitment on the Reserved Instance Marketplace
D) Share the Savings Plan with the other accounts through AWS Resource Access Manager

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
With consolidated billing, Savings Plans and Reserved Instance discounts are shared across all accounts in the organization by default. A Compute Savings Plan applies to eligible EC2, Fargate and Lambda usage in any account, so the unused commitment is absorbed automatically.

**Why option B is wrong:**
Savings Plans cannot be transferred between accounts. Their benefit is shared through the organization's billing family instead.

**Why option C is wrong:**
The Reserved Instance Marketplace is for selling eligible Standard Reserved Instances, not Savings Plans.

**Why option D is wrong:**
AWS RAM shares resources such as subnets and Transit Gateways, and it is not used to share billing discounts.

</details>

## Question 8

A central networking team owns a VPC with large subnets in us-east-1. Application teams in 20 accounts, all in one OU, need to launch EC2 instances and load balancers directly into those subnets so they share the same network, routing and NAT gateways. The team wants to avoid managing peering connections and duplicate NAT infrastructure. Which solution meets these requirements?

A) Create a VPC peering connection between the central VPC and a VPC in each application account
B) Attach every application VPC to a Transit Gateway and share the NAT gateways through route tables
C) Expose the central VPC as an endpoint service with AWS PrivateLink and create interface endpoints in each account
D) Enable resource sharing with AWS Organizations in AWS RAM and share the subnets with the OU (VPC sharing)

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
VPC sharing through AWS RAM lets the owner account share subnets with other accounts or OUs in the organization. Participant accounts launch their own resources into those subnets, which gives one shared network, NAT and routing configuration.

**Why option A is wrong:**
Peering connects separate VPCs, and each account would still need its own subnets and NAT gateways. Managing 20 peerings is also what the team wants to avoid.

**Why option B is wrong:**
A Transit Gateway interconnects separate VPCs, but the application teams would still run in their own VPCs and subnets. NAT gateways would not be shared as if they were in one subnet.

**Why option C is wrong:**
PrivateLink exposes individual services to consumers. It does not let other accounts place their instances into the owner's subnets.

</details>

## Question 9

A company is starting a multi-account strategy on AWS. It wants a secure landing zone with separate accounts for logging and security, preventive and detective guardrails, centralized CloudTrail and Config logs, and a way for teams to request new, pre-baselined accounts. The company has a small cloud team and wants to minimize custom code. Which approach is best?

A) Create the accounts manually in AWS Organizations and write custom Lambda functions to configure each one
B) Set up AWS Control Tower to build the landing zone and use Account Factory to provision new accounts
C) Use a single AWS account with separate VPCs and IAM groups per team
D) Use AWS CloudFormation StackSets only, with the management account used for all workloads

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
AWS Control Tower orchestrates Organizations, IAM Identity Center, CloudTrail and Config to create a governed multi-account environment with managed guardrails (SCPs and Config rules). Account Factory provisions new accounts with a consistent baseline with little custom code.

**Why option A is wrong:**
This can work but requires building and maintaining the baseline, logging and guardrail automation yourself, which contradicts the requirement to minimize custom code.

**Why option C is wrong:**
A single account does not provide the isolation boundary, separate logging and security accounts, or SCP guardrails that the company asked for.

**Why option D is wrong:**
StackSets can deploy baselines, but they do not provide guardrails or an account vending process on their own. Running workloads in the management account is also a poor practice.

</details>

## Question 10

A finance team needs a monthly cost report for 40 accounts in one organization, broken down by the Project tag that engineers already apply to resources. Reports must be produced from a single place without logging in to each account. Which TWO actions should the solutions architect take?

A) Use AWS CloudTrail event history to calculate the spend per project
B) Activate the Project tag as a user-defined cost allocation tag in the management account
C) Create a separate Cost and Usage Report in every member account and merge them manually each month
D) Use AWS Trusted Advisor to allocate spend to projects
E) Use AWS Cost Explorer or the Cost and Usage Report in the management account and group or filter by the Project tag

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, E

**Explanation:**
Tags must be activated as cost allocation tags in the management account before they appear in billing data. Cost Explorer or the CUR from the management account can then group the consolidated costs of all member accounts by that tag.

**Why option A is wrong:**
CloudTrail records API activity and not cost data, so it cannot calculate spend.

**Why option C is wrong:**
The management account already receives consolidated data for the whole organization, so per-account reports and manual merging add effort for no benefit.

**Why option D is wrong:**
Trusted Advisor provides optimization checks and recommendations, not tag-based cost allocation reports.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
