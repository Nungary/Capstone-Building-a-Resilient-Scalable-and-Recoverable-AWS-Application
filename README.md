# Capstone Week 5 — Resilient, Scalable & Recoverable AWS Architecture

> **Primary Region:** `eu-west-1` (Ireland)  
> **DR Region:** `eu-central-1` (Frankfurt)  
> **Networking:** Built on the **default VPC** in both regions.

## Summary

This project implements a resilient AWS architecture that:

- Survives **AZ failures** via **Multi-AZ** design (ALB + ASG across 2 AZs).
- Scales automatically under load using **target tracking Auto Scaling**.
- Supports **regional disaster recovery** using a **warm-standby** pattern in a secondary region.
- Provides **observability** using **CloudWatch dashboard + alarms**.

## Architecture (Logical)

![Architecture Diagram](docs/architecture-diagram.png)

| Tier | Services | Resilience / Scaling mechanism |
| --- | --- | --- |
| Edge | Application Load Balancer (primary + DR) | Multi-AZ ALBs, Route 53 failover routing |
| Web | EC2 Auto Scaling Group (nginx) | Multi-AZ (2 AZs), target tracking scale-out / scale-in |
| Relational Data | RDS MySQL (primary Multi-AZ) + DR read replica | Multi-AZ failover (primary), cross-region replica (DR) |
| Cache | ElastiCache Redis replication group | Replica + automatic failover |
| Async / Worker | SQS + ECS Fargate service | Application Auto Scaling based on SQS depth |
| State | DynamoDB Global Table | Cross-region sync `eu-west-1` ↔ `eu-central-1` |
| Observability | CloudWatch dashboard + alarms | Central monitoring + alerting |

## Live Endpoints

- **Primary ALB (eu-west-1):** `http://capstone-alb-1119147291.eu-west-1.elb.amazonaws.com`
- **DR ALB (eu-central-1):** `http://capstone-alb-dr-1933974459.eu-central-1.elb.amazonaws.com`
- **RDS Endpoint (Primary):** `capstone-mysql-primary.cb2wywyq61to.eu-west-1.rds.amazonaws.com`

## Evidence (Screenshots)

Place your screenshots under `docs/screenshots/` and reference them here.

### Screenshot naming convention

Save your evidence images into `docs/screenshots/` using these filenames (or update the links below to match your names):

- `01-vpc-subnets-primary.png`
- `02-security-groups-primary.png`
- `03-alb-primary.png`
- `04-target-group-primary-healthy.png`
- `05-asg-instances-primary.png`
- `06-browser-primary-alb.png`
- `07-rds-primary-multi-az.png`
- `08-dynamodb-global-table.png`
- `09-asg-scaling-policy.png`
- `10-asg-activity-scale-out.png`
- `11-ec2-scale-out.png`
- `12-cloudwatch-alarmhigh.png`
- `13-redis-replication-group.png`
- `14-sqs-queues.png`
- `15-ecs-service.png`
- `16-ecs-tasks-scale-out.png`
- `17-dr-vpc-subnets.png`
- `18-dr-alb.png`
- `19-browser-dr-alb.png`
- `20-rds-dr-replica.png`
- `21-route53-failover-records.png`
- `22-cloudwatch-dashboard.png`
- `23-cloudwatch-alarms.png`

### High Availability (Day 1)

- Default VPC subnets across 2+ AZs (primary region)
- Security groups created (ALB / EC2 / RDS)
- ALB created and healthy
- ASG created with 2 instances across 2 AZs
- Target group shows healthy targets
- Browser proof: ALB serves the nginx page including Instance ID + AZ
- RDS MySQL created with **Multi-AZ = Yes**
- RDS Multi-AZ failover behavior documented (see `docs/dr/dr-runbook.md`)
- DynamoDB Global Table created (`eu-west-1` + replica in `eu-central-1`)

![01-vpc-subnets-primary](docs/screenshots/01-vpc-subnets-primary.png)

