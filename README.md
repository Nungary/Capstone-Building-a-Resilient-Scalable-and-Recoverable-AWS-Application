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

## Architecture 
<img width="1081" height="761" alt="Untitled Diagram drawio" src="https://github.com/user-attachments/assets/c70b6b27-b79b-4649-a64c-940ad092a6d3" />



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

<img width="1919" height="1022" alt="Screenshot 2026-05-29 095933" src="https://github.com/user-attachments/assets/431a25d0-1c4b-4d68-8fb6-ab346ff67921" />

- **DR ALB (eu-central-1):** `http://capstone-alb-dr-1933974459.eu-central-1.elb.amazonaws.com`
  
<img width="1919" height="1024" alt="Screenshot 2026-05-29 111002" src="https://github.com/user-attachments/assets/d7adcdcc-60af-417e-b186-f079b0e94f2f" />


### High Availability 

- Default VPC subnets across 2+ AZs (primary region)
<img width="1412" height="873" alt="Screenshot 2026-05-29 093223" src="https://github.com/user-attachments/assets/8319644f-f3c6-4148-81dd-1d14fe0123c7" />


- Security groups created (ALB / EC2 / RDS)
 <img width="1919" height="957" alt="Screenshot 2026-05-29 094205" src="https://github.com/user-attachments/assets/4dd36031-8d68-4a8c-8ee7-5b7379304ad8" />


- ALB created and healthy
  
  <img width="1919" height="960" alt="Screenshot 2026-05-29 094750" src="https://github.com/user-attachments/assets/931c3070-4490-4139-b034-70c8b808e22f" />


- ASG created with 2 instances across 2 AZs
  <img width="1919" height="960" alt="Screenshot 2026-05-29 095505" src="https://github.com/user-attachments/assets/a6147c3d-b709-4a95-9ded-8ad82fff4cbb" />

- Target group shows healthy targets
  <img width="1919" height="960" alt="Screenshot 2026-05-29 100240" src="https://github.com/user-attachments/assets/0ddb94ab-c3a1-4504-a985-93a0db81f61d" />

- Browser proof: ALB serves the nginx page including Instance ID + AZ
 <img width="1919" height="1022" alt="Screenshot 2026-05-29 095933" src="https://github.com/user-attachments/assets/35eddec6-d2fb-48ac-be0b-9a17df8e47fc" />

 <img width="1919" height="1025" alt="Screenshot 2026-05-29 100035" src="https://github.com/user-attachments/assets/9efdee9a-5bf3-44f0-8f5f-05eec8e95cef" />

 <img width="1654" height="1079" alt="Screenshot 2026-05-29 095736" src="https://github.com/user-attachments/assets/463e0847-af68-4f11-ba8d-69f1f112aaf6" />
 

- RDS MySQL created with **Multi-AZ = Yes**
- RDS Multi-AZ failover behavior documented (see `docs/dr/dr-runbook.md`)
  
  <img width="1919" height="955" alt="Screenshot 2026-05-29 102228" src="https://github.com/user-attachments/assets/f0e6262a-2946-4be9-b7c4-7832997c72cf" />

- DynamoDB Global Table created (`eu-west-1` + replica in `eu-central-1`)

  <img width="1919" height="956" alt="Screenshot 2026-05-29 101301" src="https://github.com/user-attachments/assets/549eaf57-c9ec-4ecf-8038-b4bfc6eb09d8" />

<img width="1919" height="966" alt="Screenshot 2026-05-29 101519" src="https://github.com/user-attachments/assets/6907de9e-28c9-4ba0-acf1-f85081f071ae" />


### Scalability 

- ASG target tracking policy created (`capstone-web-scaling-policy`)
  <img width="1919" height="834" alt="Screenshot 2026-05-29 102555" src="https://github.com/user-attachments/assets/6b91670c-d7e6-417f-9247-f78fa8eec39e" />

- Auto-created CloudWatch alarms (TargetTracking AlarmHigh/AlarmLow)
  
  <img width="1919" height="832" alt="Screenshot 2026-05-29 102650" src="https://github.com/user-attachments/assets/fc25fb3e-99ad-4a61-abf5-47c6b0993e66" />

- Load test evidence:
  - ASG activity history showing scale-out events (DesiredCapacity increases)
    
    <img width="1919" height="950" alt="Screenshot 2026-05-29 112152" src="https://github.com/user-attachments/assets/6c99353f-94e0-4b64-a48d-4568c01ae5cc" />

  - EC2 instances list showing scale-out (2 → N)
    
    <img width="1919" height="954" alt="Screenshot 2026-05-29 112406" src="https://github.com/user-attachments/assets/ee23066a-dcd7-4a73-90e5-40ea8724a26a" />

- Redis replication group created + status available
  
  <img width="1919" height="961" alt="Screenshot 2026-05-29 103830" src="https://github.com/user-attachments/assets/9622e797-3358-4617-a3b6-60f9baaab5b0" />

- SQS main queue + DLQ created
 
  <img width="1919" height="956" alt="Screenshot 2026-05-29 103715" src="https://github.com/user-attachments/assets/cd12527e-819c-4576-aa02-2ba7d4f87eb0" />

- ECS service `capstone-worker` created
  <img width="1919" height="954" alt="Screenshot 2026-05-29 103806" src="https://github.com/user-attachments/assets/ed0cc467-3066-432f-8c76-ea28d6fb812f" />

- Worker scale-out evidence:
  - ECS Tasks view showing many tasks launched
    <img width="1919" height="956" alt="Screenshot 2026-05-29 113509" src="https://github.com/user-attachments/assets/8768e3bb-b301-4f5f-8100-91f79abfce34" />

#### What is cached (ElastiCache) and why

