# Guided Lab: Database Selection, RDS Multi-AZ vs Read Replica, Snapshot Restore and a DynamoDB Comparison

## Objective

Launch a small RDS MySQL instance, then compare Multi-AZ (high availability) with read replicas (read scaling), use a custom parameter group, take and restore a snapshot, and finish by doing the same job in DynamoDB to see when each engine fits.

## Scenario

A startup runs an orders database. It needs relational integrity for orders, automatic failover for outages, a way to offload reporting reads, and a safe way to recover from a bad change. A separate session-style lookup workload is a candidate for DynamoDB.

## Services Used

- Amazon RDS for MySQL (db.t3.micro or db.t4g.micro), Multi-AZ, read replica, snapshots, parameter groups
- Amazon DynamoDB (on-demand)
- AWS CLI v2 and the `mysql` client (optional)

## Prerequisites

- Personal AWS account, non-root IAM identity with RDS, EC2 (for security groups), and DynamoDB permissions, region `us-east-1`.
- The default VPC and its default DB subnet group (the console creates one automatically; the CLI uses the default VPC subnets if no group is given).
- Your public IP as `<YOUR_IP>` if you want to connect from your laptop. The password `<DB_PASSWORD>` is a placeholder: choose your own and never commit it.

## Expected Cost

RDS is the cost risk here. A single-AZ db.t3.micro MySQL instance is Free Tier eligible (750 hours per month for 12 months on older accounts; check your account's plan, since newer accounts use a credits-based free plan). Outside the free tier it costs roughly $0.017 per hour plus about $0.115 per GB-month of gp2/gp3 storage. Multi-AZ doubles the instance and storage cost, and a read replica costs about the same as another instance, so expect under $1 if you finish in one to two hours. Snapshots beyond the allocated storage are billed per GB-month. DynamoDB on-demand with a few items costs effectively nothing. Delete everything promptly: a forgotten Multi-AZ instance plus replica is the equivalent of three running databases.

## Step by Step

### Step 1: Create a parameter group and the database

Console: RDS > Parameter groups > Create (family `mysql8.0`, name `lab10-params`), then Databases > Create database > Standard create > MySQL > Templates: Free tier (or Dev/Test) > DB instance `lab10-db`, master username `admin`, a password, db.t3.micro, 20 GiB gp2, Public access: No (or Yes only if you test from your laptop), automated backups enabled.

```bash
aws rds create-db-parameter-group --db-parameter-group-name lab10-params \
  --db-parameter-group-family mysql8.0 --description "lab10"
aws rds modify-db-parameter-group --db-parameter-group-name lab10-params \
  --parameters "ParameterName=max_connections,ParameterValue=100,ApplyMethod=immediate"

aws rds create-db-instance --db-instance-identifier lab10-db \
  --engine mysql --db-instance-class db.t3.micro --allocated-storage 20 \
  --master-username admin --master-user-password '<DB_PASSWORD>' \
  --db-parameter-group-name lab10-params \
  --backup-retention-period 1 --no-multi-az --no-publicly-accessible \
  --storage-encrypted
aws rds wait db-instance-available --db-instance-identifier lab10-db
```

This takes about 5 to 10 minutes. Dynamic parameters like `max_connections` apply immediately, while static parameters show `pending-reboot`. Note that `--backup-retention-period` greater than 0 is also what enables read replicas and point-in-time recovery.

### Step 2: Convert to Multi-AZ and trigger a failover

Console: select `lab10-db` > Modify > Multi-AZ deployment: Create a standby instance > Apply immediately.

```bash
aws rds modify-db-instance --db-instance-identifier lab10-db --multi-az --apply-immediately
aws rds wait db-instance-available --db-instance-identifier lab10-db
aws rds describe-db-instances --db-instance-identifier lab10-db \
  --query 'DBInstances[0].[MultiAZ,AvailabilityZone,SecondaryAvailabilityZone,Endpoint.Address]'
```

Failure experiment: reboot with failover and watch the AZ change while the endpoint DNS name stays the same.

```bash
aws rds reboot-db-instance --db-instance-identifier lab10-db --force-failover
aws rds describe-events --source-type db-instance --source-identifier lab10-db --duration 30
```

Look for "Multi-AZ instance failover started/completed" events. Failover typically takes 60 to 120 seconds, and the application reconnects to the same endpoint. Then note the key point: the standby serves no traffic, so Multi-AZ gives availability, not read scaling.

### Step 3: Add a read replica

Turn Multi-AZ off first if you want to cut cost (`modify-db-instance --no-multi-az --apply-immediately`), then:

```bash
aws rds create-db-instance-read-replica --db-instance-identifier lab10-replica \
  --source-db-instance-identifier lab10-db --db-instance-class db.t3.micro
aws rds wait db-instance-available --db-instance-identifier lab10-replica
aws rds describe-db-instances --db-instance-identifier lab10-replica \
  --query 'DBInstances[0].[Endpoint.Address,ReadReplicaSourceDBInstanceIdentifier]'
```

The replica has its own endpoint, replicates asynchronously, and can be queried (read-only). If you have a MySQL client and network access (a temporary instance in the same VPC, or public access with a security group rule for `<YOUR_IP>/32` on port 3306):

```bash
mysql -h <REPLICA_ENDPOINT> -u admin -p -e "CREATE DATABASE x;"   # fails: read-only
```

Expect error 1290 (the server is running with `--read-only`). You can also promote it (`aws rds promote-read-replica --db-instance-identifier lab10-replica`), which makes it a standalone instance and breaks replication, a disaster-recovery pattern, not a failover.

### Step 4: Snapshot and restore

```bash
aws rds create-db-snapshot --db-instance-identifier lab10-db --db-snapshot-identifier lab10-snap1
aws rds wait db-snapshot-available --db-snapshot-identifier lab10-snap1

aws rds restore-db-instance-from-db-snapshot --db-instance-identifier lab10-restored \
  --db-snapshot-identifier lab10-snap1 --db-instance-class db.t3.micro --no-multi-az
aws rds wait db-instance-available --db-instance-identifier lab10-restored
```

A restore always creates a new instance with a new endpoint; you cannot restore over the existing one. Point-in-time recovery (`restore-db-instance-to-point-in-time --restore-time` or `--use-latest-restorable-time`) works the same way, from automated backups within the retention period.

### Step 5: DynamoDB comparison

Model an access pattern like "get an order by id" in DynamoDB with no servers to size:

```bash
aws dynamodb create-table --table-name lab10-orders \
  --attribute-definitions AttributeName=order_id,AttributeType=S \
  --key-schema AttributeName=order_id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws dynamodb wait table-exists --table-name lab10-orders

aws dynamodb put-item --table-name lab10-orders \
  --item '{"order_id":{"S":"1001"},"total":{"N":"42"},"status":{"S":"paid"}}'
aws dynamodb get-item --table-name lab10-orders --key '{"order_id":{"S":"1001"}}'
```

Comparison notes to write down: a join across orders and customers is natural in RDS but requires denormalization or multiple queries in DynamoDB; DynamoDB scales to very high request rates with single-digit millisecond latency and is Multi-AZ by default; ad hoc analytical queries on DynamoDB need a scan or an export. For repeated reads of hot data, ElastiCache (Redis or Memcached) sits in front of either database as an in-memory cache, and DynamoDB has its own cache, DAX, for microsecond reads.

## Validation

- `describe-db-instances` shows `MultiAZ: true` and a different `AvailabilityZone` after the forced failover, with the same endpoint address.
- The replica has a distinct endpoint and rejects writes.
- The restored instance appears with its own endpoint.
- `get-item` returns the order from DynamoDB.

## Cleanup

Delete instances first, then snapshots, then parameter groups. Delete the replica before its source.

```bash
aws rds delete-db-instance --db-instance-identifier lab10-replica --skip-final-snapshot
aws rds delete-db-instance --db-instance-identifier lab10-restored --skip-final-snapshot
aws rds delete-db-instance --db-instance-identifier lab10-db --skip-final-snapshot --delete-automated-backups
aws rds wait db-instance-deleted --db-instance-identifier lab10-db
aws rds delete-db-snapshot --db-snapshot-identifier lab10-snap1
aws rds delete-db-parameter-group --db-parameter-group-name lab10-params
aws dynamodb delete-table --table-name lab10-orders
```

Then confirm in the RDS console that no instances, snapshots (manual snapshots are kept until deleted), or read replicas remain. If you opened port 3306 to your IP, remove that security group rule.

## Exam Takeaway

Multi-AZ is for availability and durability: synchronous standby in another AZ, automatic failover on the same endpoint, no read traffic served (except the Multi-AZ DB cluster deployment, which has readable standbys). Read replicas are for read scaling and cross-Region needs: asynchronous, separate endpoints, can be promoted, and do not give automatic failover on their own. Backups and snapshots restore to a new instance. Choose RDS for relational, transactional, SQL-join workloads, Aurora when you need higher scale and faster failover, DynamoDB for key-value access at massive scale with serverless operations, and ElastiCache when the requirement is caching or sub-millisecond reads in front of a database.

The main point is to identify which service reduces operational risk while still fitting the scenario's central requirement.
---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
