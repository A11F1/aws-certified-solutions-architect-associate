# Guided Lab: Run nginx on ECS Fargate, Scale the Service and Self-Heal

## Objective

Run a public nginx image on Amazon ECS with the Fargate launch type, with no servers to manage. You define a task definition, create a service, read container logs in CloudWatch, scale the service, and prove that ECS replaces a stopped task.

## Scenario

A team wants to containerize a small web front end without managing EC2 instances. To keep the cost down, the lab runs a single Fargate task with a public IP and a restrictive security group, instead of an Application Load Balancer. The ALB variant is covered in the cost notes and the exam takeaway.

## Services Used

- Amazon ECS (cluster, task definition, service) on AWS Fargate
- Amazon CloudWatch Logs
- AWS IAM (task execution role)
- Amazon VPC (default VPC, security group)

## Prerequisites

- AWS account, region `us-east-1`, permissions for ECS, IAM role creation, EC2 networking and CloudWatch Logs.
- AWS CLI v2 and `curl`. Your public IP: `curl -s https://checkip.amazonaws.com`.
- No Docker needed: the image is pulled from the public Amazon ECR gallery mirror of the official nginx image.

## Expected Cost

- Fargate 0.25 vCPU and 0.5 GB: about $0.012 per hour per task (roughly $0.04 per vCPU-hour and $0.0044 per GB-hour). Two tasks for an hour is about 2-3 cents.
- Each public IPv4 address costs about $0.005 per hour while the task runs.
- CloudWatch Logs: negligible for nginx access logs at this scale.
- Comparison only (do not create): an ALB costs about $0.0225/hour plus LCUs, which is roughly $16-20 per month when idle, and an EKS cluster has a control-plane fee of $0.10/hour (about $73/month) before any worker costs, so this lab does NOT create EKS.
- Avoid: leaving the service running (desired count above 0) after the lab.

## Step by Step

### Step 1: Create the cluster, log group and execution role

Console: ECS > Clusters > Create cluster > name `lab-cluster` > Infrastructure: AWS Fargate (serverless).

```bash
export AWS_REGION=us-east-1
aws ecs create-cluster --cluster-name lab-cluster
aws logs create-log-group --log-group-name /ecs/lab-nginx
aws logs put-retention-policy --log-group-name /ecs/lab-nginx --retention-in-days 1

# Execution role (skip if ecsTaskExecutionRole already exists in your account)
cat > trust.json <<'EOF'
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Principal": { "Service": "ecs-tasks.amazonaws.com" }, "Action": "sts:AssumeRole" }] }
EOF
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

The execution role lets ECS pull images and write logs on your behalf; a separate task role (not needed here) is what your application code uses to call AWS APIs.

### Step 2: Register the task definition

Console: ECS > Task definitions > Create new > Fargate, 0.25 vCPU / 0.5 GB, container `web`, image `public.ecr.aws/docker/library/nginx:latest`, port 80, log driver awslogs.

```bash
cat > taskdef.json <<EOF
{
  "family": "lab-nginx",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "web",
    "image": "public.ecr.aws/docker/library/nginx:latest",
    "essential": true,
    "portMappings": [{ "containerPort": 80, "protocol": "tcp" }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/lab-nginx",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "web"
      }
    }
  }]
}
EOF
aws ecs register-task-definition --cli-input-json file://taskdef.json
```

### Step 3: Create the security group and the service

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID --query 'Subnets[0:2].SubnetId' --output text | tr '\t' ',')
SG_ID=$(aws ec2 create-security-group --group-name lab-ecs-sg --description "nginx from my IP" \
  --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr <YOUR_IP>/32

aws ecs create-service --cluster lab-cluster --service-name lab-nginx-svc \
  --task-definition lab-nginx --desired-count 1 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNETS],securityGroups=[$SG_ID],assignPublicIp=ENABLED}"
aws ecs wait services-stable --cluster lab-cluster --services lab-nginx-svc
```

