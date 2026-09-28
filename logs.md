---
title: CloudWatch Logs
summary: Practical guide to CloudWatch Logs — log groups/streams, storage classes, deletion protection, bearer token authentication, Insights queries, metric & subscription filters, Contributor Insights, Data Protection pricing, Live Tail, the CloudWatch agent, anomaly detection, and IAM
last_verified: 2026-09-27
categories: [Amazon Web Services, Cloud monitoring, Observability]
artifact: https://claude.ai/code/artifact/fe129108-8b1d-4fae-a1fd-d01c924e5569
---

# CloudWatch Logs

CloudWatch Logs is the log-ingestion and storage component of [Amazon CloudWatch](amazon-cloudwatch.md). This article goes past the overview into the parts you actually touch when operating it: structure and retention, the two storage classes, Logs Insights query syntax, turning log lines into metrics and real-time streams, Live Tail, the CloudWatch agent, log anomaly detection, and the IAM permissions each of these needs.

## Structure: groups, streams, events

- **Log group** — a namespace for a related set of logs (typically one per application/Lambda function/service), holding retention policy, storage class, and encryption settings.
- **Log stream** — a sequence of log events sharing the same source within a group (e.g. one stream per EC2 instance, per Lambda execution environment, per container task).
- **Log event** — a single timestamped record with a message payload, ingested via `PutLogEvents`, the CloudWatch agent, or a native AWS service integration.

### Retention

Retention is set per log group. By default, a log group's events **never expire**. To set an explicit retention period:

```bash
aws logs put-retention-policy \
  --log-group-name /my/app \
  --retention-in-days 90
```

`--retention-in-days` only accepts one of a fixed set of values:

```
1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365,
400, 545, 731, 1096, 1827, 2192, 2557, 2922, 3288, 3653
```

(3653 ≈ 10 years.) To revert to "never expire," call `delete-retention-policy` instead. Note: CloudWatch Logs doesn't delete expired events immediately — it can take up to 72 hours (occasionally longer) after the retention period is reached.

## Log classes: Standard vs. Infrequent Access

Set at **log group creation** (`aws logs create-log-group --log-group-name ... --log-group-class STANDARD|INFREQUENT_ACCESS`) and cannot be changed afterward — to switch, create a new group and repoint your log source.

| | Standard | Infrequent Access |
|---|---|---|
| Ingestion price | Full price (baseline) | ~50% cheaper ingestion |
| Storage / Insights query pricing | Same as Standard | Same as Standard |
| Live Tail | Yes | **No** |
| Metric filters / alarm-from-logs | Yes | **No** |
| Subscription filters | Yes | **No** |
| Data Protection (PII masking) | Yes | **No** |
| Logs Insights queries | Yes | Yes |
| Best for | Logs needing real-time monitoring or frequent access | Compliance/audit logs and forensic logs queried only ad hoc after an incident |

Only ingestion cost and the feature set differ — storage and query costs are identical, so Infrequent Access is a pure discount for logs you're not actively alarming or tailing.

## Deletion protection

A per-log-group boolean flag (added late 2025), **off by default**. Enable at creation or any time after, via console, CLI, or SDK:

```bash
aws logs put-log-group-deletion-protection \
  --log-group-identifier /my/app \
  --deletion-protection-enabled
```

Once enabled, any attempt to delete that log group fails with a `ValidationException` ("Cannot delete log group with deletion protection enabled. Disable deletion protection first.") until you explicitly disable it first. There's no MFA step and no soft-delete/recovery window involved — it's a pure gate check on the `DeleteLogGroup` call, nothing more.

## Data Protection: masking sensitive data

