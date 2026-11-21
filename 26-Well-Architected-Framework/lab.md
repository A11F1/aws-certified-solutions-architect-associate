# Guided Lab: Run a Well-Architected Review and Gather Evidence

## Objective

Define a small sample workload in the AWS Well-Architected Tool, answer questions for each of the six pillars, review high-risk items (HRIs), and back your answers with evidence from Trusted Advisor and a CloudWatch dashboard.

## Scenario

A small static site with a Lambda API and a DynamoDB table (or a single EC2 instance if that is what you have) runs in a personal account. You act as the architect reviewing it and produce an improvement plan mapped to the pillars.

## Services Used

- AWS Well-Architected Tool
- AWS Trusted Advisor
- Amazon CloudWatch (dashboard, alarm)
- AWS IAM, AWS CloudTrail (as evidence sources)

## Prerequisites

- AWS CLI v2, Region `us-east-1`.
- IAM permissions for `wellarchitected:*`, `cloudwatch:*` and `support:*` (Trusted Advisor).
- A small workload to review. If you have none, review a hypothetical one: a single `t3.micro` web server with a public IP, a single-AZ RDS database and no backups configured. Reviewing a weak design makes the exercise more instructive.
- Trusted Advisor: all accounts get the core security and service-quota checks; the full check set needs a Business, Enterprise On-Ramp or Enterprise Support plan, which you should not buy for this lab.

## Expected Cost

The Well-Architected Tool is free. Trusted Advisor core checks are free. A CloudWatch dashboard is free for the first three (up to 50 metrics); the 4th and beyond cost about 3 USD per month each. A standard alarm is about 0.10 USD/month, with the first 10 free. Do not buy a support plan for this lab, and delete the dashboard afterwards.

## Step by Step

### Step 1: Define the workload

Console: AWS Well-Architected Tool > Define workload. Name `lab-sample-workload`, environment Pre-production, Regions `us-east-1`, review owner your name, lens `AWS Well-Architected Framework`.

```bash
aws wellarchitected create-workload \
  --workload-name lab-sample-workload \
  --description "Sample workload for a learning review" \
  --environment PREPRODUCTION \
  --review-owner "<YOUR_NAME>" \
  --aws-regions us-east-1 \
  --lenses wellarchitected \
  --region us-east-1
aws wellarchitected list-workloads --region us-east-1
```

### Step 2: Answer a few questions per pillar

Console: open the workload > Start reviewing > pick Operational excellence and answer, then repeat. For each question select the best practices you actually follow (or "None of these"). Aim for two or three questions per pillar, enough to generate findings. Suggested questions:

- Operational excellence: how do you reduce defects and ease remediation (deployment automation)? how do you understand workload health?
- Security: how do you manage identities? how do you protect data at rest?
- Reliability: how do you design your workload service architecture? how do you back up data?
- Performance efficiency: how do you select compute resources?
- Cost optimization: how do you monitor usage and cost?
- Sustainability: how do you select Regions and use managed services to reduce impact?

```bash
aws wellarchitected list-answers --workload-id <WORKLOAD_ID> --lens-alias wellarchitected --pillar-id security --region us-east-1
aws wellarchitected get-answer --workload-id <WORKLOAD_ID> --lens-alias wellarchitected --question-id <QUESTION_ID> --region us-east-1
aws wellarchitected update-answer --workload-id <WORKLOAD_ID> --lens-alias wellarchitected \
  --question-id <QUESTION_ID> --selected-choices <CHOICE_ID_1> <CHOICE_ID_2> \
  --notes "Evidence: CloudTrail enabled in all Regions" --region us-east-1
```

The `notes` field is where you record evidence. Use the console for most answers; it is much faster than the CLI.

### Step 3: Review high-risk items

Console: workload > Overview shows counts of High and Medium risk items per pillar. Open Improvement plan to see the prioritized list, each with linked guidance.

```bash
aws wellarchitected get-lens-review --workload-id <WORKLOAD_ID> --lens-alias wellarchitected --region us-east-1 \
  --query "LensReview.PillarReviewSummaries[].[PillarName,RiskCounts]"
aws wellarchitected list-lens-review-improvements --workload-id <WORKLOAD_ID> --lens-alias wellarchitected --region us-east-1
```

