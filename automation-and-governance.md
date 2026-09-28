---
title: CloudWatch Automation, Governance & Ingestion
summary: Practical setup guide for wiring CloudWatch alarms to Auto Scaling and EventBridge, cross-account observability via OAM, Metric Streams, Logs data protection, Telemetry Configuration (Ingestion), and network/internet monitoring
last_verified: 2026-09-23
categories: [Amazon Web Services, Cloud monitoring, Observability, Cloud governance]
artifact: https://claude.ai/code/artifact/21d5a19b-7e9a-44d7-9f14-7d1356c8d345
---

# CloudWatch Automation, Governance & Ingestion

This article is the practical companion to [Amazon CloudWatch](amazon-cloudwatch.md), going one level deeper into the mechanics of wiring alarms to automated actions, sharing observability data across accounts, streaming metrics out of CloudWatch, masking sensitive log data, and monitoring network/internet paths. Each section includes the actual API/CLI shapes involved, not just the concept.

## Alarm actions → EC2 Auto Scaling

An alarm's `--alarm-actions` parameter accepts one or more ARNs to invoke when the alarm enters `ALARM` state. To drive Auto Scaling directly (no SNS/Lambda in between), the ARN points at an Auto Scaling **scaling policy**, not the Auto Scaling group itself.

```bash
# 1. Create a step/simple scaling policy on the Auto Scaling group and capture its PolicyARN
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name scale-out-on-cpu \
  --scaling-adjustment 1 \
  --adjustment-type ChangeInCapacity
# -> returns { "PolicyARN": "arn:aws:autoscaling:us-east-1:111122223333:scalingPolicy:...:autoScalingGroupName/my-asg:policyName/scale-out-on-cpu" }

# 2. Point a CloudWatch alarm at that policy ARN
aws cloudwatch put-metric-alarm \
  --alarm-name cpu-mon \
  --alarm-description "Alarm when CPU exceeds 70 percent" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions "Name=AutoScalingGroupName,Value=my-asg" \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:autoscaling:us-east-1:111122223333:scalingPolicy:...:autoScalingGroupName/my-asg:policyName/scale-out-on-cpu
```

`--alarm-actions` also accepts an SNS topic ARN (`arn:aws:sns:us-east-1:111122223333:MyTopic`) directly — no separate integration needed; CloudWatch publishes to SNS itself when the alarm fires.

## EventBridge: reacting to alarm state changes

Every CloudWatch alarm state transition is also emitted as an event on the default EventBridge bus, independent of `--alarm-actions`. This is the mechanism to use when the response is more than "notify" or "scale" — e.g. running a Systems Manager Automation document, invoking a Lambda that opens a ticket, or fanning out to multiple targets.

The event has `source: "aws.cloudwatch"` and `detail-type: "CloudWatch Alarm State Change"`. Minimal pattern matching any alarm state change:

```json
{
  "source": ["aws.cloudwatch"],
  "detail-type": ["CloudWatch Alarm State Change"]
}
```

Narrower pattern — only a specific alarm transitioning from `OK` to `ALARM`:

```json
{
  "source": ["aws.cloudwatch"],
  "detail-type": ["CloudWatch Alarm State Change"],
  "detail": {
    "alarmName": ["cpu-mon"],
    "state": { "value": ["ALARM"] },
    "previousState": { "value": ["OK"] }
  }
}
```

Wiring it up:

```bash
aws events put-rule \
  --name cpu-mon-to-alarm \
  --event-pattern file://pattern.json

aws events put-targets \
  --rule cpu-mon-to-alarm \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:111122223333:function:open-incident"

aws lambda add-permission \
  --function-name open-incident \
  --statement-id AllowEventBridge \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:111122223333:rule/cpu-mon-to-alarm
```

Note the naming history here: this event bus and rule engine is what CloudWatch Events used to be before AWS rebranded/evolved it into Amazon EventBridge (see [Amazon CloudWatch § History](amazon-cloudwatch.md)) — the API (`events:PutRule`, `events:PutTargets`) is the same either way.

## Cross-account observability (OAM)

CloudWatch's cross-account observability is implemented by the **Observability Access Manager (OAM)** service, made of two resource types:

- **Sink** — created once, in the *monitoring account*. It's the attachment point that source accounts link to.
- **Link** — created in each *source account*, pointing at the monitoring account's sink. Once linked, the source account's metrics/logs/traces become queryable from the monitoring account without copying data.

