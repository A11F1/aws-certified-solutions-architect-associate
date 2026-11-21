# Week 9 Main Lab: Architecture Review Capstone

## Objective

Act as the reviewer. Take a classic three-tier architecture (ALB, Auto Scaling group, RDS) and audit it against the six Well-Architected pillars, with evidence for every answer. Identify the top risks, propose three fixes (and optionally implement them), outline an AWS Pricing Calculator estimate, define RTO/RPO targets with a matching DR strategy, then take a timed 20-question mock exam block and log every mistake in the error notebook. The review document and the exam practice are the main deliverables; deploying the architecture is optional. Finish with a readiness checklist.

Target time: 90 to 120 minutes (45 to 60 minutes for the review, 15 for the estimate, 30 for the exam block and error log). Region for the optional deployment: `us-east-1`.

## Modules Covered

- [26 - Well-Architected Framework](../26-Well-Architected-Framework/README.md): the six pillars and review questions.
- [27 - Real-World Use Cases](../27-Casos-de-Uso-Reais/README.md): matching requirements to reference architectures.
- [28 - Hands-on Labs](../28-Labs-Praticos/README.md): the three-tier lab you review here.
- [29 - Practice Exams](../29-Simulados-e-Questoes/README.md): mock block and error notebook.
- [30 - Glossary](../30-Glossario/README.md): precise vocabulary for the review.
- [31 - Resources and Links](../31-Recursos-e-Links/README.md): official docs and pricing pages to cite as evidence.

## Scenario and Architecture

A retail startup runs an order portal. The business says: "We lose about $2,000 for every hour the site is down. We can tolerate losing up to 15 minutes of orders. Traffic doubles on weekends and tripled during the last promotion. Customer data is regulated. Keep the bill below $400 per month." A contractor delivered the following design, which matches the lab in `28-Labs-Praticos/lab.md` (ALB in public subnets, ASG of two `t3.micro` web instances, RDS MySQL in private subnets). Your job is to decide whether it meets those needs.

```
                      Route 53 (optional)
                            |
                     +------v------+
   Internet -------> |     ALB     |  public subnets, 2 AZs, HTTP :80 only
                     +------+------+
                            |
              +-------------+-------------+
              |                           |
      +-------v-------+           +-------v-------+
      | EC2 t3.micro  |           | EC2 t3.micro  |      Auto Scaling group lab-web-asg
      | AZ us-east-1a |           | AZ us-east-1b |      (min 2, max 2, EC2 health check)
      +-------+-------+           +-------+-------+
              |      SG references SG     |
              +-------------+-------------+
                            |
                    +-------v--------+
                    | RDS MySQL      |  private subnets, single-AZ, db.t3.micro,
                    | lab-mysql      |  1 day backup retention, not encrypted
                    +----------------+

 Not present: HTTPS, WAF, scaling policy, Multi-AZ DB, cross-Region copy, tagging, alarms, budget.
```

Deliverables of this lab (keep them in a scratch file in your own notes, not in the repo):

1. Well-Architected review table with evidence.
2. Risk register ranked by impact and likelihood.
3. Three fixes, with the CLI commands if you deploy.
4. Pricing Calculator estimate outline.
5. RTO/RPO targets and the chosen DR strategy.
6. Mock exam score and updated error notebook.
7. Final readiness checklist.

## Services Used

- Review: AWS Well-Architected Framework, AWS Well-Architected Tool (free, optional), AWS Pricing Calculator (free).
- Optional deployment of the reviewed stack: VPC, Application Load Balancer, EC2 Auto Scaling, Amazon RDS for MySQL, AWS Certificate Manager, Amazon CloudWatch, AWS Budgets.
- Practice: `29-Simulados-e-Questoes/mini-simulados-por-dominio.md` and `29-Simulados-e-Questoes/caderno-de-erros.md`.

## Prerequisites

