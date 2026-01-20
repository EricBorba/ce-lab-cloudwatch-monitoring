# Lab M2.06 - CloudWatch Monitoring and Alarms

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-cloudwatch-monitoring](https://github.com/cloud-engineering-bootcamp/ce-lab-cloudwatch-monitoring)

**Activity Type:** Individual  
**Estimated Time:** 30-45 minutes

## Learning Objectives

- [ ] View and interpret CloudWatch metrics
- [ ] Create CloudWatch alarms
- [ ] Set up SNS notifications
- [ ] Install CloudWatch agent for custom metrics
- [ ] Monitor application health

## Your Task

Set up comprehensive monitoring for your EC2 instance:
1. Create CPU utilization alarm
2. Create StatusCheckFailed alarm
3. Set up email notifications
4. Stress test instance to trigger alarm
5. Document monitoring strategy

**Success:** Receive email when CPU exceeds threshold

## Quick Steps

```bash
# 1. Create SNS topic for notifications
aws sns create-topic --name ec2-alerts
aws sns subscribe --topic-arn arn:aws:sns:region:account:ec2-alerts \
  --protocol email --notification-endpoint your@email.com
# (Confirm subscription in email)

# 2. Create CPU alarm
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
  --dimensions Name=InstanceId,Value=i-xxxxx \
  --alarm-actions arn:aws:sns:region:account:ec2-alerts

# 3. Stress test to trigger alarm
ssh instance
sudo yum install -y stress
stress --cpu 2 --timeout 600s
# Wait 10 minutes for alarm

# 4. Check alarm status
aws cloudwatch describe-alarms --alarm-names high-cpu-alarm
```

## 📤 What to Submit

**Submission Type:** File Upload (ZIP)

Create a ZIP file named `lab-m2-cloudwatch-monitoring.zip` containing:

1. **CloudWatch Configuration:**
   - `alarm-config.json` (your alarm configuration exported/documented)
   - `sns-topic-details.txt` (SNS topic ARN and subscription info)
   - `metrics-query.txt` (any custom metrics queries you created)

2. **Screenshots** (in `screenshots/` folder):
   - CloudWatch Metrics dashboard showing CPU utilization
   - Alarm creation in AWS Console
   - Alarm in "ALARM" state (after triggering)
   - Alarm in "OK" state (after resolving)
   - SNS topic subscription confirmation
   - Email notification received
   - CloudWatch Logs (if configured)

3. **Test Documentation:**
   - `stress-test-log.txt` (output from your CPU stress test)
   - `alarm-timeline.txt` (when alarm triggered, duration, recovery)

4. **Lab Report** (`README.md`):
   - Step-by-step process
   - Why monitoring is important
   - Explanation of your alarm thresholds (why did you choose them?)
   - How SNS notifications work
   - Metrics analysis: What did you observe during the stress test?
   - Real-world use cases for CloudWatch alarms

**File Structure:**
```
lab-m2-cloudwatch-monitoring.zip
├── README.md
├── configs/
│   ├── alarm-config.json
│   ├── sns-topic-details.txt
│   └── metrics-query.txt
├── test-results/
│   ├── stress-test-log.txt
│   └── alarm-timeline.txt
└── screenshots/
    ├── 01-cloudwatch-metrics-dashboard.png
    ├── 02-alarm-creation.png
    ├── 03-alarm-triggered.png
    ├── 04-alarm-resolved.png
    ├── 05-sns-subscription.png
    ├── 06-email-notification.png
    └── 07-cloudwatch-logs.png (optional)
```

## Grading: 100 points
- Alarm creation and configuration: 30pts
- SNS notifications working (email received): 25pts
- Metrics analysis and testing: 25pts
- Documentation and insights: 20pts