```bash
# In the monitoring account: create the sink and allow source accounts to attach
aws oam create-sink --name my-monitoring-sink
# -> capture the returned Sink ARN

aws oam put-sink-policy \
  --sink-identifier arn:aws:oam:us-east-1:111122223333:sink/abc123 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "AWS": "444455556666" },
      "Action": ["oam:CreateLink", "oam:UpdateLink"],
      "Resource": "*",
      "Condition": {
        "ForAllValues:StringEquals": {
          "oam:ResourceTypes": ["AWS::CloudWatch::Metric", "AWS::Logs::LogGroup", "AWS::XRay::Trace"]
        }
      }
    }]
  }'

# In each source account: link it to the monitoring account's sink
aws oam create-link \
  --label-template "$AccountName" \
  --resource-types "AWS::CloudWatch::Metric" "AWS::Logs::LogGroup" "AWS::XRay::Trace" \
  --sink-identifier arn:aws:oam:us-east-1:111122223333:sink/abc123
```

The IAM principal creating the link needs `oam:CreateLink` + `oam:TagResource` in the source account, and the sink's policy must grant `oam:CreateLink` to that source account. A link can optionally scope which metric namespaces and which log groups are shared — you are not forced to expose everything. Once linked, the monitoring account's CloudWatch console gets an account-selector on dashboards, Logs Insights, and alarms, letting one team query across every linked source account.

## Metric Streams

A metric stream continuously pushes CloudWatch metrics to a destination — almost always an **Amazon Data Firehose** delivery stream, which can in turn land data in S3, forward to an HTTP endpoint, or hand off to a partner integration (Datadog, Dynatrace, Honeycomb, etc.).

Setup shape:

1. Create a Firehose delivery stream with source **Direct PUT** and a destination — e.g. an HTTP endpoint destination for a third-party vendor, or S3 for a self-managed data lake.
2. Create the metric stream itself, pointing at that Firehose stream, choosing an **output format**:
   - `json` — plain, human-readable JSON, one object per metric per period.
   - `opentelemetry0.7` — binary Protobuf `ExportMetricsServiceRequest` messages (OpenTelemetry proto v0.7.0), with metric name formatted as `amazonaws.com/{namespace}/{metric_name}` and AWS-specific resource attributes (`cloud.provider`, `cloud.account.id`, `cloud.region`, `aws.exporter.arn`).
   - `opentelemetry1.0` — the newer OTLP-aligned format; AWS documents it as fully compatible with, and a superset of, 0.7 in terms of which metrics are exportable, while 0.7 continues to be supported.
3. Optionally scope the stream with an include/exclude filter by namespace, so only relevant metrics leave the account.

```bash
aws cloudwatch put-metric-stream \
  --name my-metric-stream \
  --firehose-arn arn:aws:firehose:us-east-1:111122223333:deliverystream/my-stream \
  --role-arn arn:aws:iam::111122223333:role/CWMetricStreamsRole \
  --output-format opentelemetry1.0 \
  --include-filters Namespace=AWS/EC2 Namespace=AWS/RDS
```

Metric Streams exists specifically to get around the throttling limits of polling `GetMetricData`/`ListMetrics` — instead of an external tool pulling on a schedule, CloudWatch pushes near-real-time as metrics land.

## Logs: Data Protection (masking sensitive data)

A **data protection policy** on a log group (or account-wide, via `PutAccountPolicy`) masks sensitive data *as it is ingested* — it has no effect on data already in the log group. A policy document requires exactly two statement blocks with matching `DataIdentifier` lists: one with an `Audit` operation (finds and optionally routes findings to a destination) and one with a `Deidentify` operation (actually masks it with asterisks for viewers who lack `logs:Unmask`).

```json
{
  "Name": "data-protection-policy",
  "Description": "Mask emails and US driver's license numbers",
  "Version": "2021-06-01",
  "Statement": [
    {
      "Sid": "audit-policy",
      "DataIdentifier": [
        "arn:aws:dataprotection::aws:data-identifier/EmailAddress",
        "arn:aws:dataprotection::aws:data-identifier/DriversLicense-US"
      ],
      "Operation": {
        "Audit": {
          "FindingsDestination": {
            "CloudWatchLogs": { "LogGroup": "dpp-findings" },
            "Firehose": { "DeliveryStream": "dpp-findings-stream" },
            "S3": { "Bucket": "dpp-findings-bucket" }
          }
        }
      }
    },
    {
      "Sid": "redact-policy",
      "DataIdentifier": [
        "arn:aws:dataprotection::aws:data-identifier/EmailAddress",
        "arn:aws:dataprotection::aws:data-identifier/DriversLicense-US"
      ],
      "Operation": {
        "Deidentify": { "MaskConfig": {} }
      }
    }
  ]
}
```

