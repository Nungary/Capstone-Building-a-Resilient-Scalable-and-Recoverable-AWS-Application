# DR Runbook (Warm Standby)

> **Primary:** `eu-west-1`  
> **DR:** `eu-central-1`

## Objectives (RTO / RPO)

- **RTO (Recovery Time Objective):** 30 minutes
- **RPO (Recovery Point Objective):** 5 minutes

How the design supports these:

- **RTO support:** Warm standby compute (DR ALB + DR ASG running) + Route 53 failover routing.
- **RPO support:** DynamoDB Global Tables replication + RDS cross-region read replica (replication lag monitored).

## Goal

Demonstrate a warm-standby Disaster Recovery pattern where:

- Primary region handles production traffic.
- DR region stays online with reduced capacity.
- Route 53 failover moves traffic to DR when primary is unhealthy.

## Preconditions

- Primary stack deployed:
  - ALB + target group + listener
  - ASG (min/desired 2)
  - RDS Multi-AZ
  - DynamoDB Global Table
- DR stack deployed:
  - DR ALB + target group + listener
  - DR ASG warm standby (min/desired 1)
  - RDS cross-region read replica creating/available
- Route 53 hosted zone exists
- Failover record created:
  - `app.capstone.local` PRIMARY → primary ALB (with health check)
  - `app.capstone.local` SECONDARY → DR ALB

## Failover Drill (DNS-based)

### 1) Confirm steady state

- Primary ALB serves app over HTTP.
- Route 53 health check is `Healthy`.

Evidence to capture:

- Route 53 health check status = `Healthy`
- `app.capstone.local` resolves to primary
- Primary ALB page loads and shows `eu-west-1*` AZ

### 2) Simulate primary failure

Options:

- **Health check path override** (recommended)
  - Temporarily change the health check ResourcePath to a non-existent endpoint (e.g. `/does-not-exist`).
  - Wait for Route 53 health check to become `Unhealthy`.

- **Application failure simulation**
  - Stop nginx on all primary instances (not recommended unless required).

### 3) Observe DNS failover

- Use `nslookup app.capstone.local` or resolve from a browser.
- Validate it resolves to DR ALB and DR page loads.

Evidence to capture:

- Route 53 health check status = `Unhealthy`
- `app.capstone.local` resolves to DR
- DR ALB page loads and shows `eu-central-1*` AZ

### 4) Recovery

- Restore the health check path to `/`.
- Wait for health check to turn `Healthy`.
- Confirm traffic returns to primary.

Evidence to capture:

- Health check returns to `Healthy`
- DNS resolves back to primary
- Primary page loads again

## Failover drill steps (AWS Console)

### A) Force the health check to fail

1. Go to **Route 53** → **Health checks**.
2. Open the primary health check created for the primary ALB.
3. Click **Edit**.
4. Change **Path** from `/` to `/does-not-exist`.
5. Save changes.
6. Wait until the health check status becomes **Unhealthy**.

### B) Validate failover

1. Go to **Route 53** → **Hosted zones** → your zone (`capstone.local`).
2. Open the record `app.capstone.local`.
3. Confirm PRIMARY/SECONDARY records exist.
4. In your browser, open `http://app.capstone.local`.
5. Confirm the page is served by the **DR ALB** (AZ should show `eu-central-1*`).

### C) Recover / failback

1. Edit the health check and set **Path** back to `/`.
2. Wait for **Healthy**.
3. Confirm `http://app.capstone.local` returns to primary.

## Optional: Failover drill checks (CLI)

Use these to capture timestamped CLI evidence:

- Resolve DNS:
  - `nslookup app.capstone.local`
- Confirm current Route 53 records:
  - `aws route53 list-resource-record-sets --hosted-zone-id <ZONE_ID> --query 'ResourceRecordSets[?Name==\`app.capstone.local.\`]' --output table`

## Record of results (fill in)

| Item | Value |
| --- | --- |
| Failover start time (when health check edited) | ___ |
| Health check became Unhealthy | ___ |
| First successful DR page load | ___ |
| Observed downtime | ___ |
| Replica lag (RDS) at failover time (seconds) | ___ |
| Observed data loss | ___ |
| Failback start time | ___ |
| First successful primary page load after failback | ___ |

## Screenshot checklist

- Health check details showing `Healthy` (before)
- Health check details showing `Unhealthy` (during)
- Hosted zone record set showing PRIMARY/SECONDARY
- Browser screenshot loading `app.capstone.local` during failover (DR)
- Browser screenshot loading `app.capstone.local` after recovery (primary)

## Promoting DR RDS (If required)

If the assignment requires DR to accept writes (vs read replica), promote the replica:

- Promote the DR read replica to a standalone DB instance.
- Update application configuration to point to the new writer endpoint.

## Failback

- Ensure primary region is restored.
- Re-establish replication (create a new cross-region replica from the recovered writer).
- Switch Route 53 back to primary.

## Notes / Limitations

- Warm standby keeps DR capacity low (cost optimized) and scales up during an event.
- DNS failover depends on TTL and Route 53 health check evaluation.
