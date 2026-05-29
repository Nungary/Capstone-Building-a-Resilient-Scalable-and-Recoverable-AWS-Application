# Load Test Notes

This document describes the load tests executed to demonstrate scalability for:

- Web tier (EC2 Auto Scaling Group)
- Worker tier (ECS Fargate auto scaling via SQS depth)

## Web Tier Load Test (ASG scale-out / scale-in)

### Goal

Increase traffic to the primary ALB to trigger the ASG target tracking policy and observe:

- Desired capacity increases (scale-out)
- New instances launch and register to the target group
- After traffic stops, capacity returns to baseline (scale-in)

### Command approach

A simple loop was used to send sustained HTTP requests to the ALB.

Example approach (Git Bash): run a short sustained spike against the ALB DNS name.

Notes:

- The goal is to exceed the target tracking threshold long enough for CloudWatch to evaluate and the ASG to react.
- Collect evidence for both **scale-out** and **scale-in**.

### Expected behavior

- **Scale-out:** ASG desired capacity increases and new instances launch.
- **Registration:** new instances register into the target group and become `healthy`.
- **Scale-in:** after traffic stops and cooldown passes, ASG returns to baseline.

### Evidence to capture

- ASG Activity history showing scale-out and scale-in
- EC2 Instances list showing more instances during peak
- Target group Targets view showing additional healthy targets
- CloudWatch dashboard capacity widget

Recommended screenshots (minimum):

- ASG `capstone-web-asg` → **Activity** tab showing scale-out events
- EC2 Instances list showing additional instances during peak
- Target group `capstone-tg-web` → **Targets** showing new targets becoming healthy
- CloudWatch AlarmHigh for target tracking (if it entered ALARM)
- After the spike: ASG Activity showing scale-in / instance termination

### Results (fill in)

- **Start time:** ___
- **End time:** ___
- **Baseline capacity:** 2
- **Peak desired capacity:** ___
- **Time to first scale-out:** ___
- **Time to return to baseline:** ___

## Worker Tier Load Test (ECS scale-out / scale-in)

### Goal

Inject messages into `capstone-orders` SQS queue and observe:

- Queue depth rises
- ECS service desired/running task count increases
- After the queue is drained/purged, tasks scale back down

### Command approach

Send a batch (e.g., 200) SQS messages using AWS CLI.

Notes:

- The goal is to increase `ApproximateNumberOfMessagesVisible` so the ECS scaling policy increases desired tasks.
- After evidence is captured, the queue can be drained (if you have a real consumer) or purged to show scale-in.

### Expected behavior

- **Queue depth rises** shortly after injection.
- **ECS desired tasks increases** (and tasks start/pending/running increases).
- **Scale-in** occurs after queue depth returns to low/zero and cooldown passes.

### Evidence to capture

- ECS Tasks tab showing many tasks created
- ECS service events / deployments showing scaling
- SQS Monitoring tab showing message spike
- CloudWatch dashboard SQS + ECS widgets

Recommended screenshots (minimum):

- SQS queue `capstone-orders` → **Monitoring** showing message spike
- ECS cluster `capstone-cluster` → service `capstone-worker` → **Events** showing scaling
- ECS Tasks tab showing many tasks created (scale-out)
- CloudWatch dashboard widgets for SQS depth and ECS utilization
- After drain/purge: ECS tasks reduced back toward baseline (scale-in)

### Results (fill in)

- **Messages injected:** ___ (e.g., 200)
- **Baseline ECS desired tasks:** 1
- **Peak ECS desired tasks:** ___
- **Time to scale-out:** ___
- **Time to scale-in:** ___
