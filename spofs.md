# SPOF Analysis (Single Points of Failure)

This document summarizes potential single points of failure and how they were mitigated for this capstone.

## Network / Entry

- **ALB (primary)**
  - **Risk:** Single load balancer or single AZ attachment.
  - **Mitigation:** ALB spans multiple subnets in different AZs.

- **Route 53 failover**
  - **Risk:** Single region outage.
  - **Mitigation:** Route 53 failover records route to DR ALB if primary health check fails.

## Web Tier

- **EC2 instance failure**
  - **Risk:** Single EC2 instance down causes outage.
  - **Mitigation:** Auto Scaling Group with multiple instances across AZs + ELB health checks.

- **AZ failure**
  - **Risk:** Losing one AZ removes capacity.
  - **Mitigation:** ASG spans at least two AZs. ALB routes only to healthy targets.

## Data Tier

- **RDS (primary)**
  - **Risk:** Primary database host failure.
  - **Mitigation:** RDS Multi-AZ deployment with synchronous standby and automatic failover.

- **Regional outage**
  - **Risk:** Primary RDS region down.
  - **Mitigation:** Cross-region read replica in DR region (warm standby pattern).

## State

- **Session/state storage**
  - **Risk:** Single-region state store unavailable.
  - **Mitigation:** DynamoDB Global Table with replica in DR region.

## Cache

- **Redis primary node failure**
  - **Risk:** Cache unavailable.
  - **Mitigation:** ElastiCache replication group (primary + replica) with automatic failover.

## Async / Worker

- **Worker compute failure**
  - **Risk:** Single worker down stalls processing.
  - **Mitigation:** ECS Fargate service with Application Auto Scaling.

- **Backpressure and poison messages**
  - **Risk:** Unprocessable messages block queue.
  - **Mitigation:** SQS Dead Letter Queue configured with redrive policy.

## Observability

- **Undetected failures**
  - **Risk:** Outage without alerting.
  - **Mitigation:** CloudWatch dashboard + alarms (ALB 5xx, unhealthy hosts, RDS CPU, SQS age).