Covered in full, with a worked policy JSON, in [CloudWatch Automation, Governance & Ingestion § Logs: Data Protection](automation-and-governance.md#logs-data-protection-masking-sensitive-data). The short version relevant to what shows up in the console:

A data protection policy needs **two** statement blocks sharing the same `DataIdentifier` list, doing different jobs:

- **`Audit`** — finds matches and optionally routes findings to a destination (another log group, Firehose stream, or S3 bucket). This is also the source of the **Sensitive data count** shown per log group in the console — it's the `LogEventsWithFindings` metric in the `AWS/Logs` namespace, a count of log events matched by the policy's data identifiers, alarmable like any other metric.
- **`Deidentify`** — the statement that actually **masks** the matched text with asterisks for any viewer who lacks `logs:Unmask`. The underlying data isn't deleted, only hidden from view by default; someone with `logs:Unmask` can still retrieve the real value via `GetLogEvents`/`FilterLogEvents` with `unmask=true`, or Logs Insights' `unmask` command.

Masking (and the sensitive-data count) only applies going forward from when the policy is set — there's no retroactive scan of already-ingested events.

**Pricing — this is not bundled into ingestion/storage.** Data Protection charges **$0.12 per GB of log data scanned**, on top of normal ingestion and storage fees. AWS's own worked example: a log group ingesting 1 GB/day costs roughly $6.25/month in ingestion + ~$0.03/month storage, but Data Protection adds **~$3.60/month** (30 GB scanned × $0.12) — often more than 50% on top of the log group's other costs, since it scans every ingested byte rather than sampling. Scope it to log groups that genuinely handle PII rather than enabling broadly. Also unavailable on Infrequent Access log groups (see [Log classes](#log-classes-standard-vs-infrequent-access) above).

## Bearer token authentication

Lets a client that **can't do AWS SigV4 signing** — a plain OpenTelemetry exporter, for example — send logs to CloudWatch's HTTP ingestion endpoints (including the OTLP `/v1/logs` endpoint) with a simple `Authorization: Bearer <token>` header instead of full SDK/IAM signing.

Setup: generate the token as an IAM service-specific credential scoped to `logs.amazonaws.com`:

```bash
aws iam create-service-specific-credential \
  --user-name my-otel-exporter \
  --service-name logs.amazonaws.com
```

The returned `ServiceCredentialSecret` **is** the bearer token — shown exactly once, never retrievable again. Enable it per log group with `put-bearer-token-authentication`. Expiration is configurable from 1 to 36,600 days, or never (AWS explicitly advises against "never"). AWS's own guidance: prefer SigV4 with short-term credentials wherever the client supports it — bearer tokens exist specifically as a fallback for clients that structurally can't do SigV4, not as a general-purpose alternative.

## CloudWatch Logs Insights query syntax

Every query is a pipeline of commands joined with `|`. **Put `filter` as early as possible** — it's applied before later commands and dramatically cuts the data volume `stats`/`sort` have to process.

Core commands:

- **`fields`** — select/derive fields to work with (`fields @timestamp, @message, @message.statusCode`).
- **`filter`** — restrict to matching events (`filter status_code >= 500`, `filter @message like /timeout/`).
- **`stats`** — aggregate: `count()`, `sum()`, `avg()`, `min()`, `max()`, `pct()`, optionally `by` a grouping field.
- **`sort`** — `sort <field> desc|asc`.
- **`limit`** — cap results; defaults to 10,000, can be raised up to 100,000.
- **`parse`** — extract fields out of unstructured text using a glob-style pattern with `*` capture groups.
- **`display`** — set which fields are shown in the output (only the last `display` in a query is honored).

### Example 1 — top 10 slowest Lambda invocations

```
fields @timestamp, @duration, @requestId
| filter @type = "REPORT"
| sort @duration desc
| limit 10
```

### Example 2 — error count by HTTP status code, bucketed over time

```
fields @timestamp, statusCode
| filter statusCode >= 400
| stats count() as errorCount by statusCode, bin(5m)
| sort errorCount desc
```

### Example 3 — parsing an unstructured log line

Given a line like `2026-08-18 12:03:44 INFO user=alice action=login result=success`:

```
fields @timestamp, @message
| parse @message "* * * user=* action=* result=*" as date, time, level, user, action, result
| filter result = "success"
| stats count() by user
```

### Example 4 — 95th-percentile API latency by endpoint

```
fields @timestamp, latency, path
| stats pct(latency, 95) as p95 by path
| sort p95 desc
| limit 20
```

## Metric filters

A **metric filter** watches incoming log events for a pattern and increments (or publishes a value into) a CloudWatch metric each time it matches — turning log content into something you can graph and alarm on directly.

Filter pattern styles:

- **Unstructured text** — plain terms (`ERROR`), phrase match (`"connection refused"`), exclusion (`-DEBUG`), or `%glob%` substrings.
- **JSON logs** — field selectors on one line: `{ $.statusCode = 500 }`, with `&&` / `||` / `!=` / `>` / `<`:
  ```
  { $.eventType = "*" && $.sourceIPAddress != 123.123.* }
  ```
- **Space-delimited logs** (e.g. access logs) — name the positional fields, optionally with conditions:
  ```
  [ip, server, username, timestamp, request, status_code, bytes > 1000]
  [w1=ERROR || w1=%WARN%, w2]
  ```

Publish a *value from the log* (not just a count) as the metric, e.g. extracting a `latency` field:

```
{ $.latency = * }   # filter pattern
# metricValue: $.latency
```

Create one via the CLI:

```bash
aws logs put-metric-filter \
  --log-group-name /my/app \
  --filter-name Http5xxErrors \
  --filter-pattern '{ $.statusCode = 5* }' \
  --metric-transformations \
      metricName=Http5xxCount,metricNamespace=MyApp,metricValue=1,defaultValue=0
```

Always set a `defaultValue` (even `0`) — without one, CloudWatch reports nothing for minutes with no match instead of a clean zero, which distorts alarms and graphs. JSON/space-delimited filters can also publish up to **3 dimensions** extracted from the log fields — avoid high-cardinality fields (IP, request ID) as dimensions, since each unique value becomes a billed custom-metric time series, and AWS may disable a filter that generates >1000 distinct dimension combinations.

## Subscription filters

A **subscription filter** streams matching log events, in near real time, to a destination — for fan-out processing rather than just visualization:

- **Lambda** — same-account only, invoked asynchronously per batch of events.
- **Kinesis Data Firehose** — same-account delivery stream, typically landing in S3/OpenSearch/a third-party sink.
- **Kinesis Data Stream** — for custom streaming consumers.

Each log group supports **up to 2 subscription filters**, and log data delivered to the destination is Base64-encoded and gzip-compressed.

```bash
aws logs put-subscription-filter \
  --log-group-name /my/app \
  --filter-name ShipErrorsToFirehose \
  --filter-pattern 'ERROR' \
  --destination-arn arn:aws:firehose:us-east-1:123456789012:deliverystream/my-stream \
  --role-arn arn:aws:iam::123456789012:role/CWLtoFirehoseRole
```

(The `--role-arn`/`iam:PassRole` is required for Kinesis/Firehose destinations, but not for Lambda, where CloudWatch Logs invokes the function directly via a resource-based policy instead.)

## Contributor Insights

Despite the hub article's brief mention of it as metric-focused, Contributor Insights is actually **log-based**: it works by defining a **rule** against one or more log groups (wildcards allowed in `LogGroupNames`), not against a CloudWatch metric directly. A rule (JSON, `CloudWatchLogRule` schema) specifies:

- `LogFormat` — JSON, Common Log Format, space-delimited, or a custom pattern.
- `Contribution.Keys` — which field(s) in each log event identify "who is contributing" (an IP, a client ID, a tenant, a host).
- `Contribution.ValueOf` (optional) — a numeric field to **sum** per contributor, instead of just counting occurrences.
- `Filters` (optional) — restrict which events count at all.

The output is a time-series graph of the **top-N contributors** ranked by count or summed value, over the selected time range — addable to a dashboard like any other widget. Concrete example: rank VPC Flow Log source/destination IP pairs by total bytes transferred — `Keys: [srcaddr, dstaddr]`, `ValueOf: bytes`, `Aggregate: Sum`. This is the tool for "which specific client/IP/tenant is actually driving this spike," a question a plain aggregate metric can't answer on its own.

## Live Tail

Live Tail streams matching log events to the console (or CLI) as they arrive — for watching a deployment or incident unfold, not for historical analysis. **Not available on Infrequent Access log groups.**

```bash
aws logs start-live-tail \
  --log-group-identifiers arn:aws:logs:us-east-1:123456789012:log-group:/my/app \
  --log-event-filter-pattern 'ERROR 500'
```

The filter pattern accepts the same syntax as metric/subscription filters, including regular expressions, e.g. matching several status codes at once: `{ $.statusCode = %4[0-9]{2}% }`.

## The CloudWatch agent

The CloudWatch agent is a process you install on EC2, on-prem, or hybrid hosts to collect **both logs and metrics** (including custom StatsD/collectd metrics) that AWS services don't emit natively. Configuration is a single JSON file, referenced by the agent at startup.

Minimal config collecting a custom app log file and host CPU/memory, plus a StatsD listener for custom app metrics:

```json
{
  "agent": { "metrics_collection_interval": 60 },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/myapp/app.log",
            "log_group_name": "/myapp/app-log",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 30
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "cpu": { "measurement": ["cpu_usage_idle", "cpu_usage_user"] },
      "mem": { "measurement": ["mem_used_percent"] },
      "statsd": {
        "service_address": ":8125",
        "metrics_collection_interval": 10
      }
    }
  }
}
```

Install and run (Linux, via the SSM-distributed package):

```bash
sudo yum install -y amazon-cloudwatch-agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

`collectd` metrics (Linux only) are enabled the same way by adding a `"collectd": {}` block under `metrics_collected`; StatsD works on both Linux and Windows.

## Log anomaly detection

Enabled **per log group**. After creation, the detector trains against the **past two weeks** of log events (training takes up to ~15 minutes), building a baseline of the log group's normal structure via automatic pattern extraction (separating static text from dynamic/variable tokens in each log line). Once trained, it flags:

- New patterns never seen in the group before.
- Known patterns occurring at an abnormal rate/volume.

Optionally scope a detector to only evaluate events matching a filter pattern (**Anomaly detection filter pattern**), rather than the whole log group. Detected anomalies surface in the CloudWatch console under Logs → Log Anomalies, each linked back to the matching log events. Note this is a distinct feature from metric-level [Anomaly Detection](amazon-cloudwatch.md#log--metric-analytics), which works on numeric CloudWatch metrics rather than raw log text.

## IAM permissions

**Writing logs** (what an application/instance role needs to ingest):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents",
      "logs:DescribeLogStreams"
    ],
    "Resource": "arn:aws:logs:*:123456789012:log-group:/myapp/*"
  }]
}
```

**Querying logs** (what an analyst/dashboard role needs, e.g. for Logs Insights):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "logs:StartQuery",
      "logs:GetQueryResults",
      "logs:StopQuery",
      "logs:GetLogGroupFields",
      "logs:DescribeLogGroups"
    ],
    "Resource": "arn:aws:logs:*:123456789012:log-group:/myapp/*"
  }]
}
```

Scope the `Resource` ARN to the specific log group prefixes you need — CloudWatch Logs actions support resource-level restriction, so a query-only role never needs `PutLogEvents`, and an ingestion role never needs `StartQuery`.

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md) — parent overview article
- [[amazon-eventbridge]] *(not yet written)*

## References

1. "CloudWatch Logs Insights language query syntax" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html>
2. "Sample queries" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html>
3. "PutRetentionPolicy" — Amazon CloudWatch Logs API Reference. <https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutRetentionPolicy.html>
4. "put-retention-policy" — AWS CLI Command Reference. <https://docs.aws.amazon.com/cli/latest/reference/logs/put-retention-policy.html>
5. "Log classes" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch_Logs_Log_Classes.html>
6. "Filter pattern syntax for metric filters" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntaxForMetricFilters.html>
7. "put-metric-filter" — AWS CLI Command Reference. <https://docs.aws.amazon.com/cli/latest/reference/logs/put-metric-filter.html>
8. "Log group-level subscription filters" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html>
9. "PutSubscriptionFilter" — Amazon CloudWatch Logs API Reference. <https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutSubscriptionFilter.html>
10. "Troubleshoot with CloudWatch Logs Live Tail" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs_LiveTail.html>
11. "start-live-tail" — AWS CLI Command Reference. <https://awscli.amazonaws.com/v2/documentation/api/latest/reference/logs/start-live-tail.html>
12. "Amazon CloudWatch Logs announces regular expression filter pattern support for Live Tail" — AWS What's New, Nov 2023. <https://aws.amazon.com/about-aws/whats-new/2023/11/amazon-cloudwatch-logs-filter-pattern-live-tail>
13. "Collect metrics, logs, and traces using the CloudWatch agent" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html>
14. "Create the CloudWatch agent configuration file" / "Examples of configuration files" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file.html>
15. "Log anomaly detection" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/LogsAnomalyDetection.html>
16. "Enable anomaly detection on a log group" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/LogsAnomalyDetection-Enable.html>
17. "Using identity-based policies (IAM policies) for CloudWatch Logs" / "CloudWatch Logs permissions reference" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/iam-identity-based-access-control-cwl.html>
18. "Amazon CloudWatch Logs adds deletion protection" — AWS What's New, November 2025. <https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-cloudwatch-deletion-protection-logs>
19. "Protecting log groups from deletion" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/protecting-log-groups-from-deletion.html>
20. "PutLogGroupDeletionProtection" — Amazon CloudWatch Logs API Reference. <https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutLogGroupDeletionProtection.html>
21. "Setting up bearer token authentication" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_HTTP_Endpoints_BearerTokenAuth.html>
22. "Help protect sensitive log data with masking" / "Understanding data protection policies" — Amazon CloudWatch Logs User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/mask-sensitive-log-data.html>
23. "Amazon CloudWatch Pricing" — AWS product page (Data Protection: $0.12/GB scanned). <https://aws.amazon.com/cloudwatch/pricing/>
24. "Contributor Insights rule syntax" / "Rule examples" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContributorInsights-RuleSyntax.html>

---
*Last verified 2026-09-27 against the sources above. AWS CLI parameter names and console flows change over time — re-verify against docs.aws.amazon.com before relying on version-specific details. Deletion protection and bearer token authentication are newer features — re-check their exact API shapes at docs.aws.amazon.com before relying on them.*