![02-security-groups-primary](docs/screenshots/02-security-groups-primary.png)

![03-alb-primary](docs/screenshots/03-alb-primary.png)

![04-target-group-primary-healthy](docs/screenshots/04-target-group-primary-healthy.png)

![05-asg-instances-primary](docs/screenshots/05-asg-instances-primary.png)

![06-browser-primary-alb](docs/screenshots/06-browser-primary-alb.png)

![07-rds-primary-multi-az](docs/screenshots/07-rds-primary-multi-az.png)

![08-dynamodb-global-table](docs/screenshots/08-dynamodb-global-table.png)

### Scalability (Day 2)

- ASG target tracking policy created (`capstone-web-scaling-policy`)
- Auto-created CloudWatch alarms (TargetTracking AlarmHigh/AlarmLow)
- Load test evidence:
  - ASG activity history showing scale-out events (DesiredCapacity increases)
  - EC2 instances list showing scale-out (2 → N)
  - Optional: target group registering additional instances
- Redis replication group created + status available
- SQS main queue + DLQ created
- ECS service `capstone-worker` created
- Worker scale-out evidence:
  - ECS Tasks view showing many tasks launched
  - CloudWatch metrics for SQS depth and ECS service utilization

#### What is cached (ElastiCache) and why

ElastiCache Redis is used to cache hot / frequently-read data (e.g., session lookups or repeated reads) to reduce repeated database calls to RDS. This improves latency during traffic spikes and protects the database tier from overload.

#### Load test summary (short)

Fill this in using your observations/screenshots:

- **Web tier start:** ASG desired capacity = 2
- **Web tier peak:** ASG desired capacity reached: ___ (e.g., 4 or 6)
- **Scale-out trigger:** Target tracking policy on `ALBRequestCountPerTarget`
- **Scale-in behavior:** Returned to 2 instances after load stopped in ~___ minutes

- **Worker tier start:** ECS desired tasks = 1
- **Worker tier peak:** ECS desired tasks reached: ___
- **Scale-out trigger:** SQS `ApproximateNumberOfMessagesVisible` target tracking
- **Scale-in behavior:** Returned to 1 task after queue drained/purged in ~___ minutes

![09-asg-scaling-policy](docs/screenshots/09-asg-scaling-policy.png)

![10-asg-activity-scale-out](docs/screenshots/10-asg-activity-scale-out.png)

![11-ec2-scale-out](docs/screenshots/11-ec2-scale-out.png)

![12-cloudwatch-alarmhigh](docs/screenshots/12-cloudwatch-alarmhigh.png)

![13-redis-replication-group](docs/screenshots/13-redis-replication-group.png)

![14-sqs-queues](docs/screenshots/14-sqs-queues.png)

![15-ecs-service](docs/screenshots/15-ecs-service.png)

![16-ecs-tasks-scale-out](docs/screenshots/16-ecs-tasks-scale-out.png)

### Disaster Recovery (Day 3)

#### RTO / RPO

- **RTO (Recovery Time Objective):** 30 minutes
- **RPO (Recovery Point Objective):** 5 minutes

Mapping to services:

- **RTO support:** Warm standby in `eu-central-1` (DR ALB + DR ASG) + Route 53 failover routing.
- **RPO support:** DynamoDB Global Tables (near-real-time replication) + RDS cross-region read replica lag monitored.

- DR default VPC + DR security groups
- DR ALB created and serving
- DR ASG warm standby (min/desired = 1)
- DR browser proof: DR ALB serves the nginx page including Instance ID + AZ
- RDS cross-region read replica creation + status (eventually `available`)
- Route 53 failover records:
  - `app.capstone.local` PRIMARY → primary ALB (with health check)
  - `app.capstone.local` SECONDARY → DR ALB

#### Failover simulation evidence

During a failover drill, the Route 53 primary health check was forced to fail (for example, by changing the health check path to `/does-not-exist`) to trigger automatic DNS failover to the DR ALB.