- All earlier weekly labs completed or skimmed; your notes and cheat sheets handy during the review, but closed during the timed exam block.
- For the optional deployment: personal AWS account, admin-level identity, AWS CLI v2, and the stack from `28-Labs-Praticos/lab.md` running (resource names below assume `lab-alb`, `lab-web-asg`, `lab-mysql`; replace with yours). If you skip deployment, nothing in AWS is needed.
- A timer, a pen, and a rule: no looking at answers until the block is finished.

## Expected Cost

- Paper-only path: $0. The Pricing Calculator and Well-Architected Tool are free.
- Optional deployment, approximate us-east-1 on-demand prices that you must confirm on the AWS pricing pages: ALB about $0.0225 per hour plus LCU, RDS `db.t3.micro` about $0.017 per hour plus storage, two `t3.micro` about $0.0104 per hour each, public IPv4 addresses about $0.005 per hour each. About $0.10 to $0.15 per hour in total, so a 2-hour session is typically under $1 to a few dollars.
- Fix 1 (Multi-AZ) roughly doubles the RDS instance and storage cost while enabled. Keep it on for only a short test and delete the stack the same day.
- Do not add a NAT gateway (about $0.045 per hour plus data) for this lab. Same-day cleanup is mandatory because the ALB, RDS, and public IPv4 addresses bill by the hour even when idle.

## Step by Step

### Step 1: Time-box and set up the review workspace

Create a plain document with these sections: Requirements, Pillar table, Risks, Fixes, Cost, DR, Exam log. Write down the requirements in measurable form first, because every review answer depends on them.

| Requirement | Number you will use |
|---|---|
| Availability target | 99.9% per month (about 43 minutes of downtime) |
| RTO | 30 minutes for AZ failure, 4 hours for Region loss |
| RPO | 15 minutes (AZ failure), 1 hour (Region loss) |
| Peak load | 3x baseline, weekends |
| Data class | Regulated customer data (encryption at rest and in transit) |
| Monthly budget | $400 |

### Step 2: Inspect the architecture for evidence (optional commands)

If the stack is deployed, gather real evidence instead of guessing. Run these read-only checks:

```bash
# Reliability: DB Multi-AZ, backups, encryption, public exposure
aws rds describe-db-instances --db-instance-identifier lab-mysql \
  --query 'DBInstances[0].[MultiAZ,BackupRetentionPeriod,StorageEncrypted,PubliclyAccessible,DeletionProtection]'

# Reliability / Performance: ASG size, health check type, scaling policies
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-web-asg \
  --query 'AutoScalingGroups[0].[MinSize,MaxSize,DesiredCapacity,HealthCheckType,AvailabilityZones]'
aws autoscaling describe-policies --auto-scaling-group-name lab-web-asg --query 'ScalingPolicies[].PolicyName'

# Security: ALB listeners (is there HTTPS?)
ALB_ARN=$(aws elbv2 describe-load-balancers --names lab-alb --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[].[Port,Protocol]'

# Operational excellence: alarms and tags
aws cloudwatch describe-alarms --query 'MetricAlarms[].AlarmName'
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project --query 'ResourceTagMappingList[].ResourceARN'

# Cost: any budget?
aws budgets describe-budgets --account-id <ACCOUNT_ID> --query 'Budgets[].BudgetName'
```

Console equivalents: RDS > Databases > lab-mysql > Configuration; EC2 > Auto Scaling groups > Details and Automatic scaling; EC2 > Load Balancers > Listeners; CloudWatch > Alarms.

If you did not deploy, write the expected answers from the diagram (for the lab as shipped: Multi-AZ false, retention 1, unencrypted, HTTP only, no scaling policy, no alarms, no budget).

### Step 3: Run the six-pillar review

Fill every row with an answer, the evidence (command output, console screenshot, or diagram line), a rating (OK, Medium, High risk), and a proposed action. The questions follow the intent of the official Well-Architected Framework pillar questions; the real framework has more, so treat this as a focused checklist.