ElastiCache Redis is used to cache hot / frequently-read data (e.g., session lookups or repeated reads) to reduce repeated database calls to RDS. This improves latency during traffic spikes and protects the database tier from overload.

#### Load test summary 

observations

- **Web tier start:** ASG desired capacity = 2
- **Web tier peak:** ASG desired capacity reached: 4
- **Scale-out trigger:** Target tracking policy on `ALBRequestCountPerTarget`
- **Scale-in behavior:** Returned to 2 instances after load stopped in ~ about 10 minutes

- **Worker tier start:** ECS desired tasks = 1
- **Worker tier peak:** ECS desired tasks reached: 60
- **Scale-out trigger:** SQS `ApproximateNumberOfMessagesVisible` target tracking
- **Scale-in behavior:** Returned to 1 task after queue drained/purged in about 10 minutes

![10-asg-activity-scale-out]
<img width="1908" height="960" alt="image" src="https://github.com/user-attachments/assets/d49c2e0c-3e2e-4b0a-9dae-ac1e22056f61" />

![11-ec2-scale-out]
<img width="1910" height="945" alt="image" src="https://github.com/user-attachments/assets/40e85099-5c07-42aa-b52d-2493a5b6d105" />

![13-redis-replication-group]

<img width="1914" height="850" alt="image" src="https://github.com/user-attachments/assets/14ee1658-71a3-4a9c-b805-5136b8ef083c" />

![14-sqs-queues]

<img width="1911" height="956" alt="image" src="https://github.com/user-attachments/assets/a099dd52-036b-4519-a3e2-7c72d5958fc1" />

![16-ecs-tasks-scale-out])
<img width="1907" height="957" alt="image" src="https://github.com/user-attachments/assets/6efd4d18-9a37-4c8c-8296-72d732d9b493" />


### Disaster Recovery 

#### RTO / RPO

- **RTO (Recovery Time Objective):** 30 minutes
- **RPO (Recovery Point Objective):** 5 minutes

Mapping to services:

- **RTO support:** Warm standby in `eu-central-1` (DR ALB + DR ASG) + Route 53 failover routing.
- **RPO support:** DynamoDB Global Tables (near-real-time replication) + RDS cross-region read replica lag monitored.

- DR default VPC + DR security groups
  <img width="1912" height="955" alt="image" src="https://github.com/user-attachments/assets/fe2864a7-2f5a-47bb-9216-c626cc3b010e" />

- DR ALB created and serving
  <img width="1915" height="959" alt="image" src="https://github.com/user-attachments/assets/425b1eb5-8985-4678-9d2b-ed649a4eb913" />

- DR ASG warm standby (min/desired = 1)
  <img width="1907" height="958" alt="image" src="https://github.com/user-attachments/assets/3d714ef2-ec39-4618-9c2a-7999821140bb" />

- DR browser proof: DR ALB serves the nginx page including Instance ID + AZ
  <img width="1919" height="1024" alt="Screenshot 2026-05-29 111002" src="https://github.com/user-attachments/assets/c039ce86-99e1-4360-9522-cc6da1453abf" />

- RDS cross-region read replica creation + status (eventually `available`)
  <img width="1908" height="954" alt="image" src="https://github.com/user-attachments/assets/de51d735-69b8-41b4-b3ff-19d0a269159c" />

- Route 53 failover records:
  - `app.capstone.local` PRIMARY → primary ALB (with health check)
    <img width="1906" height="848" alt="image" src="https://github.com/user-attachments/assets/455c139c-05f2-49f8-a959-60cfe84e213c" />

  - `app.capstone.local` SECONDARY → DR ALB
    <img width="1914" height="952" alt="image" src="https://github.com/user-attachments/assets/da4476a8-49e6-4880-9393-cd140bc57d2c" />


#### Failover simulation evidence

During a failover drill, the Route 53 primary health check was forced to fail by changing the health check path to `/does-not-exist`) to trigger automatic DNS failover to the DR ALB.



- **Failover started at
  <img width="1919" height="1022" alt="Screenshot 2026-05-29 140502" src="https://github.com/user-attachments/assets/10247c6d-ffac-4de6-befc-a8e5b2ad757e" />
- **DR serving confirmed
  <img width="1919" height="1079" alt="Screenshot 2026-05-29 141711" src="https://github.com/user-attachments/assets/1161bc4c-7e8e-42c9-9281-c7c0e9199505" />
 
- **Observed downtime:**
  <img width="1919" height="1078" alt="Screenshot 2026-05-29 141358" src="https://github.com/user-attachments/assets/5cdfb550-7fb0-42a6-84c2-24b3e295c56e" />


### Observability 

- CloudWatch custom alarms created:
  - `capstone-alb-5xx-errors`
  - `capstone-alb-unhealthy-hosts`
  - `capstone-rds-high-cpu`
  - `capstone-sqs-message-age`
    <img width="1919" height="961" alt="Screenshot 2026-05-29 104959" src="https://github.com/user-attachments/assets/6d5c8c66-3109-45bc-9ea7-ff56ca0b0c86" />

- CloudWatch dashboard created: `capstone-infrastructure`

  <img width="1919" height="957" alt="Screenshot 2026-05-29 105156" src="https://github.com/user-attachments/assets/f5b25f16-fc91-4513-ad2f-79bbce3d3dd0" />


#### Logs (for troubleshooting)

- **ECS worker logs:** CloudWatch Logs log group `/ecs/capstone-worker` (awslogs driver).
- **EC2 web tier logs:** Nginx access/error logs available on instances (`/var/log/nginx/`).

## Backup (AWS Backup)

Backup policies were configured using AWS Backup for:

- **RDS** snapshots (schedule + retention)
- **DynamoDB** backups (schedule + retention)

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