Fill this in using your test:

- **Failover started at (timestamp):** ___
- **DR serving confirmed at (timestamp):** ___
- **Observed downtime:** ___ (minutes/seconds)
- **Observed data loss:** ___ (should be within RPO)

![17-dr-vpc-subnets](docs/screenshots/17-dr-vpc-subnets.png)

![18-dr-alb](docs/screenshots/18-dr-alb.png)

![19-browser-dr-alb](docs/screenshots/19-browser-dr-alb.png)

![20-rds-dr-replica](docs/screenshots/20-rds-dr-replica.png)

![21-route53-failover-records](docs/screenshots/21-route53-failover-records.png)

### Observability (Day 4)

- CloudWatch custom alarms created:
  - `capstone-alb-5xx-errors`
  - `capstone-alb-unhealthy-hosts`
  - `capstone-rds-high-cpu`
  - `capstone-sqs-message-age`
- CloudWatch dashboard created: `capstone-infrastructure`

![22-cloudwatch-dashboard](docs/screenshots/22-cloudwatch-dashboard.png)

![23-cloudwatch-alarms](docs/screenshots/23-cloudwatch-alarms.png)

#### Logs (for troubleshooting)

- **ECS worker logs:** CloudWatch Logs log group `/ecs/capstone-worker` (awslogs driver).
- **EC2 web tier logs:** Nginx access/error logs available on instances (`/var/log/nginx/`).

## Backup (AWS Backup)

Backup policies were configured using AWS Backup for:

- **RDS** snapshots (schedule + retention)
- **DynamoDB** backups (schedule + retention)

Add screenshots under `docs/screenshots/`:

- `24-aws-backup-vaults.png`
- `25-aws-backup-plan.png`
- `26-aws-backup-jobs.png`

## Repository Layout

```
.
├── README.md
├── lab.env
├── spofs.md
└── docs/
    ├── dr/
    │   └── dr-runbook.md
    ├── load-test/
    │   └── load-test-notes.md
    ├── observability/
    │   └── dashboard-definition.json
    └── screenshots/
        └── (put your evidence images here)
```

## How to Reproduce (CLI Summary)

A concise command log and resource IDs are captured in `lab.env`. The build followed these phases:

- Networking selection (default VPC + 2 subnets in different AZs)
- Security groups (ALB/EC2/RDS/Redis/Fargate)
- ALB + target group + listener
- Launch template + ASG
- RDS Multi-AZ
- DynamoDB Global Table
- Web tier scaling policy
- ElastiCache Redis
- SQS (main + DLQ)
- ECS cluster + IAM roles + Fargate task definition + service + scaling
- DR region: ALB + warm standby ASG + RDS replica
- Route 53 failover record + health check
- CloudWatch alarms + dashboard

## Cleanup

Deleted resources in reverse order to minimize dependency issues:

- ECS service/cluster, SQS queues
- ElastiCache replication group
- ASG + launch template, ALB + target group
- RDS instances (replica then primary)
- DynamoDB table (and replicas)
- Route 53 records + health check + hosted zone
- Security groups (after dependencies removed)

## Trade-offs & Improvements (Reflection)

### Trade-offs made

- **Cost vs. resilience:** Warm standby in DR uses a smaller footprint (1 instance) to reduce cost.
- **Simplicity vs. realism:** The web tier serves a simple nginx page; in a production app, caching and worker processing would be application-driven.
- **DNS failover behavior:** Route 53 failover depends on health check evaluation intervals and DNS caching.

### Potential improvements

- Add HTTPS (ACM + ALB HTTPS listener) and enforce TLS end-to-end.
- Centralize EC2 web logs into CloudWatch Logs (agent) for easier troubleshooting.
- Add a real SQS consumer implementation to demonstrate message processing (not only scaling).
- Automate provisioning with IaC (Terraform/CloudFormation) for repeatability.

