# AWS CloudWatch Auto Alarms

An AWS Lambda function (Python) that automatically creates and deletes CloudWatch alarms for EC2 instances and Lambda functions based on their tags. Triggered by EventBridge (CloudWatch Events).

## How It Works

- **EC2 instance starts** → checks for the `Create_Auto_Alarms` tag → creates CPU, memory, and disk alarms
- **EC2 instance terminates** → deletes all `AutoAlarm-*` alarms for that instance
- **Lambda function tagged** → creates error and throttle alarms
- **Lambda function deleted** → deletes its alarms
- **Manual scan** → invoke with `{"action": "scan"}` to process all existing instances

## Default Alarms Created

| Service | Metric | Threshold |
|---|---|---|
| EC2 | CPUUtilization | 75% |
| Lambda | Errors | ≥ 1 |
| Lambda | Throttles | ≥ 1 |
| EC2 (CWAgent) | mem_used_percent | 75% |
| EC2 (CWAgent) | disk_used_percent | 80% |

## Setup

### 1. Deploy the Lambda

- Runtime: **Python 3.11**
- Handler: `cw_auto_alarms.lambda_handler`

### 2. Environment Variables

| Variable | Default | Description |
|---|---|---|
| `ALARM_TAG` | `Create_Auto_Alarms` | Tag key to trigger alarm creation |
| `CREATE_DEFAULT_ALARMS` | `true` | Create default alarms automatically |
| `CLOUDWATCH_NAMESPACE` | `CWAgent` | Namespace for CW Agent metrics |
| `DEFAULT_ALARM_SNS_TOPIC_ARN` | *(empty)* | SNS topic for alarm actions |
| `ALARM_CPU_HIGH_THRESHOLD` | `75` | CPU alarm threshold (%) |
| `ALARM_MEMORY_HIGH_THRESHOLD` | `75` | Memory alarm threshold (%) |
| `ALARM_DISK_PERCENT_LOW_THRESHOLD` | `20` | Minimum free disk (%) |

### 3. IAM Permissions

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:DescribeInstances",
    "ec2:DescribeImages",
    "ec2:CreateTags",
    "cloudwatch:PutMetricAlarm",
    "cloudwatch:DescribeAlarms",
    "cloudwatch:DeleteAlarms",
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Resource": "*"
}
```

### 4. EventBridge Rules

Create two EventBridge rules to trigger the Lambda:

**EC2 state changes:**
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": { "state": ["running", "terminated"] }
}
```

**Lambda tag/delete events:**
```json
{
  "source": ["aws.lambda"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventName": ["TagResource20170331v2", "DeleteFunction20150331"]
  }
}
```

## Tagging EC2 Instances

Add the tag `Create_Auto_Alarms` (any value) to an EC2 instance to enable auto alarm creation. Custom alarms can also be added via tags in the format:

```
AutoAlarm-<Namespace>-<MetricName>-<ComparisonOperator>-<Period>-<Statistic> = <Threshold>
```