| Pillar | Question | Evidence to collect | Expected finding for this design | Rating |
|---|---|---|---|---|
| Operational Excellence | How do you know the workload is healthy and who is alerted? | `describe-alarms`, dashboards, runbooks | No alarms or dashboards; no runbook | High |
| Operational Excellence | Is infrastructure defined as code and are changes reviewed? | Repo, CloudFormation/Terraform | Built by CLI/console only; not reproducible | Medium |
| Operational Excellence | Are resources tagged for ownership and cost? | Tagging API output | No consistent tags | Medium |
| Security | How is data encrypted at rest and in transit? | `StorageEncrypted`, listeners | RDS unencrypted, ALB HTTP only | High |
| Security | How is access to the DB restricted and credentials managed? | Security groups, secret store | SG-to-SG rule is good; credentials handling to confirm (use Secrets Manager) | Medium |
| Security | Which edge protections exist (WAF, Shield, DDoS)? | ALB attachments | None; Shield Standard only | Medium |
| Security | Are API calls and changes audited? | CloudTrail, Config | Confirm trail enabled | Medium |
| Reliability | Does the data tier survive an AZ failure? | `MultiAZ` | False: single point of failure | High |
| Reliability | Does the compute tier replace failed instances and use both AZs? | ASG health check, AZ list | Two AZs OK; health check type EC2, not ELB | Medium |
| Reliability | Are backups tested and is there a DR plan with RTO/RPO? | Backup retention, restore test | 1-day retention, no tested restore, no cross-Region copy | High |
| Performance Efficiency | Does capacity follow demand? | Scaling policies | Fixed size, no policy; cannot absorb 3x peak | High |
| Performance Efficiency | Are instance types and DB engine suited to the load? | CloudWatch CPU, DB metrics | `t3.micro` burstable credits may be exhausted under sustained load | Medium |
| Performance Efficiency | Is static content cached close to users? | CloudFront | No CDN | Low to Medium |
| Cost Optimization | Do you know the monthly cost and do alerts exist? | Budgets, Cost Explorer | No budget | Medium |
| Cost Optimization | Is pricing model matched to usage (Savings Plans, Reserved, right-sizing)? | Compute Optimizer, Cost Explorer | All on-demand; fine until steady state is known | Low |
| Cost Optimization | Is anything idle or over-provisioned? | Utilization metrics | Likely fine at this size | Low |
| Sustainability | Is utilization high and are efficient resources used (Graviton, scaling in)? | Utilization, instance family | x86 `t3`; consider Graviton (`t4g`) and scale-in | Low |
| Sustainability | Are data retention and Region choice considered? | Lifecycle rules | No retention policy | Low |

### Step 4: Rank the top risks

Score likelihood and impact from 1 to 3 and multiply. Example result for this design:

| Rank | Risk | Pillar | L x I | Business link |
|---|---|---|---|---|
| 1 | Single-AZ RDS: an AZ outage or maintenance takes orders offline and RTO is unknown | Reliability | 3 x 3 = 9 | $2,000 per hour of downtime |
| 2 | Fixed capacity with no scaling policy: promotions will overload two `t3.micro` instances | Performance, Reliability | 3 x 3 = 9 | Peak is 3x baseline |
| 3 | Regulated data unencrypted at rest and HTTP in transit | Security | 2 x 3 = 6 | Compliance |
| 4 | No alarms, no budget, no tested restore | Ops, Cost, Reliability | 3 x 2 = 6 | Unknown failures and bill surprises |
| 5 | 1-day retention, no cross-Region copy | Reliability | 1 x 3 = 3 | Region loss, RPO 1 hour |

Write one sentence per risk linking it to a requirement from Step 1. Reviews fail when findings are not tied to business impact.

### Step 5: Propose three fixes (implement optionally)

Choose the three fixes with the best risk reduction per dollar.

Fix 1 (risk 1): Multi-AZ RDS with longer backups. This gives automatic failover (RTO typically a few minutes, no data loss since replication is synchronous) and point-in-time recovery (RPO of about 5 minutes with PITR).

```bash
aws rds modify-db-instance --db-instance-identifier lab-mysql \
  --multi-az --backup-retention-period 7 --deletion-protection --apply-immediately
aws rds describe-db-instances --db-instance-identifier lab-mysql \
  --query 'DBInstances[0].[DBInstanceStatus,MultiAZ,BackupRetentionPeriod]'
```