```bash
aws logs put-data-protection-policy \
  --log-group-identifier my-log-group \
  --policy-document file://policy.json
```

The `FindingsDestination` object is optional but, if present, its named destinations (log group, Firehose stream, S3 bucket) must already exist — the API does not create them. A user with `logs:Unmask` can still see the real values via `GetLogEvents`/`FilterLogEvents` with `unmask=true`, or in Logs Insights with the `unmask` query command.

## Internet Monitor

Internet Monitor works by first defining a **monitor**: a named set of resources (VPCs, CloudFront distributions, WorkSpaces directories, or Amazon EC2 instance/ALB associations) whose client-side internet traffic AWS wants to profile. From the resources listed, Internet Monitor builds a traffic profile of the *city-networks* (city + client ASN pairs) actually used to reach the application.

Two settings control the monitor's behavior and cost:

- **Traffic percentage** — how much of the resource's traffic Internet Monitor samples to build its profile (default 100%; below 100% may miss some city-networks entirely).
- **Health event thresholds** — a monitor only creates a **health event** when *both* a local availability/performance score drops below a configured threshold (default: 95%) for a geography, *and* the estimated overall traffic impact exceeds a separately configured percentage — this two-part gate avoids paging on a statistically noisy score drop that affects almost no real users.

```bash
aws internetmonitor create-monitor \
  --monitor-name my-app-monitor \
  --resources arn:aws:ec2:us-east-1:111122223333:vpc/vpc-0123456789abcdef0 \
  --traffic-percentage-to-monitor 100 \
  --health-events-config '{
    "AvailabilityScoreThreshold": 95,
    "PerformanceScoreThreshold": 95
  }'
```

A health event, once created, includes the impacted geography, client ASN(s), the affected percentage of traffic, and whether the cause looks like an AWS-side or internet-side (ISP/transit) issue — the whole point of Internet Monitor is separating "the problem is on our side" from "the problem is somewhere on the open internet, outside anything AWS or the customer can directly fix."

## Network Flow Monitor

Network Flow Monitor is agent-based, not passive — it measures *actual workload* traffic (not synthetic probes) by running a lightweight agent alongside real EC2 instances and Kubernetes workloads. It reports near-real-time metrics like retransmissions, packet loss, latency, and data transferred, for flows both between EC2 instances and toward other AWS services (S3, DynamoDB, etc.).

For EC2, the agent is distributed as an AWS Systems Manager **Distributor package** named `AmazonCloudWatchNetworkFlowMonitorAgent`:

```bash
# Install the agent via SSM (instance must have SSM Agent + the right IAM role)
aws ssm create-association \
  --name AmazonCloudWatchNetworkFlowMonitorAgent \
  --targets "Key=tag:Environment,Values=production"
```

The instance role needs the `CloudWatchNetworkFlowMonitorAgentPublishPolicy` managed policy attached so the agent can publish flow metrics. After installation, the agent must be explicitly **activated** before it starts sending data — installing alone is not sufficient. For self-managed Kubernetes, the agent is deployed as a workload rather than through SSM.

This is the tool for the specific question "is this slowdown my application, or the network path underneath it?" — it isolates network-layer symptoms (retransmits, loss) from application-layer ones (which would show up in Container/Lambda Insights or Application Signals instead).

## Telemetry Configuration (the "Ingestion" page)

**Telemetry Configuration** — the console's **Ingestion** page, described there as letting you "view and understand your telemetry collection coverage across AWS data sources" — is an audit/governance tool, not a data pipeline: it doesn't move or transform telemetry, it tells you whether telemetry is even switched on across your resources, and can optionally turn it on for you. It's built on the same `observabilityadmin` API family as [resource tags for telemetry](cloudwatch-settings.md#resource-tags-for-telemetry), but is a distinct, sibling capability, not the same feature.

It has two halves:

### Discovery & coverage

Scans the account (or, if enabled org-wide, every member account) and reports, per resource type, what fraction have a given signal turned on — e.g. "73% of EC2 instances have Detailed Monitoring enabled," "12 Lambda functions have Active Tracing off." Two views: an aggregate **Telemetry config** page, and a per-resource **Discovered data sources** list, filterable by account ID or tags.

Discovery currently covers a fixed, narrower set of resource↔signal pairs than enablement rules do:

| Resource | Signal discovered |
|---|---|
| EC2 | Detailed Monitoring (metrics) |
| VPC | Flow Logs |
| VPC | Route 53 Resolver query logs |
| Lambda | Active Tracing (X-Ray) |
| EKS | Control Plane Logs |
| WAFv2 | WAF Logs |
| NLB | Access logs (ALBs are not yet supported for discovery) |

Under the hood, discovery runs on an **AWS Config service-linked configuration recorder** — CloudWatch piggybacks on Config's resource-discovery machinery, with no additional charge for the config items used this way. A discovery pass can take up to ~24 hours to reflect newly created resources.

### Enablement rules

The fix-it half: a rule automatically turns on a missing signal for every matching resource — current and future — without hand-editing each one. The supported list is broader than discovery alone:

- Everything in the discovery table above, **except Lambda Active Tracing** — despite being a discovery pair, it does not appear in AWS's supported list for enablement rules at all, so it stays a manual/IaC-managed setting regardless of this feature.
- CloudTrail events, S3 server access logs, CloudFront logs, Amazon MSK cluster metrics, ALB access logs, Bedrock AgentCore logs/memory/gateway/traces, OpenTelemetry enrichment metrics, Security Hub findings, Bedrock Knowledge Base logs.

Documented defaults per rule (only what AWS states explicitly — several resource types have no documented default beyond "on/off"):

| Resource / signal | Default configuration | Configurable at rule creation? |
|---|---|---|
| VPC Flow Logs | Log group `/aws/vpc/<vpc-id>` if unspecified (`<account-id>` macro supported) | Log group pattern: yes. Format/fields/destination: no |
| EKS Control Plane Logs | Log group `/aws/eks/<cluster-name>/cluster`, auto-created | Which log types (`api`/`audit`/`authenticator`/`controllerManager`/`scheduler`): yes |
| WAF Web ACL Logs | Log group always prefixed `aws-waf-logs-` | Prefix fixed |
| Route 53 Resolver query logs | `/aws/route53resolver` if unspecified | Log group pattern: yes |
| NLB access logs | `/aws/nlb/access-logs` prefix if unspecified | Log group pattern: yes |
| CloudTrail events | Managed log groups `aws/cloudtrail/<event-types>`; retention follows the log group's own setting | Event type: yes |
| EC2 Detailed Monitoring | No interval/parameter documented | Nothing documented |
| Security Hub findings | Managed log group `aws/securityhub_cspm/findings` | Not stated |
| Bedrock AgentCore (Runtime/Browser/Code Interpreter) | Separate logs and traces rules; the traces rule auto-enables Transaction Search plus the needed X-Ray resource/permission policies | Not stated |
| Bedrock AgentCore Gateway/Memory/Workload Identity | `/aws/bedrock/agentcore` if unspecified | Log group pattern: yes |
| CloudFront logs | Only documented behavior: skips distributions already logging | Not stated |
| S3 server access logs | CloudWatch Logs destination only; bucket selection is **tag-based only** | Destination fixed |
| MSK cluster metrics | Metrics only | Monitoring granularity (`PER_BROKER`, `PER_TOPIC_PER_BROKER`, etc.): yes |
| OpenTelemetry enrichment metrics | Account-level only | No destination or resource-level targeting |
| ALB access logs | Types: access / connection / health-check logs; CloudWatch Logs destination only | Log type: yes. Destination fixed |
| Bedrock Knowledge Base | `APPLICATION_LOGS` type; CloudWatch Logs only | Not stated |

Cross-cutting behavior that applies to every rule: a rule only acts on resources an AWS Config discovery pass finds **non-compliant** — it never touches, duplicates, or overrides telemetry that's already flowing (whether you set it up manually or via IaC). Encryption defaults to an AWS-managed KMS key unless you supply your own (a multi-Region key is required for cross-Region rules). An org-level rule sets a floor that account/OU-level rules can only add to, never shrink.

### Where the enabled data actually lands

The Ingestion/Telemetry Config page itself is only a coverage dashboard — it never shows the underlying data. Once a signal is on (via a rule or manually), find the data in its native home:

- **Logs** (the large majority of the list: Flow Logs, EKS, WAF, Resolver, NLB/ALB, CloudTrail, S3, CloudFront, Security Hub, Bedrock Knowledge Base) → ordinary [CloudWatch Logs](logs.md) log groups, viewable via the Logs console, **Logs Insights**, or **Live Tail**.
- **Metrics** (EC2 Detailed Monitoring, MSK, OTel enrichment metrics) → ordinary [CloudWatch metrics](metrics-and-alarms.md) in their standard namespace (`AWS/EC2`, `AWS/Kafka`, etc.) — dashboard or alarm on them like any other metric.
- **Traces** — the *only* entry on the whole list that produces traces is **Bedrock AgentCore**; its traces rule enables **CloudWatch Transaction Search**, so those traces are viewable in the classic [X-Ray](aws-xray.md) console or, more usefully, queried via Transaction Search's Logs Insights-style syntax against the `aws/spans` log group (see [CloudWatch Generative AI Observability](genai-observability.md) and the query cookbook in the X-Ray article). Every other signal on this list — Flow Logs, WAF, CloudTrail, etc. — never touches X-Ray at all.

### What kind of governance this actually is

Worth being precise about, since it's easy to over-read: this is a **telemetry-coverage audit with an optional auto-remediation layer**, not a security or compliance audit. It checks whether a telemetry *setting* is on or off — the observability equivalent of "is the monitoring agent installed" — not whether a resource is configured securely (that's Config Rules / Security Hub) or whether the data being emitted is useful or well-tagged (that's a data-quality question this feature doesn't address). The governance angle comes entirely from enablement rules: an org can mandate "every Lambda gets tracing" as a standing rule for future resources too — the same shape as a Config conformance pack, just scoped to telemetry on/off rather than resource-configuration correctness.

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md) — the overview article this one extends
- [CloudWatch Settings & S3 Tables Integration](cloudwatch-settings.md) — a second export path for Logs data (Iceberg/S3 Tables) alongside Metric Streams, and the sibling "resource tags for telemetry" admin feature
- [CloudWatch Logs](logs.md) — where most telemetry-config-enabled signals actually land
- [CloudWatch Generative AI Observability](genai-observability.md) — Bedrock AgentCore tracing, the one signal on this page that flows to X-Ray
- Amazon EventBridge *(not yet written)*
- AWS Identity and Access Management (IAM) *(not yet written)*

## References

1. "put-metric-alarm" — AWS CLI Command Reference. <https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudwatch/put-metric-alarm.html>
2. "Alarm events and EventBridge" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-and-eventbridge.html>
3. "Amazon CloudWatch events" — Amazon EventBridge Reference. <https://docs.aws.amazon.com/eventbridge/latest/ref/events-ref-cloudwatch.html>
4. "CreateSink" / "CreateLink" — Amazon CloudWatch Observability Access Manager API Reference. <https://docs.aws.amazon.com/OAM/latest/APIReference/API_CreateSink.html>
5. "CloudWatch Observability Access Manager examples using AWS CLI" — AWS CLI User Guide. <https://docs.aws.amazon.com/cli/v1/userguide/cli_oam_code_examples.html>
6. "Use metric streams" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Metric-Streams.html>
7. "CloudWatch metric stream output in OpenTelemetry 0.7.0 format" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-metric-streams-formats-opentelemetry.html>
8. "PutDataProtectionPolicy" — Amazon CloudWatch Logs API Reference. <https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutDataProtectionPolicy.html>
9. "How Internet Monitor works" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-IM-inside-internet-monitor.html>
10. "Configuring thresholds for creating health events in Amazon CloudWatch Internet Monitor" — AWS Cloud Operations Blog. <https://aws.amazon.com/blogs/mt/configuring-thresholds-for-creating-health-events-in-amazon-cloudwatch-internet-monitor/>
11. "What is Network Flow Monitor?" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-NetworkFlowMonitor-What-is-NetworkFlowMonitor.html>
12. "Install agents on EC2 instances with SSM" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-NetworkFlowMonitor-agents-ec2-install-ssm.html>
13. "What is telemetry discovery and enablement?" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/telemetry-config-what-is.html>
14. "Setting up telemetry configuration" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/telemetry-config-turn-on.html>
15. "Enable telemetry configuration for your organization" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/telemetry-config-organization.html>
16. "Viewing AWS resource telemetry in CloudWatch" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/telemetry-config-view-resources.html>
17. "Working with telemetry enablement rules" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/telemetry-config-rules.html>

---
*Last verified 2026-09-23 against the sources above. CLI/API shapes and thresholds shown are illustrative of the documented pattern — check current parameter names against the linked reference pages before running against a live account. Telemetry Configuration in particular is a newer, actively evolving feature (its list of supported resource types/signals is likely to grow) — re-verify the discovery/enablement-rule coverage tables at docs.aws.amazon.com before relying on them.*
