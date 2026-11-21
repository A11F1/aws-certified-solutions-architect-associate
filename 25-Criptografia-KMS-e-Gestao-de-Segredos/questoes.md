# Review Questions

## Question 1

A scenario requires the right decision on encryption and secrets. Which approach fits best?

A) Apply a generic solution without considering KMS.
B) Use KMS with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of encryption and secrets with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take KMS into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine Secrets Manager with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of encryption and secrets with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of rotation when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of encryption and secrets with the best balance between requirement, risk, and operations.

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
B) The one that ignores envelope encryption signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of encryption and secrets with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on envelope encryption and auditoria with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of encryption and secrets with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company runs an Amazon RDS for MySQL database. Security policy requires that the master password be rotated every 30 days, and applications must always retrieve the current password without a redeployment. The team wants the least custom code. Which solution should they use?

A) Store the password as a SecureString in AWS Systems Manager Parameter Store and write a cron job to change it
B) Enable automatic key rotation on the AWS KMS key that encrypts the database
C) Store the credentials in AWS Secrets Manager and enable automatic rotation with the managed rotation function for RDS
D) Store the password in an encrypted Lambda environment variable and update it each month

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Secrets Manager provides built-in rotation for RDS credentials that changes the password in both the secret and the database on a schedule. Applications fetch the current value at runtime through the API.

**Why option A is wrong:**
Parameter Store SecureString protects values but has no native automatic rotation, so you would build and maintain the rotation logic yourself.

**Why option B is wrong:**
KMS key rotation changes the key material used for encryption and has no effect on the database user's password.

**Why option D is wrong:**
Environment variables are static per function version. Updating them monthly is manual and requires a deployment.

</details>

## Question 7

An application in Account B must read objects from an S3 bucket in Account A. The objects are encrypted with SSE-KMS using a customer managed key in Account A. Account B's IAM role already has s3:GetObject permission on the bucket, but requests fail with an access denied error on decryption. What should the solutions architect do?

A) Update the key policy in Account A to allow Account B's role to use the key, and grant kms:Decrypt to the role in an IAM policy in Account B
B) Add kms:Decrypt to the IAM role in Account B only, because IAM permissions are enough for any KMS key
C) Re-encrypt the objects with the AWS managed key aws/s3 so that Account B can use it
D) Make the bucket public so that KMS is bypassed

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
For cross-account use of a customer managed key, the key policy in the owning account must allow the external principal and the external account must also grant the permission through IAM. Both sides must allow the call.

**Why option B is wrong:**
For a key in another account, an IAM policy alone is not enough because the key policy must also grant access to that account.

**Why option C is wrong:**
AWS managed keys cannot be used across accounts, because their key policy cannot be modified.

**Why option D is wrong:**
Public access does not bypass KMS, because objects encrypted with SSE-KMS still require permission to decrypt, and it would expose the data.

</details>

## Question 8

A financial services company needs a hardware security module that only the company controls. It must be single-tenant, the company must manage the HSM users and keys, and AWS must not have access to the key material. Which service meets these requirements?

A) AWS KMS with a customer managed key
B) AWS KMS with key material imported into the key
C) AWS Secrets Manager with a customer managed key
D) AWS CloudHSM

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
CloudHSM provides dedicated, single-tenant HSMs in your VPC. You manage the HSM users and keys, and AWS cannot access the key material.

**Why option A is wrong:**
KMS uses HSMs operated by AWS in a multi-tenant service, so the company does not have exclusive control of the hardware.

**Why option B is wrong:**
Imported key material is still stored and used inside KMS HSMs, which are managed by AWS. This does not give single-tenant hardware control.

**Why option C is wrong:**
Secrets Manager stores secrets and uses KMS for encryption, so it has the same limitations and it is not an HSM.

</details>

## Question 9

An application must encrypt 50 GB backup files before uploading them to S3, using a key that is protected by AWS KMS. The security team does not want plaintext keys stored long term. What is the recommended approach?

A) Call the KMS Encrypt API directly with the whole 50 GB file
B) Call GenerateDataKey, encrypt the file locally with the plaintext data key, then discard it and store the encrypted data key with the file
C) Create a KMS key and store its plaintext key material next to the file
D) Generate a data key and store the plaintext data key in the S3 object metadata

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
This is envelope encryption. KMS generates a data key, the application encrypts the large file locally and stores only the encrypted copy of the data key, which is decrypted through KMS when needed.

**Why option A is wrong:**
The KMS Encrypt API can only encrypt up to 4 KB of data directly, so it is not usable for large files.

**Why option C is wrong:**
KMS keys never expose their plaintext key material, and it cannot be exported.

**Why option D is wrong:**
Storing the plaintext data key next to the encrypted data lets anyone with access to the object decrypt it, which defeats the protection.

</details>

## Question 10

A Lambda function must read a database password stored in AWS Secrets Manager. The secret is encrypted with a customer managed KMS key. The company enforces least privilege. Which TWO configurations should the solutions architect apply?

A) Store the password in plaintext in a Lambda environment variable as a fallback
B) Allow secretsmanager:GetSecretValue on the specific secret in the function's execution role
C) Attach the AdministratorAccess managed policy to the execution role
D) Make the secret readable by any principal in the account
E) Allow the execution role to use kms:Decrypt on the customer managed key through the key policy or an IAM policy

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B, E

**Explanation:**
The function needs permission to retrieve the specific secret, and permission to decrypt it with the customer managed key, both scoped narrowly. These two permissions satisfy least privilege.

**Why option A is wrong:**
Keeping the password in plaintext in an environment variable exposes it in configuration and defeats using Secrets Manager.

**Why option C is wrong:**
AdministratorAccess grants far more than necessary and violates least privilege.

**Why option D is wrong:**
Letting any principal read the secret is overly permissive and does not follow least privilege.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> When you want to take this topic into data, lakehouse, and hands-on project scenarios, use as your next step: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
