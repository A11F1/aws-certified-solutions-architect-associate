# Guided Lab

## Objective

Build a minimal working flow that demonstrates **Subnets, routes, and private egress lab**, with clear behavior validation and controlled cost for a study environment.

## Services Used

VPC, NAT Gateway, VPC Endpoint.

## Expected Cost Range

- Short study environment: low cost if run for only a few hours.
- Avoid keeping resources running after validation.
- Prefer the free tier when available.

## Step by Step

1. Create the essential resources of the flow in a lab account.
2. Configure minimal access policies and basic observability.
3. Run a small test load to validate the functional response.
4. Simulate a simple failure (component interruption) and observe recovery.
5. Record a key metric and adjust one setting to compare the result.

## Validation

- The end-to-end flow runs without critical errors.
- Logs/metrics show the expected behavior.
- There is evidence of a decision between performance and cost.

## Cleanup

1. Remove the resources created in the lab.
2. Delete temporary data and unnecessary snapshots.
3. Check billing the next day to confirm everything was shut down.

## Exam Takeaway

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
