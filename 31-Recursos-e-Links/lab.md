# Guided Lab: Curate Your Official AWS Resources into a Personal Study Plan

## Objective

Use AWS's own documentation, whitepapers, Skill Builder, and re:Post to assemble a prioritized, dated study plan. Along the way you will practice the exact reading skills that pay off on the exam: finding a service's FAQ, its quotas, and its decision guidance.

## Scenario

You have dozens of bookmarks and no plan. You will pick the five services that appear most in your weak areas, read the right page of each source, and produce a table with what to read, why, and when.

## Services Used

- AWS Documentation (docs.aws.amazon.com)
- AWS Whitepapers and Guides: Well-Architected Framework, Disaster Recovery whitepaper
- AWS Skill Builder
- AWS re:Post
- No AWS resources are created.

## Prerequisites

- A browser, a free AWS account (only needed to sign in to Skill Builder and re:Post; both can use an AWS Builder ID).
- Your weak areas, ideally from the error notebook in the practice-exam module.

## Expected Cost

$0. Many Skill Builder courses are free; some features such as labs or official practice exams require a paid subscription, so check the current plan details before paying. Avoid paying for unofficial "dump" content.

## Step by Step

### Step 1: Choose five target services or topics

List the five services or topics that cost you the most points, for example S3 storage classes, VPC connectivity, RDS Multi-AZ versus read replicas, IAM policy evaluation, and DR strategies.

### Step 2: Search the AWS documentation without the CLI

1. Go to docs.aws.amazon.com and use the search box, or search the web with `site:docs.aws.amazon.com <topic>`.
2. Prefer the **User Guide** for concepts, the **API Reference** only for parameters, and the **Developer Guide** for application patterns.
3. Use the left-hand table of contents of the User Guide and look for "How it works", "Best practices", "Security", and "Quotas" pages.
4. On any doc page, use the "Document history" link near the bottom to see what changed recently, which helps you avoid outdated blog posts.

### Step 3: Read a service FAQ

1. Open aws.amazon.com/<service>/faqs (for example `aws.amazon.com/s3/faqs`).
2. Skim every question heading; read fully the ones that start with "When should I use...", "What is the difference between...", and "How is it billed?".
3. Write down three "X versus Y" distinctions (for example S3 Standard-IA versus One Zone-IA, or Multi-AZ versus read replica).

### Step 4: Read a service's quotas page

1. In the service's User Guide, find "Quotas" (also titled "Service quotas" or "Limits"). Alternatively, search "<service> quotas" in the AWS General Reference, which lists endpoints and quotas.
2. Note which values are adjustable and which are hard limits, and whether they are per Region or per account.
3. Pick two quotas and write the design consequence (for example "Lambda concurrency is limited per Region, so use reserved concurrency to protect critical functions").

### Step 5: Read the whitepapers with a purpose

- **AWS Well-Architected Framework:** read one pillar overview per day (Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability) and note its design principles. Map them to the exam domains: Security to Secure, Reliability to Resilient, Performance Efficiency to High-performing, Cost Optimization to Cost-optimized.
- **Disaster Recovery of Workloads on AWS (Disaster Recovery whitepaper):** learn the four strategies in order of increasing cost and decreasing RTO/RPO: backup and restore, pilot light, warm standby, and multi-site active/active. Write one AWS service example for each.
- Find them at aws.amazon.com/whitepapers or in the Well-Architected documentation. Verify you are reading the latest version by checking the published or updated date.

### Step 6: Use Skill Builder and re:Post

1. Sign in at skillbuilder.aws, search for "AWS Certified Solutions Architect - Associate", and open the Exam Prep material for SAA-C03. Add the free items to your learning plan and note which items need a subscription.
2. Go to repost.aws, search a confusing topic, and prefer answers that are AWS-verified or accepted and that cite documentation. Treat unverified answers as hints, not facts.
3. Check the date on every answer; AWS services change often.

### Step 7: Fill the study-plan table

| Priority | Topic / service | Source (doc, FAQ, whitepaper, Skill Builder, re:Post) | Exact page or title | Question it answers | Time (min) | Planned date | Done |
|----------|-----------------|--------------------------------------------------------|---------------------|---------------------|-----------|--------------|------|
| 1 | | | | | | | [ ] |
| 2 | | | | | | | [ ] |
| 3 | | | | | | | [ ] |
| 4 | | | | | | | [ ] |
| 5 | | | | | | | [ ] |

Rules: no more than 60 minutes of reading for any single row, and every row must name a question you want answered. Schedule at least one practice or hands-on lab after each reading block.

## Validation

- The table has five rows with exact page titles, not just "S3 docs".
- You can name the four DR strategies and rank them by cost and recovery speed.
- For each of your five topics you wrote at least one FAQ distinction and one quota or limit.
- You have a dated plan covering the next two weeks, with review days.

## Cleanup

Nothing to remove. Prune your bookmarks to the pages listed in the table, and discard everything else that is not an official or well-vetted source.

## Exam Takeaway

The exam is built from AWS documentation and Well-Architected thinking, so go to the source. Know the DR strategy spectrum (backup and restore, pilot light, warm standby, multi-site), where each trades cost against RTO and RPO, and know each Well-Architected pillar's design principles. When a service FAQ says "use X for Y", that sentence is often the deciding rule in a scenario question.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