Conversion to Multi-AZ takes several minutes. Cost warning: this doubles the DB instance charge, so remove it in cleanup. To test failover (the failure experiment), reboot with failover once the status is `available` and `MultiAZ` is true:

```bash
aws rds reboot-db-instance --db-instance-identifier lab-mysql --force-failover
```

Measure how long until `describe-db-instances` returns to `available` and the endpoint accepts connections again. That number is your measured RTO for an AZ-level database failure.

Fix 2 (risk 2): demand-based scaling and ELB-aware health checks.

```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name lab-web-asg \
  --min-size 2 --max-size 6 --health-check-type ELB --health-check-grace-period 120

aws autoscaling put-scaling-policy --auto-scaling-group-name lab-web-asg \
  --policy-name cpu-target-50 --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":50.0}'
```

Raising `max-size` to 6 caps cost while allowing a 3x peak. Optionally, generate CPU load on the instances for a few minutes (for example with a simple loop run through Systems Manager) and watch `DesiredCapacity` rise. Keep `max-size` small for a lab to avoid surprise charges.

Fix 3 (risk 3 and 4): encryption plus alarms.

Encrypting an existing unencrypted RDS instance requires copying a snapshot with a KMS key and restoring from it (this causes a cutover). For a lab, practice the sequence on a snapshot without switching traffic:

```bash
aws rds create-db-snapshot --db-instance-identifier lab-mysql --db-snapshot-identifier lab-mysql-pre-enc
aws rds wait db-snapshot-available --db-snapshot-identifier lab-mysql-pre-enc
aws rds copy-db-snapshot --source-db-snapshot-identifier lab-mysql-pre-enc \
  --target-db-snapshot-identifier lab-mysql-enc --kms-key-id alias/aws/rds
# Production cutover: restore a new instance from the encrypted snapshot, then repoint the application.
# aws rds restore-db-instance-from-db-snapshot --db-instance-identifier lab-mysql-enc --db-snapshot-identifier lab-mysql-enc ...
```

Add an HTTPS listener with an ACM certificate when you own a domain (DNS validation); without a domain, document it as a required production change. Add alarms and a budget:

```bash
aws sns create-topic --name review-alerts   # then subscribe your email and confirm

aws cloudwatch put-metric-alarm --alarm-name alb-5xx \
  --namespace AWS/ApplicationELB --metric-name HTTPCode_ELB_5XX_Count \
  --dimensions Name=LoadBalancer,Value=<ALB_NAME_SUFFIX> \
  --statistic Sum --period 60 --evaluation-periods 2 --threshold 5 \
  --comparison-operator GreaterThanOrEqualToThreshold --treat-missing-data notBreaching \
  --alarm-actions <SNS_TOPIC_ARN>

aws cloudwatch put-metric-alarm --alarm-name rds-cpu-high \
  --namespace AWS/RDS --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=lab-mysql \
  --statistic Average --period 300 --evaluation-periods 2 --threshold 80 \
  --comparison-operator GreaterThanThreshold --alarm-actions <SNS_TOPIC_ARN>
```

`<ALB_NAME_SUFFIX>` is the part of the load balancer ARN after `loadbalancer/` (for example `app/lab-alb/1234567890abcdef`). Create a monthly Budget as in Week 8 if you have not already.

Record in your review, for each fix: the risk it addresses, the pillar, the extra monthly cost, and the residual risk that remains.

### Step 6: Pricing Calculator outline

Open https://calculator.aws and build an estimate named `order-portal`. Do not trust numbers from this repo; use the calculator and the official pricing pages for current prices. Outline:

| Line item | Settings to enter | Notes |
|---|---|---|
| Amazon EC2 | 2 to 6 instances, `t3.small` or `t3.medium` for production, Linux, On-Demand vs 1-year Compute Savings Plan | Compare both columns |
| Application Load Balancer | 1 ALB, estimated LCU from requests per second | Hourly plus LCU |
| Amazon RDS for MySQL | `db.t3.medium`, Multi-AZ, 100 GB gp3, 7-day backups | Multi-AZ doubles instance cost |
| Data transfer | Outbound GB per month to the internet | Often the surprise line |
| Public IPv4 addresses | Count of addresses | Charged hourly |
| CloudWatch | Alarms, logs ingestion, dashboards | Small |
| AWS Backup / snapshots | Storage and cross-Region copy GB | For the DR requirement |
| NAT gateway | Only if private instances need outbound internet | Hourly plus per GB |

Compute the total and compare it with the $400 budget. Write the monthly estimate, the biggest line item, and one lever to cut it (Savings Plan, right-sizing, Graviton, scheduled scale-down).

### Step 7: Define RTO/RPO and choose the DR strategy

| Failure | RTO target | RPO target | Mechanism |
|---|---|---|---|
| Instance failure | under 5 minutes | 0 | ASG with ELB health checks |
| AZ failure | under 30 minutes | 0 (synchronous Multi-AZ) | Multi-AZ RDS plus ASG across 2 AZs |
| Region failure | under 4 hours | under 1 hour | Backup and restore with cross-Region AWS Backup copy; consider pilot light if the business wants under 1 hour |
| Bad deployment or data corruption | under 1 hour | 15 minutes | RDS point-in-time restore, blue/green or rollback |

Justify the Region-level row by comparing strategy cost to downtime cost. Backup and restore is the cheapest, and at $2,000 per hour a 4-hour outage costs about $8,000, which can justify pilot light (replicated database in the second Region, compute stopped) if a Region failure is considered likely enough. State your decision and your reasoning.

### Step 8: Timed mock exam block (20 questions)

Open `29-Simulados-e-Questoes/mini-simulados-por-dominio.md`. It contains four blocks of five questions (Resilience, Performance, Security, Cost).

1. Close all notes. Set a timer for 40 minutes (the file suggests 8 to 10 minutes per block; the real exam averages about 2 minutes per question, so 40 minutes for 20 is already faster than exam pace).
2. Answer all 20 questions on paper without opening the "Show answer" sections.
3. Read the key requirement in each question before choosing; eliminate two options first.
4. When the timer ends, grade yourself and compute the percentage per block.

| Block | Domain | Score (of 5) | Time used |
|---|---|---|---|
| A | Resilience | | |
| B | Performance | | |
| C | Security | | |
| D | Cost | | |
| Total | | /20 | |

Target: 16 or more of 20 (80%). The real exam is scored on a scale of 100 to 1000 with a passing score of 720, but the repo's blocks are practice only, so treat your percentage as a trend, not a prediction.

### Step 9: Log every mistake

For each wrong or guessed question, append an entry to `29-Simulados-e-Questoes/caderno-de-erros.md` using the repo's template:

```
- Domain: Resilient Architectures
- Topic: RDS Multi-AZ vs read replica
- Option chosen: C (read replica)
- Correct option: B (Multi-AZ)
- Technical reason for the mistake: Confused performance scaling with high availability.
- Decision rule to avoid repeating it: Failover/high availability = Multi-AZ; read scaling = replica.
```

