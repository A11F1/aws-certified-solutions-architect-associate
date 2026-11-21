# Review Questions

## Question 1

A scenario requires the right decision on IAM and security. Which approach fits best?

A) Apply a generic solution without considering least privilege.
B) Use least privilege with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of IAM and security with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take least privilege into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine policies with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of IAM and security with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of roles when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of IAM and security with the best balance between requirement, risk, and operations.

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
B) The one that ignores KMS signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of IAM and security with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on CloudTrail and SCP with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of IAM and security with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company runs a Java application on EC2 instances that must read and write objects in a single S3 bucket. A junior engineer proposes creating an IAM user and storing its access keys in a configuration file on each instance. The security team wants to eliminate long-lived credentials and follow least privilege. What should the architect recommend?

A) Store the IAM user's access keys in environment variables that are loaded at boot
B) Create an IAM role with a policy scoped to that bucket and attach it to the instances through an instance profile
C) Store the access keys in AWS Secrets Manager with automatic rotation and have the application fetch them at startup
D) Use the account root user's access keys because they have unrestricted permissions to S3

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
An IAM role delivered through an instance profile gives the application temporary credentials that the SDK retrieves and refreshes automatically from the instance metadata service. Nothing long-lived is stored, and the attached policy can be limited to the one bucket.

**Why option A is wrong:**
Environment variables are still long-lived access keys sitting on the instance, where they can leak through logs, process listings, or AMI snapshots.

**Why option C is wrong:**
Rotation reduces risk but you still manage an IAM user and the application needs credentials to reach Secrets Manager in the first place. A role removes the problem entirely.

**Why option D is wrong:**
Root access keys violate least privilege and are a serious security anti-pattern. AWS recommends not creating them at all.

</details>

## Question 7

A development account runs a Lambda function that must read objects from an S3 bucket owned by a separate production account. The security team requires that no IAM user keys are shared between accounts and that access can be revoked centrally by the production account. Which solution meets these requirements?

A) Create an IAM user in the production account and place its access keys in the Lambda environment variables
B) Make the bucket public and require a custom Referer header to restrict access
C) Copy the bucket contents into the development account every night using a cron job on an EC2 instance
D) Create an IAM role in the production account with S3 read permissions and a trust policy for the development account, and let the Lambda execution role call sts:AssumeRole

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
Cross-account access with a role gives temporary credentials through AWS STS. The production account owns the role and its permissions, so it can restrict or revoke access by editing the trust policy or the role at any time. The development account also needs to allow its Lambda execution role to call sts:AssumeRole.

**Why option A is wrong:**
This shares long-lived keys between accounts and stores them in plain-text configuration, which is exactly what the requirements forbid.

**Why option B is wrong:**
A Referer header is trivially spoofed and a public bucket exposes production data to anyone on the internet.

**Why option C is wrong:**
Copying data duplicates sensitive production data, adds operational overhead, and still requires some cross-account permission to do the copy. Access is also not revocable in real time.

</details>

## Question 8

A company uses AWS Organizations with 40 accounts. The security team needs to guarantee that no one, including account administrators, can stop or delete the organization's CloudTrail trails. Which approach enforces this?

A) Attach a Service Control Policy to the relevant OUs that denies cloudtrail:StopLogging and cloudtrail:DeleteTrail
B) Create an IAM policy in each account that denies those actions and attach it to the administrator groups
C) Set a permissions boundary on all IAM roles and trust administrators to keep the boundary in place
D) Create an AWS Config rule that detects when logging is turned off and sends an SNS notification

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
SCPs define the maximum permissions available in member accounts and apply even to account administrators, though not to the management account itself. An explicit deny in an SCP cannot be overridden by any IAM policy in the member account.

**Why option B is wrong:**
An account administrator can edit or detach an IAM policy in their own account, so this control can be removed by the very people it is meant to restrict.

**Why option C is wrong:**
Permissions boundaries are set per principal in each account and administrators who can manage IAM can alter or remove them. They are not an organization-wide preventive control.

**Why option D is wrong:**
A Config rule is detective: it reports after logging has already been stopped. The requirement is to prevent the action.

</details>

## Question 9

A company stores sensitive customer records in S3. Compliance requires encryption with a key the company controls, an audit trail of every key use, the ability to disable the key immediately, and automatic yearly key rotation with minimal operational effort. Which option meets these requirements?

A) Server-side encryption with Amazon S3 managed keys (SSE-S3)
B) Server-side encryption with the AWS managed key aws/s3
C) Server-side encryption with a customer managed KMS key with automatic rotation enabled
D) Client-side encryption using keys generated and stored on an on-premises server

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
A customer managed KMS key gives you control of the key policy, lets you disable the key, logs every use in CloudTrail, and supports automatic rotation that you can enable with a single setting.

**Why option A is wrong:**
With SSE-S3 the keys are fully managed by S3. You cannot control the key lifecycle, disable it, or get per-use key audit entries.

**Why option B is wrong:**
AWS managed keys are rotated automatically and their use is logged, but you cannot edit their key policy or disable them, so the control requirement is not met.

**Why option D is wrong:**
It could satisfy the control requirement but demands building and operating your own key management and rotation, which contradicts the minimal operational effort constraint.

</details>

## Question 10

During an account review, a security engineer finds that the AWS account root user has no MFA and that the team uses it for daily administration. Which TWO actions should the engineer take to follow security best practices?

A) Enable multi-factor authentication on the root user
B) Create access keys for the root user so automation scripts can authenticate
C) Continue using the root user but share its password in a team vault
D) Delete the root user to eliminate the risk
E) Create IAM Identity Center or IAM users with scoped permissions for daily work and reserve the root user for tasks that only it can perform

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A, E

**Explanation:**
The root user should be protected with MFA and used only for the small set of tasks that require it, such as changing the account's support plan or closing the account. Everyday work should use identities with least-privilege permissions.

**Why option B is wrong:**
Root access keys are unrestricted and long-lived. Best practice is to not create them, and to delete any that exist.

**Why option C is wrong:**
Sharing the password keeps everyone operating with unrestricted privileges and makes actions impossible to attribute to an individual.

**Why option D is wrong:**
The root user cannot be deleted. It is created with the account and is required for certain tasks.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
