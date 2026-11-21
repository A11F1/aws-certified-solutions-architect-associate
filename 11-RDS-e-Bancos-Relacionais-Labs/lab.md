# Guided Lab: RDS Multi-AZ Failover, Read Replica and Point-in-Time Restore

## Objective

Provision a tiny RDS for MySQL instance and prove, with your own eyes, the three RDS features that the exam keeps contrasting: Multi-AZ (high availability), read replicas (read scaling), and automated backups with point-in-time restore (recovery).

## Scenario

A small team runs an orders database. They want it to survive an Availability Zone failure, offload reporting queries, and recover from an accidental `DELETE` without a data-loss argument. You build all three on a `db.t3.micro` instance and test each one.

## Services Used

- Amazon RDS for MySQL
- Amazon VPC (default VPC and a security group)
- AWS Secrets Manager (managed master password)
- AWS CLI v2 (or AWS CloudShell)

## Prerequisites

- AWS account, region `us-east-1`, an IAM user/role with RDS, EC2 (security groups) and Secrets Manager permissions.
- The default VPC with its default subnet group (the lab uses it).
- A MySQL client (`mysql`) in CloudShell or on your laptop. In CloudShell: `sudo dnf install -y mariadb105`.
- Your public IP: `curl -s https://checkip.amazonaws.com`.

## Expected Cost

- Single-AZ `db.t3.micro` with 20 GiB gp3 is Free Tier eligible (750 hours/month for 12 months on legacy Free Tier accounts; newer accounts use the credit-based plan).
- Multi-AZ and read replicas are NOT covered by the Free Tier: you pay for a second instance and its storage. Expect roughly $0.02-$0.05 per instance-hour, so a 1-2 hour lab costs well under $1.
- Backups up to the size of your database are free; restored instances are billed like any instance.
- Avoid: leaving the instances running overnight, leaving snapshots around, and making the database publicly accessible beyond your own /32.

## Step by Step

### Step 1: Create a security group that allows only your IP

Console: EC2 > Security Groups > Create security group. Name `lab-rds-sg`, default VPC, inbound rule MySQL/Aurora (3306) from `<YOUR_IP>/32`.

```bash
export AWS_REGION=us-east-1
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SG_ID=$(aws ec2 create-security-group --group-name lab-rds-sg \
  --description "Lab RDS access from my IP" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 3306 --cidr <YOUR_IP>/32
```

### Step 2: Create a Multi-AZ MySQL instance with automated backups

Console: RDS > Create database > Standard create > MySQL > Templates: Dev/Test > Availability: Multi-AZ DB instance > identifier `lab-mysql` > Master password: Managed in AWS Secrets Manager > Instance class `db.t3.micro` > Storage gp3 20 GiB, disable storage autoscaling > Public access Yes (lab only) > security group `lab-rds-sg` > Additional configuration: backup retention 7 days.

```bash
aws rds create-db-instance \
  --db-instance-identifier lab-mysql \
  --engine mysql \
  --db-instance-class db.t3.micro \
  --allocated-storage 20 --storage-type gp3 \
  --master-username admin --manage-master-user-password \
  --multi-az \
  --backup-retention-period 7 \
  --publicly-accessible \
  --vpc-security-group-ids $SG_ID \
  --no-deletion-protection

aws rds wait db-instance-available --db-instance-identifier lab-mysql
```

Multi-AZ creation takes 10-20 minutes. Note that Multi-AZ keeps a synchronous standby you cannot connect to; the same endpoint always points to the current primary.

### Step 3: Load some data

```bash
ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier lab-mysql \
  --query 'DBInstances[0].Endpoint.Address' --output text)
SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier lab-mysql \
  --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text)
PW=$(aws secretsmanager get-secret-value --secret-id $SECRET_ARN \
  --query SecretString --output text | python3 -c 'import sys,json;print(json.load(sys.stdin)["password"])')

mysql -h $ENDPOINT -u admin -p"$PW" -e "
CREATE DATABASE shop; USE shop;
CREATE TABLE orders (id INT AUTO_INCREMENT PRIMARY KEY, item VARCHAR(50), created TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
INSERT INTO orders (item) VALUES ('keyboard'),('mouse'),('monitor');"
```

### Step 4: Force a Multi-AZ failover

Record the current AZ, then reboot with failover.

