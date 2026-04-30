# Lab M2.06 - CloudWatch Monitoring and Alarms
## Lab Report / Solution

**Student:** Eric Borba  
**Instance:** i-093f98279195ce0d9 (ip-172-31-35-32)  
**Region:** eu-central-1  
**Date:** 2026-04-30  

---

## Step-by-Step Process

### 1. Create the SNS Topic and Email Subscription

The first step was to set up the notification channel so alarms have somewhere to send alerts.

```bash
# Create the SNS topic
aws sns create-topic --name ec2-alerts

# Subscribe an email endpoint
aws sns subscribe \
  --topic-arn arn:aws:sns:eu-central-1:871939031886:ec2-alerts \
  --protocol email \
  --notification-endpoint eric.borba@gmail.com
```

After running the subscribe command, AWS sends a confirmation email. The subscription is only active after clicking the confirmation link. The resulting topic and confirmed subscription details were:

- **Topic ARN:** `arn:aws:sns:eu-central-1:871939031886:ec2-alerts`
- **Subscription ARN:** `arn:aws:sns:eu-central-1:871939031886:ec2-alerts:6a847857-4712-4a05-a2cf-b2986c473f8f`
- **Status:** Confirmed

---

### 2. Create the CPU Utilization Alarm (high-cpu-alarm)

With the notification channel in place, the CloudWatch alarm was created via the AWS CLI:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu-alarm \
  --alarm-description "Alert when CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-093f98279195ce0d9 \
  --alarm-actions arn:aws:sns:eu-central-1:871939031886:ec2-alerts
```

This alarm monitors the native EC2 `CPUUtilization` metric and triggers if the average exceeds 80% for two consecutive 5-minute evaluation periods.

---

### 3. Create the Anomaly Detection Alarm (agent-cpu-above85)

A second, more sophisticated alarm was configured using CloudWatch Anomaly Detection. This alarm uses a machine-learning model trained on the `cpu_usage_user` metric collected by the CloudWatch Agent to establish an expected upper bound, rather than a fixed threshold.

The CloudFormation resource was deployed as documented in `metrics-query.txt`. Key parameters:

- **Metric:** `CWAgent/cpu_usage_user` on host `ip-172-31-35-32`, dimension `cpu-total`
- **Expression:** `ANOMALY_DETECTION_BAND(m1, 85)` — sensitivity value of 85
- **Trigger:** CPU exceeds the upper boundary of the anomaly detection band

---

### 4. Install the CloudWatch Agent and Collect Custom Metrics

To enable the `CWAgent` namespace metrics, the CloudWatch Agent was installed on the instance:

```bash
# Install CloudWatch Agent
sudo apt-get install -y amazon-cloudwatch-agent

# Configure and start the agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
```

This allowed the collection of per-CPU metrics (`cpu_usage_user`) not available in the default EC2 namespace.

---

### 5. Stress Test to Trigger the Alarm

To validate the full monitoring pipeline, a CPU stress test was executed on the instance:

```bash
sudo apt-get install -y stress
stress --cpu 2 --timeout 600s
```

Output:
```
stress: info: [1113] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
stress: info: [1113] successful run completed in 600s
```

The two CPU worker processes saturated the instance's CPU, pushing utilization to ~100% for the full 10-minute duration.

---

### 6. Verify Alarm State and Email Receipt

After approximately 21 minutes (two 5-minute evaluation periods plus CloudWatch processing delay), the alarm transitioned to ALARM state at **07:41:26 UTC**. An email notification was successfully received at `eric.borba@gmail.com`.

```bash
# Verify alarm status
aws cloudwatch describe-alarms --alarm-names high-cpu-alarm
```

---

## Why Monitoring Is Important

Monitoring is a foundational practice in cloud operations for several reasons:

**Visibility:** Cloud infrastructure is dynamic and distributed. Without monitoring, there is no way to know whether a service is degraded, overloaded, or completely down. CloudWatch provides that visibility in real time.

**Proactive incident response:** Alarms like `high-cpu-alarm` allow teams to detect problems before they cause customer-facing outages. If CPU is at 100% for 10 minutes, the application is likely unresponsive or severely degraded — an alert enables intervention before users are impacted.

**Capacity planning:** Historical metrics data enables informed decisions about instance sizing, auto-scaling policies, and infrastructure investment.

**Cost control:** Monitoring can reveal idle or underutilized resources, helping avoid unnecessary spending.

**Security and compliance:** Unexpected spikes in CPU, network traffic, or disk I/O can indicate security incidents such as cryptomining, DDoS participation, or compromised workloads.

In short, you cannot manage what you cannot measure.

---

## Explanation of Alarm Thresholds

### high-cpu-alarm — 80% threshold, 2 evaluation periods

**Why 80%?**  
80% was chosen as a practical warning threshold that gives enough headroom before the instance becomes critically overloaded (100% CPU), while still being high enough to avoid false alarms during normal application bursts. A threshold of 50% on a general-purpose instance would generate too much noise; 95% would leave almost no time to react before impact.

**Why 2 evaluation periods (10 minutes)?**  
A single 5-minute spike could be a legitimate but transient event — for example, a background cron job, a short batch process, or a deployment. Requiring two consecutive periods above the threshold significantly reduces false positives and ensures the alert represents a sustained condition that warrants attention.

**Why 5-minute periods (Period=300)?**  
CloudWatch publishes EC2 metrics at 1-minute or 5-minute granularity depending on the monitoring plan (5-minutes for the free tier). Five minutes provides a reasonable balance between alert latency and metric noise reduction.

### agent-cpu-above85 — Anomaly Detection, sensitivity 85

The anomaly detection alarm uses a statistical model of historical CPU behavior rather than a fixed cutoff. The sensitivity value of 85 controls how wide the "expected" band is. This alarm is particularly useful for workloads with variable, time-of-day-dependent patterns (e.g., scheduled batch jobs), where a fixed threshold would either miss real anomalies or generate constant false positives.

---

## How SNS Notifications Work

Amazon Simple Notification Service (SNS) is a fully managed pub/sub messaging service. In this lab it serves as the "delivery pipe" between CloudWatch and the email inbox.

**Flow:**

```
EC2 CPU metric exceeds threshold
        ↓