Console: ECS > Clusters > `lab-cluster` > Services > Create > Launch type Fargate, task definition `lab-nginx`, desired tasks 1, networking: default VPC, security group `lab-ecs-sg`, Public IP turned on.

A public IP in a public subnet avoids NAT Gateway charges (about $0.045/hour plus data) that you would incur for private subnets.

### Step 4: Reach the container and read logs

```bash
TASK_ARN=$(aws ecs list-tasks --cluster lab-cluster --service-name lab-nginx-svc --query 'taskArns[0]' --output text)
ENI=$(aws ecs describe-tasks --cluster lab-cluster --tasks $TASK_ARN \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text)
PUBIP=$(aws ec2 describe-network-interfaces --network-interface-ids $ENI \
  --query 'NetworkInterfaces[0].Association.PublicIp' --output text)
curl -sI http://$PUBIP | head -3          # HTTP/1.1 200 OK, Server: nginx
aws logs tail /ecs/lab-nginx --since 10m  # your curl request appears as an access log line
```

### Step 5: Scale the service

```bash
aws ecs update-service --cluster lab-cluster --service lab-nginx-svc --desired-count 2
aws ecs wait services-stable --cluster lab-cluster --services lab-nginx-svc
aws ecs describe-services --cluster lab-cluster --services lab-nginx-svc \
  --query 'services[0].[desiredCount,runningCount]'
```

Without a load balancer, each task has its own public IP. In production, an ALB target group would spread traffic across tasks, and Service Auto Scaling (target tracking on CPU or `ALBRequestCountPerTarget`) would change the desired count automatically.

### Step 6: Failure experiment, stop a task

```bash
aws ecs stop-task --cluster lab-cluster --task $(aws ecs list-tasks --cluster lab-cluster --service-name lab-nginx-svc --query 'taskArns[0]' --output text) --reason "lab failure test"
sleep 20
aws ecs describe-services --cluster lab-cluster --services lab-nginx-svc \
  --query 'services[0].events[0:4].message'
aws ecs list-tasks --cluster lab-cluster --service-name lab-nginx-svc
```

The service scheduler notices running < desired and starts a replacement task automatically (it will have a new ARN and a new public IP).

Security test: from a different network (or after deleting your /32 rule), `curl --max-time 5 http://$PUBIP` times out because the security group does not allow the source.

## Validation

- `curl` returns `HTTP/1.1 200 OK` with `Server: nginx` from your allowed IP.
- CloudWatch log group `/ecs/lab-nginx` contains streams prefixed `web/web/<task-id>` with access log lines.
- `runningCount` reached 2 after scaling and returned to the desired count after you stopped a task.
- Service events show "has started 1 tasks" after the stop.

## Cleanup

Set desired count to 0 first, then delete the service, then supporting resources.

```bash
aws ecs update-service --cluster lab-cluster --service lab-nginx-svc --desired-count 0
aws ecs delete-service --cluster lab-cluster --service lab-nginx-svc --force
aws ecs wait services-inactive --cluster lab-cluster --services lab-nginx-svc
aws ecs delete-cluster --cluster lab-cluster
aws ecs deregister-task-definition --task-definition lab-nginx:1
aws logs delete-log-group --log-group-name /ecs/lab-nginx
# Security group can only be deleted after the task network interfaces are gone (about a minute)
aws ec2 delete-security-group --group-id $SG_ID
# Optional, only if you created it for this lab:
aws iam detach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name ecsTaskExecutionRole
```

## Exam Takeaway

Fargate is the "serverless containers" answer: you pay per vCPU and memory for running tasks and manage no instances, while ECS on EC2 gives more control (GPU, custom AMIs, Spot, cheaper at sustained high utilization) but means patching and capacity management. ECS is the simpler AWS-native orchestrator, while EKS is chosen for Kubernetes portability and carries a $0.10/hour control-plane fee per cluster. Know the roles: the task execution role pulls images and writes logs; the task role gives the application AWS permissions. For production, place tasks in private subnets behind an ALB, and let Service Auto Scaling adjust the desired count.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