Pick the top three HRIs and write a one-line remediation for each. Save a milestone (console: Save milestone, or `aws wellarchitected create-milestone --workload-id <WORKLOAD_ID> --milestone-name baseline`) so you can compare after fixes.

### Step 4: Gather evidence with Trusted Advisor

Console: Trusted Advisor > Recommendations. Review Security (for example, MFA on root, public S3 buckets, open security groups), Cost optimization (idle resources, only with paid plans), Fault tolerance, Performance and Service limits.

```bash
aws support describe-trusted-advisor-checks --language en --region us-east-1   # needs Business or higher support
aws trustedadvisor list-recommendations --region us-east-1                       # newer API; available per account plan
```

If the CLI call returns `SubscriptionRequiredException`, use the console for the free core checks and note the limitation. Also supplement with IAM Credential Report and CloudTrail status:

```bash
aws iam generate-credential-report && aws iam get-credential-report --query Content --output text | base64 --decode
aws cloudtrail describe-trails --query "trailList[].[Name,IsMultiRegionTrail]"
```

### Step 5: Gather evidence with a CloudWatch dashboard and alarm

Console: CloudWatch > Dashboards > Create dashboard `lab-wa-dashboard` > add widgets for your workload (for EC2: `CPUUtilization`; for Lambda: `Errors` and `Duration`; for DynamoDB: `ConsumedReadCapacityUnits`).

```bash
cat > dash.json <<'EOF'
{ "widgets": [{ "type": "metric", "x": 0, "y": 0, "width": 12, "height": 6,
  "properties": { "region": "us-east-1", "title": "Lambda errors", "stat": "Sum", "period": 300,
    "metrics": [["AWS/Lambda", "Errors", "FunctionName", "<FUNCTION_NAME>"]] } }] }
EOF
aws cloudwatch put-dashboard --dashboard-name lab-wa-dashboard --dashboard-body file://dash.json
aws cloudwatch put-metric-alarm --alarm-name lab-errors-alarm --namespace AWS/Lambda --metric-name Errors \
  --dimensions Name=FunctionName,Value=<FUNCTION_NAME> --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold --treat-missing-data notBreaching
```

A dashboard plus an alarm is evidence for the operational excellence question about understanding workload health.

### Step 6: Map findings to the six pillars

Fill in a table like this in your notes, one row per finding:

| Finding | Pillar | Evidence | Fix |
|---|---|---|---|
| Single-AZ database, no backups | Reliability | Answer + RDS console | Multi-AZ, automated backups |
| Root account without MFA | Security | Trusted Advisor | Enable MFA |
| No budget or cost tags | Cost optimization | Cost Explorer empty by tag | Budgets, tag policy |
| Manual console deployments | Operational excellence | Answers | IaC and CI/CD |
| Fixed oversized instance | Performance efficiency | CloudWatch CPU under 10% | Right-size or serverless |
| Always-on non-production | Sustainability | Utilization | Schedule shutdown, use managed services |

After fixing one item, update the answer and compare with the baseline milestone.

## Validation

- The workload appears in `list-workloads` and shows answered questions for all six pillars.
- The improvement plan lists at least three HRIs, each mapped to a pillar.
- You have one piece of evidence per HRI (Trusted Advisor check, dashboard graph, CloudTrail or credential report output).
- A milestone named `baseline` exists.

## Cleanup

```bash
aws cloudwatch delete-alarms --alarm-names lab-errors-alarm
aws cloudwatch delete-dashboards --dashboard-names lab-wa-dashboard
aws wellarchitected delete-workload --workload-id <WORKLOAD_ID> --client-request-token $(uuidgen) --region us-east-1
```

Also delete any sample resources (instances, databases) you launched only to be reviewed.

## Exam Takeaway

The Well-Architected Framework has six pillars: operational excellence, security, reliability, performance efficiency, cost optimization and sustainability. Exam questions ask you to pick the option that best fits a pillar, and good answers often trade one pillar against another (for example, Multi-AZ improves reliability but raises cost). The Well-Architected Tool finds risks through questions, while Trusted Advisor checks actual account configuration against best practices; use them together.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
