# Hands-on Labs

## Overview

This module covers **Consolidating architectural patterns through light practice**, focusing on the kind of decision that appears on the SAA-C03. The core idea is to turn business requirements into objective technical choices, balancing resilience, performance, security, and cost. On the exam, what makes the difference is not memorizing isolated services: it is identifying which constraint in the scenario weighs the most and choosing the pattern that reduces operational risk.

## Key Concepts

- Role of the services: VPC, EC2, S3, RDS, CloudWatch.
- Prioritization by dominant requirement (latency, availability, compliance, or cost).
- Integration between services with decoupling and minimal observability.
- Choosing the architecture with the lowest viable complexity for the context.

## Relevance for the Exam

On the SAA-C03, this domain usually appears in questions with two technically possible alternatives. The tiebreaker normally comes from details such as: consistency model, failover strategy, recurring operating cost, throughput impact, and the level of automation required. That is why studying this module means training your judgment in choosing, not just learning terminology.

## Practical Signals (3 to 5)

1. When the question asks for an elastic response with little manual operation, favor managed services.
2. When components depend on each other, check whether there is decoupling to absorb peaks and partial failure.
3. When there is a security constraint, confirm encryption in transit/at rest and least privilege.
4. If cost is part of the requirement, compare consumption classes, capacity mode, and access pattern.
5. If the scenario is global, check traffic routing, latency, and distribution strategy.

## Common Pitfalls

- Choosing a premium feature without a real need in the scenario.
- Confusing a high availability feature with a read scaling feature.
- Ignoring service limits and behavior during failure.
- Solving a security requirement with a tool that is not suited to the secret's lifecycle.

## Next Review Step

1. Review cheatsheet.md to consolidate decision signals.
2. Solve questoes.md without looking anything up to test your technical judgment.
3. Use flashcards.md in short daily reviews to lock in critical differences.

## Complementary Studies

To reinforce AWS fundamentals before diving into architectural scenarios:
https://github.com/Thiago-code-lab/aws-certified-cloud-practitioner-brasil

To connect architecture with generative AI and Bedrock in a complementary track:
https://github.com/Thiago-code-lab/aws-certified-ai-practitioner-brasil
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> If your goal is to connect AWS architecture with ingestion, processing, and analytics, the complementary track is: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
