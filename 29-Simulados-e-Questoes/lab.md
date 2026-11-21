# Guided Lab: Build a Timed 65-Question Mock-Exam Routine and Error Notebook

## Objective

Set up a repeatable practice-exam routine that mirrors the real SAA-C03 format, track results by exam domain, and turn every wrong answer into a specific study action. This is not an AWS build lab, so no AWS resources are created.

## Scenario

You have been reading and doing labs, but you do not know how you perform under time pressure or which domains are weakest. You will run one full timed mock, log it, and use the results to plan the next week.

## Services Used

- None (no AWS resources). Tools: a timer, a spreadsheet or the markdown tables below, and free or low-cost practice resources such as AWS Skill Builder.

## Prerequisites

- Finished at least one pass through the course modules.
- A quiet 2.5-hour block and a second 1-hour block for review.
- Access to a bank of at least 65 questions you have not seen (for example this repository's quizzes, AWS Skill Builder practice sets, or a reputable third-party exam).

## Expected Cost

$0 with free resources. Optional: AWS Skill Builder's paid subscription and third-party practice exams are priced separately; check current prices on their sites. Avoid: memorizing questions from "dump" sites. They violate the AWS certification agreement, are often wrong, and teach recall instead of reasoning.

## Step by Step

### Step 1: Verify the current exam facts

Open the official exam guide for AWS Certified Solutions Architect - Associate (SAA-C03) from the certification page at aws.amazon.com/certification/certified-solutions-architect-associate and confirm the numbers below, since AWS can revise them:

| Item | Value (as commonly published for SAA-C03) |
|------|-------------------------------------------|
| Total questions | 65 (50 scored, 15 unscored pretest questions) |
| Time | 130 minutes |
| Question types | Multiple choice and multiple response |
| Passing scaled score | 720 out of 1000 |

Domain weights in the SAA-C03 exam guide:

| Domain | Weight |
|--------|--------|
| Domain 1: Design Secure Architectures | 30% |
| Domain 2: Design Resilient Architectures | 26% |
| Domain 3: Design High-Performing Architectures | 24% |
| Domain 4: Design Cost-Optimized Architectures | 20% |

Note that the biggest domain is Secure (30%), not Resilient (26%), even though Resilient is listed second in some summaries. If the guide you download shows a newer version number or different weights, use the guide, not this table.

### Step 2: Convert the weights into a 65-question mock

Multiply 65 by each weight and round so the total stays 65:

| Domain | Weight | Questions in a 65-question mock |
|--------|--------|--------------------------------|
| Secure | 30% | 20 |
| Resilient | 26% | 17 |
| High-performing | 24% | 15 |
| Cost-optimized | 20% | 13 |
| Total | 100% | 65 |

(Exact values are 19.5, 16.9, 15.6, 13.0; the rounding above sums to 65.) Pull questions from your bank to match these counts and tag each one with its domain.

### Step 3: Run the timed session

1. Set a countdown for 130 minutes. No notes, no AWS console, no search.
2. Budget about 2 minutes per question. On a first pass, answer or flag anything that takes longer than 2 minutes and move on.
3. Reserve the last 15 minutes for flagged questions.
4. For every answer, jot the confidence level (H, M, L) on scratch paper. You will use it in review.
5. Do not pause the timer.

### Step 4: Score and log the attempt

Use this table for each mock (or the same columns in a spreadsheet):

| Attempt | Date | Score % | Secure (/20) | Resilient (/17) | High-perf (/15) | Cost (/13) | Time left (min) | Main weakness |
|---------|------|---------|--------------|-----------------|-----------------|-----------|-----------------|---------------|
| 1 | | | | | | | | |
| 2 | | | | | | | | |
| 3 | | | | | | | | |

Passing at scaled 720 is not the same as 72% raw, and unscored questions are mixed in, so treat roughly 75-80% on good practice exams as a safe target rather than a precise predictor.

### Step 5: Fill the error notebook

Review every wrong answer and every right answer you marked L (low confidence). One row per question:

| # | Domain | Service / topic | What I chose | Correct answer | Why I was wrong (knowledge, misread, or trap) | Keyword I missed | Fix (doc/lab/flashcard) | Re-test date |
|---|--------|-----------------|--------------|----------------|----------------------------------------------|------------------|------------------------|--------------|
| 1 | | | | | | | | |

Classify each error into one of three buckets: knowledge gap (study it), misread (slow down on keywords like "least operational overhead", "most cost-effective", "highly available"), or trap (two plausible options; write the deciding rule). Look for the bucket that dominates.

### Step 6: Build the weekly plan

- Spend 60 percent of study time on the weakest domain and the services that appear most in your error notebook.
- Do one hands-on lab or doc read for each repeated weakness.
- Re-test those notebook rows after 3 and 7 days.
- Take a full mock about every 5 to 7 days, and stop taking full mocks if you are scoring consistently above target for three attempts in a row.

### Step 7: Complete the resource checklist

- [ ] Official exam guide and sample questions downloaded (certification page above)
- [ ] AWS Skill Builder account created (skillbuilder.aws) and the free SAA-C03 exam prep content browsed
- [ ] Official practice question set for SAA-C03 attempted (availability and cost can change; check Skill Builder)
- [ ] AWS Well-Architected Framework pillar summaries read
- [ ] AWS Whitepapers on resilience and disaster recovery skimmed
- [ ] Service FAQ pages read for S3, EC2, VPC, RDS, DynamoDB, IAM
- [ ] This repository's quiz and flashcard modules completed
- [ ] Exam scheduled via Pearson VUE or online proctoring; ID and test-environment rules read

## Validation

- You have a completed attempt row with per-domain scores.
- The error notebook has an entry for every wrong answer, each with a concrete fix, not "review S3".
- You can name your weakest domain and your top three weak services from data, not feeling.
- The next mock date is on your calendar.

## Cleanup

Nothing to delete in AWS. Archive the error notebook and keep it, since it is your best revision material in the last week. Remove any dump-site bookmarks.

## Exam Takeaway

The exam rewards choosing the best option under constraints, and the keywords decide it: "most cost-effective", "least operational overhead", "highly available", "secure by default". Practice timing as well as content, because 130 minutes for 65 questions is tight if you overthink. Weight your study by the published domain percentages (Secure 30, Resilient 26, High-performing 24, Cost-optimized 20), then adjust by your own error data.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