Classify each mistake as Concept, Reading, Trade-off, or Rushing (the notebook's categories). Then follow the notebook's weekly routine: group mistakes by topic, re-read only the relevant module cheat sheet, redo five questions of the same type, and update the decision rule. Flag any topic missed in two or more exams as high recurrence.

## Validation

- Your review has an answer, evidence, and rating for every row of the pillar table, and every high-risk row links to a business number from Step 1.
- Risk register is ranked and the three fixes map to the top risks.
- If you deployed: `describe-db-instances` shows `MultiAZ` true and retention 7; `describe-policies` shows `cpu-target-50`; the two alarms exist; the failover reboot returned to `available` and you recorded the elapsed time.
- The Pricing Calculator estimate is saved or its line items are listed with a total compared against $400.
- You scored the 20 questions, calculated per-domain percentages, and logged each mistake with a decision rule.

Failure experiment (choose at least one): (a) force a Multi-AZ failover as in Fix 1 and measure the outage; (b) terminate one web instance through the console and confirm the ASG with ELB health checks replaces it while the ALB DNS name keeps answering from the other AZ; (c) on paper, walk a Region failure through your DR table and find the slowest step. Record what survived and what did not.

## Cleanup

If you deployed the stack, delete in this order the same day (Multi-AZ is the most expensive addition):

```bash
# Alarms, topic, extra snapshots
aws cloudwatch delete-alarms --alarm-names alb-5xx rds-cpu-high
aws sns delete-topic --topic-arn <SNS_TOPIC_ARN>
aws rds delete-db-snapshot --db-snapshot-identifier lab-mysql-pre-enc
aws rds delete-db-snapshot --db-snapshot-identifier lab-mysql-enc

# Compute tier first
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name lab-web-asg --force-delete

# Load balancer, listeners, target group (use the names from your 28-Labs-Praticos stack)
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 delete-target-group --target-group-arn <TG_ARN>

# Database: remove deletion protection first, then delete
aws rds modify-db-instance --db-instance-identifier lab-mysql --no-deletion-protection --apply-immediately
aws rds delete-db-instance --db-instance-identifier lab-mysql --skip-final-snapshot --delete-automated-backups

# Budget created in this lab, if any
aws budgets delete-budget --account-id <ACCOUNT_ID> --budget-name <NAME>
```

Then finish the remaining cleanup from the `28-Labs-Praticos/lab.md` Cleanup section (launch template, security groups, subnets, route tables, internet gateway, VPC). Wait for the target group deletion to succeed only after the load balancer is gone. Check that no snapshots, Elastic IPs, or load balancers remain in the console. Paper-only path: nothing to delete.

## Exam Takeaway

Final readiness checklist (tick when true):

- [ ] I can name the six pillars and give one design principle and one AWS service for each.
- [ ] I can pick a DR strategy from RTO/RPO numbers and justify the cost trade-off.
- [ ] I know the difference between Multi-AZ (availability, synchronous, same Region) and read replicas (read scaling, asynchronous, can be cross-Region).
- [ ] I choose the most managed, least operational-overhead option when the question says "minimize operational overhead".
- [ ] I can read a question and underline the constraint words (lowest cost, least operational overhead, most resilient, near real time, private).
- [ ] I scored 80% or higher on two consecutive timed 20-question blocks, and my error notebook shows no topic with high recurrence.
- [ ] I have read the glossary entries for terms I confused and checked the official exam guide and AWS documentation linked in the resources module.
- [ ] I know the practicalities: exam format, time per question, and that unanswered questions score as wrong, so I never leave one blank. Verify the current exam details on the official AWS certification page.

Domain mapping for what this lab exercised:

- Design Resilient Architectures: Multi-AZ, ASG health checks, backups, DR strategies, RTO/RPO.
- Design High-Performing Architectures: scaling policy, instance sizing, caching and CDN choices in the review.
- Design Secure Architectures: encryption at rest and in transit, security group references, WAF and edge protection, auditing.
- Design Cost-Optimized Architectures: Pricing Calculator, Savings Plans versus On-Demand, Budgets, right-sizing, cost of DR.
- Well-Architected Framework underpins all four exam domains; questions often hide the pillar in the wording ("least privilege" is Security, "recover" is Reliability, "reduce cost" is Cost Optimization).

## Stretch Goals

1. Use the AWS Well-Architected Tool in the console (Well-Architected Tool > Define workload) to record the same review, answer the official questions for two pillars, and compare its improvement plan with your risk register.
2. Convert the reviewed architecture to Terraform or CloudFormation so the "infrastructure as code" finding becomes Medium to OK, and add a stack drift check.
3. Add a read replica in a second Region and compare it against your cross-Region backup copy for the Region-failure row.
4. Retake a different 20-question set from `29-Simulados-e-Questoes/questoes.md` three days later and compare your per-domain scores to prove the error-notebook rules worked.


---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
