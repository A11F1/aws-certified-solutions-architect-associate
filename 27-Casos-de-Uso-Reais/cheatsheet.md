# Quick Guide

## Decision Table

| Question signal | When to use | When to avoid | Exam trap |
|---|---|---|---|
| Low manual operation | Managed service with native automation | Self-managed solution without a need | Confusing full control with the best total cost |
| Unpredictable spike | Horizontal scaling and decoupling | Fixed capacity and manual adjustment | Sizing for the average and failing at the peak |
| Strong security requirement | Least privilege + encryption + auditing | Broad permission for convenience | Thinking encryption alone solves governance |
| Cost as an explicit constraint | Choice based on consumption and access profile | A single class for all data | Cutting cost without validating functional impact |

## Service Signals

- **S3**: prioritize when the scenario needs native integration and a lower operational load.
- **RDS**: use when you need controlled elasticity and predictable behavior at peak.

## When to Use This Module in Review

- Before architecture practice exams focused on trade-offs.
- When you keep making the same mistake choosing between two similar alternatives.
- In the final stretch to calibrate your criteria for eliminating options.

## Recurring Pitfalls

- Solving availability with a performance feature.
- Trading simplicity for an oversized architecture.
- Ignoring observability when defining the final design.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> When you want to take this topic into data, lakehouse, and hands-on project scenarios, use as your next step: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