```bash
aws rds describe-db-instances --db-instance-identifier lab-mysql \
  --query 'DBInstances[0].[AvailabilityZone,SecondaryAvailabilityZone,MultiAZ]'

aws rds reboot-db-instance --db-instance-identifier lab-mysql --force-failover

# Watch your own connection drop and recover (usually 60-120 seconds)
while true; do date +%T; mysql -h $ENDPOINT -u admin -p"$PW" -e "SELECT @@hostname, COUNT(*) FROM shop.orders" 2>&1 | tail -1; sleep 5; done
```

Console equivalent: RDS > Databases > `lab-mysql` > Actions > Reboot > check "Reboot with failover?".

Afterwards the primary and secondary AZs are swapped, while the endpoint DNS name is unchanged. Check RDS > Logs & events for the `Multi-AZ instance failover started/completed` events.

### Step 5: Create a read replica

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier lab-mysql-replica \
  --source-db-instance-identifier lab-mysql \
  --db-instance-class db.t3.micro \
  --publicly-accessible
aws rds wait db-instance-available --db-instance-identifier lab-mysql-replica

REPLICA=$(aws rds describe-db-instances --db-instance-identifier lab-mysql-replica \
  --query 'DBInstances[0].Endpoint.Address' --output text)
mysql -h $REPLICA -u admin -p"$PW" -e "SELECT COUNT(*) FROM shop.orders"
```

Failure experiment: try to write to the replica. It is read-only and must be rejected.

```bash
mysql -h $REPLICA -u admin -p"$PW" -e "INSERT INTO shop.orders (item) VALUES ('should-fail')"
# ERROR 1290 (HY000): The MySQL server is running with the --read-only option
```

The replica has its own endpoint; your application must send reads there explicitly. Replication is asynchronous.

### Step 6: Point-in-time restore after a "mistake"

```bash
date -u +%FT%TZ    # note this time as <GOOD_TIME>, then wait ~5 minutes
mysql -h $ENDPOINT -u admin -p"$PW" -e "DELETE FROM shop.orders"
```

PITR needs transaction logs that RDS uploads about every 5 minutes, so check the latest restorable time:

```bash
aws rds describe-db-instances --db-instance-identifier lab-mysql \
  --query 'DBInstances[0].LatestRestorableTime'
```

Restore to a point just before the delete into a NEW instance (RDS never restores in place):

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier lab-mysql \
  --target-db-instance-identifier lab-mysql-restored \
  --restore-time <GOOD_TIME> \
  --db-instance-class db.t3.micro \
  --no-multi-az --publicly-accessible
aws rds wait db-instance-available --db-instance-identifier lab-mysql-restored
```

Console equivalent: RDS > Databases > `lab-mysql` > Actions > Restore to point in time. Pick a custom date and time, give a new identifier.

## Validation

- After the forced failover, `SecondaryAvailabilityZone` and `AvailabilityZone` are swapped and the same endpoint works again.
- The replica returns the same row count as the primary, and writes to it fail with the read-only error.
- On the restored instance, `SELECT COUNT(*) FROM shop.orders` returns 3 (the endpoint of the restored instance is different from the original).
- The original instance still shows the empty table, proving the restore created a separate copy.

## Cleanup

Delete in this order (replica and restored copy first, then the source). Skip the final snapshot unless you want it, because snapshots are billed.

```bash
aws rds delete-db-instance --db-instance-identifier lab-mysql-replica --skip-final-snapshot
aws rds delete-db-instance --db-instance-identifier lab-mysql-restored --skip-final-snapshot
aws rds wait db-instance-deleted --db-instance-identifier lab-mysql-replica
aws rds wait db-instance-deleted --db-instance-identifier lab-mysql-restored
aws rds delete-db-instance --db-instance-identifier lab-mysql --skip-final-snapshot --delete-automated-backups
aws rds wait db-instance-deleted --db-instance-identifier lab-mysql
aws ec2 delete-security-group --group-id $SG_ID
```

Also confirm no manual snapshots remain (RDS > Snapshots) and that the managed secret was removed.

## Exam Takeaway

Multi-AZ is for availability: synchronous standby, automatic failover, same endpoint, no read traffic. Read replicas are for performance: asynchronous, separate endpoints, can be cross-Region, and can be promoted (a DR option, not an automatic failover). Automated backups plus PITR are for recovery from logical errors, and always restore into a new instance. When a question says "high availability" choose Multi-AZ; "read-heavy / reporting" choose read replicas; "undo a bad delete" choose PITR.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