CloudWatch Alarm transitions to ALARM state
        ↓
CloudWatch calls the alarm's AlarmActions
        ↓
Publishes a message to SNS Topic: ec2-alerts
        ↓
SNS fans out the message to all confirmed subscriptions
        ↓
Email delivered to eric.borba@gmail.com
```

**Key concepts:**

- **Topic:** A logical channel that decouples publishers (CloudWatch) from subscribers (email, Lambda, SQS, HTTP endpoints, etc.).
- **Subscription:** An endpoint registered to a topic. Subscriptions must be confirmed by the endpoint owner to prevent spam.
- **Fan-out:** A single SNS publish can simultaneously deliver to multiple subscribers — for example, email AND a Slack webhook AND a PagerDuty integration.
- **AlarmActions:** The list of SNS Topic ARNs that CloudWatch calls when the alarm state changes.

This architecture means the alarm configuration does not need to know about individual recipients — it only knows about the topic ARN. Recipients can be added or removed from the topic independently.

---

## Metrics Analysis: What Was Observed During the Stress Test?

**Before the stress test (~07:20 UTC):**  
CPU utilization was at baseline (near 0%), consistent with an idle EC2 instance with no active workloads.

**During the stress test (07:20 – 07:30 UTC):**  
CPU utilization rose sharply and immediately to ~100% as both CPU worker processes spun up. The `stress` tool spawned tight infinite loops that fully occupied both virtual CPU cores.

**CloudWatch evaluation (07:31 – 07:41 UTC):**  
CloudWatch recorded two consecutive 5-minute average datapoints both exceeding the 80% threshold:
- **07:31:00 UTC:** 99.9953% average
- **07:36:00 UTC:** 100.0% average

Both datapoints were well above the 80% threshold, confirming that `stress` fully saturated the instance.

**Alarm transition (07:41:26 UTC):**  
After two evaluation periods, CloudWatch transitioned the alarm from OK to ALARM and published to the SNS topic. The ~5-minute delay between the second datapoint (07:36) and the alarm trigger (07:41) reflects CloudWatch's internal processing and metric aggregation latency.

**After the stress test (~08:10 UTC):**  
With the stress process completed (600s timeout), CPU returned to baseline. After the required number of below-threshold evaluation periods, the alarm returned to OK state and the monitoring system confirmed a clean recovery.

**Key observation:** The total time from stress start to alarm receipt was approximately 21 minutes. This lag is expected and by design — it eliminates false positives from short spikes. In a real production setting, this 10-minute window would be tunable by adjusting `period` and `evaluation-periods`.

---

## Real-World Use Cases for CloudWatch Alarms

**1. Auto-scaling triggers**  
Alarms on CPU or request count metrics can trigger EC2 Auto Scaling policies, automatically adding or removing instances in response to traffic changes. This is one of the most common and high-value uses of CloudWatch alarms.

**2. Database performance monitoring**  
RDS metrics such as `DatabaseConnections`, `ReadLatency`, and `FreeStorageSpace` can be monitored with alarms to catch connection pool exhaustion, slow queries, or disks filling up before they cause outages.

**3. Application error rate alerting**  
Custom metrics (published via the CloudWatch API or agent) can track application-level error rates, response times, or queue depths. Alarms fire when SLOs are at risk — for example, when the 5xx error rate exceeds 1% of requests.

**4. Cost anomaly detection**  
CloudWatch Billing alarms can alert when AWS spend unexpectedly spikes, providing a safety net against runaway resources, forgotten test environments, or compromised credentials performing unauthorized operations.

**5. Security and compliance monitoring**  
CloudTrail logs can be sent to CloudWatch Logs, and metric filters can count events like failed SSH logins, root account usage, or IAM policy changes. Alarms on these filters provide near-real-time security alerting.

**6. Serverless and container health**  
Lambda function errors, throttles, and duration; ECS task CPU and memory; EKS node conditions — all can be alarmed upon to maintain the health of modern containerized and serverless architectures.

