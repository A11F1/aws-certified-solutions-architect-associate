# Review Questions

## Question 1

A scenario requires the right decision on Route 53 and CloudFront. Which approach fits best?

A) Apply a generic solution without considering latency routing.
B) Use latency routing with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of Route 53 and CloudFront with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take latency routing into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine failover with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of Route 53 and CloudFront with the best balance between requirement, risk, and operations.

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
B) Size based on actual usage and take advantage of weighted when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of Route 53 and CloudFront with the best balance between requirement, risk, and operations.

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
B) The one that ignores CDN signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of Route 53 and CloudFront with the best balance between requirement, risk, and operations.

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
B) Practice questions focused on cache and OAC/OAI with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of Route 53 and CloudFront with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A company runs the same web application behind Application Load Balancers in us-east-1 and ap-southeast-1. Users worldwide must be sent to the Region that gives them the lowest network latency, and if one Region becomes unhealthy, traffic must automatically go to the other. Which Route 53 configuration meets these requirements?

A) Geolocation routing records for each continent, pointing to the nearest Region
B) Latency-based routing records for both ALBs, with Route 53 health checks (or alias target health evaluation) enabled
C) Weighted routing records with a 50/50 split and health checks
D) Failover routing with us-east-1 as primary and ap-southeast-1 as secondary

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Latency-based routing returns the record for the AWS Region that provides the lowest latency for the user. When combined with health checks, Route 53 stops returning an unhealthy endpoint and answers with the next best healthy Region.

**Why option A is wrong:**
Geolocation routing is based on the user's geographic location, not measured latency, so the closest country is not always the fastest Region. It is used for localization or compliance and also needs a default record.

**Why option C is wrong:**
Weighted routing splits traffic by proportion regardless of where the user is, so many users would be sent to the distant Region.

**Why option D is wrong:**
Failover routing is active-passive. All users would be sent to us-east-1 even if they are far away, and the Singapore Region would only be used during an outage.

</details>

## Question 7

A company wants users to reach its application at the root domain example.com, which is hosted in a Route 53 public hosted zone. The application sits behind an Application Load Balancer whose IP addresses change over time. Which DNS record should the solutions architect create?

A) A CNAME record for example.com pointing to the ALB DNS name
B) An A record for example.com containing the current IP addresses of the ALB
C) An NS record for example.com pointing to the ALB DNS name
D) An alias A record for example.com that targets the ALB

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
A Route 53 alias record can be created at the zone apex and maps to AWS resources such as an ALB. Route 53 follows the resource's changing IP addresses automatically, and queries to alias records for AWS resources are not charged.

**Why option A is wrong:**
DNS standards do not allow a CNAME at the zone apex (the root domain), so Route 53 rejects it.

**Why option B is wrong:**
ALB IP addresses change as the load balancer scales, so a hard-coded A record would eventually point to addresses that no longer belong to the load balancer.

**Why option C is wrong:**
NS records delegate a name to name servers and cannot point a domain to a load balancer for web traffic.

</details>

## Question 8

A streaming company stores premium videos, served as many HLS segment files, in a private S3 bucket. Only paying subscribers should be able to watch, and users must not be able to bypass CloudFront by accessing the S3 bucket directly. Which combination meets this with the best user experience?

A) Configure CloudFront origin access control (OAC) with a bucket policy that allows only CloudFront, and use CloudFront signed cookies for subscribers
B) Make the bucket public and issue CloudFront signed URLs for each segment
C) Add a bucket policy that allows the published CloudFront IP address ranges and use signed cookies
D) Keep the bucket private and give users pre-signed S3 URLs for the video segments

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
OAC lets CloudFront sign its requests to S3 so the bucket can stay private and allow access only from the distribution. Signed cookies authorize access to many files (such as all HLS segments) without changing each URL, which gives a smooth experience.

**Why option B is wrong:**
A public bucket allows anyone with an object URL to bypass CloudFront and its signed URL control completely.

**Why option C is wrong:**
IP ranges are shared by all CloudFront customers, so any other distribution could fetch the content, and the lists change frequently. OAC is the supported way to restrict access to your distribution.

**Why option D is wrong:**
Pre-signed S3 URLs bypass CloudFront, so there is no caching or edge delivery, and they need to be generated for every segment.

</details>

## Question 9

A company is launching a real-time multiplayer game that uses UDP traffic to servers behind Network Load Balancers in two Regions. Players around the world need the lowest possible latency, the company needs a small set of static IP addresses for players' firewalls, and traffic must shift quickly to the other Region if one fails. Which service should be used?

A) Amazon CloudFront with the NLBs as custom origins
B) Amazon Route 53 latency-based routing with health checks
C) AWS Global Accelerator with the NLBs as endpoints
D) Amazon S3 Transfer Acceleration

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Global Accelerator provides two static anycast IP addresses and routes TCP and UDP traffic over the AWS global network to the closest healthy endpoint. Health checks detect failures and shift traffic in seconds, without waiting for DNS caches.

**Why option A is wrong:**
CloudFront supports HTTP, HTTPS, and WebSocket traffic, not arbitrary UDP game traffic, and it does not provide the static IPs required.

**Why option B is wrong:**
Route 53 only answers DNS queries, so traffic goes over the public internet, and failover depends on client DNS caching and TTLs. It does not provide static anycast IPs.

**Why option D is wrong:**
S3 Transfer Acceleration is only for transferring objects to and from S3 buckets.

</details>

## Question 10

A company hosts its main website on an ALB in us-east-1. It wants to show a static maintenance page hosted in an S3 website bucket whenever the main site is unhealthy, with automatic switching and no manual DNS changes. Which Route 53 configuration should be used?

A) Failover routing with a primary record for the ALB associated with a health check, and a secondary record for the S3 website endpoint
B) Weighted routing with weight 100 for the ALB and weight 0 for the S3 website, with no health check
C) Simple routing with two records, one for the ALB and one for the S3 website endpoint
D) Geolocation routing with a default record for the ALB and a location-specific record for the S3 website

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Failover routing is built for active-passive designs. Route 53 answers with the primary record while its health check passes, and switches to the secondary record automatically when the health check fails. When the primary recovers, traffic returns to it.

**Why option B is wrong:**
With a weight of 0, Route 53 sends traffic to the S3 record only if all records with non-zero weight are unhealthy, and without a health check Route 53 has no signal to detect the failure.

**Why option C is wrong:**
Simple routing returns all values (or a random one) with no health evaluation, so users would regularly be sent to the maintenance page even while the site is healthy.

**Why option D is wrong:**
Geolocation routing depends on where the user is located, not on the health of the application, so it cannot switch to the maintenance page during an outage.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> When you want to take this topic into data, lakehouse, and hands-on project scenarios, use as your next step: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
