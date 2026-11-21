# Quick Guide

## Decision Table

| Question signal | Best choice | Avoid | Common trap |
|---|---|---|---|
| “Minimum necessary access” | Granular IAM policy + role | User with broad permission | Confusing agility with full permission |
| “Unpredictable spike” | ASG + ALB + decoupling | Single vertically scaled instance | Sizing for the average and collapsing at the peak |
| “No public internet” | VPC Endpoint/PrivateLink | NAT for internal AWS traffic | Paying more and widening the risk surface |
| “Low RPO / short RTO” | Multi-AZ or warm standby depending on the target | Occasional backup without testing | Choosing DR that is too cheap for an aggressive target |
| “Cost with predictable usage” | Savings Plans/RI + rightsizing | On-Demand without analysis | Ignoring commitment for a steady workload |

## Quick elimination checklist

1. Does the option violate an explicit security requirement?
2. Does the option add unnecessary components?
3. Does the option increase manual operation without technical gain?
4. Is there a managed service that solves it with less risk?

## Triplets that cause the most confusion on the SAA-C03

- **SG vs NACL**: SG is stateful at the ENI; NACL is stateless at the subnet.
- **Multi-AZ vs Read Replica**: Multi-AZ = availability; replica = read scaling.
- **SQS vs SNS vs EventBridge**: durable queue, pub/sub fan-out, event-based routing.

## Use in the final stretch

- Review before each mini practice exam.
- Mark which row of the table you got the decision wrong on.
- Update the error notebook with the correct rule.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> When you want to take this topic into data, lakehouse, and hands-on project scenarios, use as your next step: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
