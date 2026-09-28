---
title: CloudWatch Metrics & Alarms
summary: Practical, hands-on reference for publishing CloudWatch metrics (including EMF) and configuring alarms, composite alarms, and anomaly detection
last_verified: 2026-08-18
categories: [Amazon Web Services, Cloud monitoring, Observability]
artifact: https://claude.ai/code/artifact/f6bd7e61-469a-4883-978f-23ec8041bae5
---

# CloudWatch Metrics & Alarms

This article is the practical companion to [Amazon CloudWatch](amazon-cloudwatch.md): the actual data model, API/CLI syntax, and evaluation rules you need to publish metrics and build alarms that behave the way you expect.

## Metric identity: namespace, name, dimensions

A metric is uniquely identified by **namespace + metric name + the exact set of dimensions (name/value pairs)** used when it was published.

- **Namespace** — a container that isolates metrics from different applications so they don't get aggregated together. ASCII characters only (alphanumeric, `. - _ / # :` and space), 1–255 characters. There is no default namespace — you must always specify one. AWS's own namespaces follow `AWS/{service}` (e.g. `AWS/EC2`); **don't use an `AWS/`-prefixed namespace for custom metrics.**
- **Dimensions** — up to **30 dimensions per metric**. Each unique *combination* of dimension values is a distinct metric, even under the same metric name. Critically: **you can only query the exact dimension combinations you published.** If you publish `Server=Prod,Domain=Frankfurt` and `Server=Beta,Domain=Rio`, you cannot later query just `Server=Prod` across all domains — CloudWatch does not aggregate across dimensions for custom metrics (it does for some AWS-service metrics, like unfiltered `AWS/EC2` queries). The escape hatch is the **`SEARCH()`** metric-math function, which can pull back multiple metrics matching a dimension pattern.
- **Timestamps** — up to 2 weeks in the past, up to 2 hours in the future. Data older than 24h can take up to 48h to become queryable; data 3–24h old can take up to 2h. Use UTC.
- Metrics **cannot be deleted**; they simply expire after 15 months with no new data (rolling).

## Resolution & retention

Every metric is either **standard resolution** (1-minute granularity) or **high resolution** (1-second granularity, only available for custom metrics you explicitly publish that way). AWS-service metrics are standard resolution by default.

Retention is **by the resolution of the stored data point**, and older data is progressively downsampled — it isn't deleted, just aggregated to a coarser period:

| Data point period | Retained for |
|---|---|
| < 60 seconds (high-resolution) | 3 hours |
| 60 seconds (1 minute) | 15 days |
| 300 seconds (5 minutes) | 63 days |
| 3600 seconds (1 hour) | 455 days (~15 months) |

So a metric you publish at 1-minute resolution is queryable at 1-minute granularity for 15 days, then only at 5-minute granularity until day 63, then only at 1-hour granularity until day 455. **This is why you can't retroactively get fine-grained history** — decide your resolution needs up front.

Valid alarm/query **periods** are 1, 5, 10, 30, or any multiple of 60 seconds. Only metrics stored with 1-second `StorageResolution` support sub-minute periods (1/5/10/30s); everything else must use multiples of 60.

## Publishing custom metrics: PutMetricData

### CLI

```bash
# Single value, one dimension
aws cloudwatch put-metric-data \
  --namespace "MyApp/Orders" \
  --metric-name OrdersProcessed \
  --unit Count \
  --value 42 \
  --dimensions Environment=production,Service=checkout

# High-resolution (1-second) metric
aws cloudwatch put-metric-data \
  --namespace "MyApp/Orders" \
  --metric-name QueueDepth \
  --value 17 \
  --storage-resolution 1

# Pre-aggregated statistic set (avoids one API call per data point)
aws cloudwatch put-metric-data \
  --namespace "MyApp/Orders" \
  --metric-name RequestLatency \
  --statistic-values Sum=4500,Minimum=12,Maximum=890,SampleCount=60
```

- **`--value`** publishes one data point at a time — fine for low-frequency metrics.
- **`--statistic-values`** (`SampleCount, Sum, Minimum, Maximum`) lets you pre-aggregate many samples client-side (e.g. "all page-load times in the last minute") into a single API call instead of one call per sample. This is the recommended pattern for anything measured many times per minute. Caveat: **percentile statistics require raw data points** unless the statistic set collapses to a single value (`SampleCount=1` with `Min=Max=Sum`, or `Min=Max` and `Sum = Min × SampleCount`).
- **`Values` + `Counts` arrays** (SDK-level `MetricDatum.Values`/`Counts`) let you publish up to **150 distinct values** (with repeat counts) in a single metric per `PutMetricData` call, and do support percentile retrieval.

### Limits

- Max **1000 metrics per `PutMetricData` call** (combined across `MetricData` and `EntityMetricData`).
- Max **1 MB per HTTP POST** request (gzip-compressible).
- Values must be in `-2^360` to `2^360`; `NaN`/`Infinity` are rejected.
- New metrics can take **up to 15 minutes** to appear in `ListMetrics`/console search — use `GetMetricData` directly if you need it immediately.
- **Every `PutMetricData` call is billed** — calling it more often for a high-resolution metric directly increases cost. Prefer statistic sets or EMF (below) over one call per sample.

## Embedded Metric Format (EMF): the preferred high-cardinality path

Rather than calling `PutMetricData` from application code, you can write a specially structured JSON line to **CloudWatch Logs**, and CloudWatch automatically extracts one or more metrics from it — no separate API call, no extra cost beyond normal log ingestion. This is how the CloudWatch agent, Lambda's built-in metrics, and most modern instrumentation libraries publish custom metrics, and it's the recommended approach for high-cardinality or high-frequency custom metrics because it avoids `PutMetricData`'s per-call cost and throttling.

```json
{
  "_aws": {
    "Timestamp": 1735689600000,
    "CloudWatchMetrics": [
      {
        "Namespace": "MyApp/Orders",
        "Dimensions": [["Service", "Environment"]],
        "Metrics": [
          { "Name": "ProcessingTime", "Unit": "Milliseconds" },
          { "Name": "OrdersProcessed", "Unit": "Count" }
        ]
      }
    ]
  },
  "Service": "checkout",
  "Environment": "production",
  "ProcessingTime": 152,
  "OrdersProcessed": 1
}
```

Key points:
- The `_aws.CloudWatchMetrics[].Dimensions` array lists which top-level JSON keys should become dimension name/value pairs — the actual values (`"checkout"`, `"production"`) live alongside as normal top-level fields, so the same log line is both a structured log event (queryable in Logs Insights) *and* a metric source.
- `Timestamp` is milliseconds since epoch.
- You can emit multiple `MetricDirective` objects and multiple dimension sets from one log event.
- Because the metric is derived from a log event, it inherits log ingestion cost/throughput characteristics rather than `PutMetricData` call-based cost — usually cheaper at high volume.

## Metric Math

Metric math lets you combine/transform existing metrics into a new time series, used in dashboards, Metrics Insights, and directly inside alarms. Common functions:

```text
SUM(METRICS())                          -- sum every metric matched by a search expression
RATE(m1)                                -- per-second rate of change of m1
METRICS()                               -- all metrics referenced by other math expressions in the same request
ANOMALY_DETECTION_BAND(m1, 2)           -- expected-value band for m1, width = 2 standard deviations
SEARCH('{MyApp/Orders,Service} MetricName="OrdersProcessed"', 'Sum', 300)
```

`ANOMALY_DETECTION_BAND`'s second argument is the band width in standard deviations: `3` = low sensitivity (fewer alarms, wider band), `2` = medium/recommended, `1.5` = high sensitivity (more alarms, narrower band).

## Alarms

### Anatomy of an alarm

Three settings control when an alarm fires:

- **Period** — how long each evaluated data point covers (e.g. 300 = 5 minutes).
- **Evaluation Periods** — how many of the most recent periods to look at.
- **Datapoints to Alarm** — how many of those evaluation periods must be breaching to trigger `ALARM` (an "M out of N" alarm — e.g. 2 out of 3).

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "checkout-high-latency" \
  --namespace "MyApp/Orders" \
  --metric-name ProcessingTime \
  --dimensions Name=Service,Value=checkout \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --threshold 500 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

This is a **2-out-of-3** alarm: it fires only when 2 of the last 3 five-minute periods breach 500ms average — not on a single spike.

### Alarm states

- **`OK`** — metric is within threshold.
- **`ALARM`** — threshold breached per the evaluation rule above.
- **`INSUFFICIENT_DATA`** — not enough real data points exist to evaluate (only reachable depending on `treat-missing-data`, see below).

Alarms only invoke actions **on state change** (not on every evaluation) — except Auto Scaling actions, which re-invoke every minute the alarm remains in the triggering state. Alarm history is retained for **30 days**.

### `treat-missing-data`: the setting almost everyone gets wrong

When some expected data points don't arrive (dead connection, intermittent metric, resource simply idle), CloudWatch needs a rule for what to do with the gap. `--treat-missing-data` accepts:

| Value | Missing data points are treated as | Typical use |
|---|---|---|
| `missing` (default) | Neither good nor bad — if the whole evaluation window ends up missing, alarm goes to `INSUFFICIENT_DATA` | General default; safe but can mask real outages if the metric source itself dies |
| `notBreaching` | "Good" / within threshold | Metrics that only emit data *when something happens* (e.g. DynamoDB `ThrottledRequests` — no data = no throttling = good) |
| `breaching` | "Bad" / violating threshold | Metrics that should report continuously — a gap itself likely signals a problem (e.g. a health-check heartbeat metric, or an EC2 recovery-action alarm) |
| `ignore` | N/A — the alarm **keeps its current state** unchanged | When you don't want gaps to cause any transition either way |

CloudWatch actually queries a wider **evaluation range** than just your Evaluation Periods, specifically so it can fill gaps with real older data points before ever falling back to your missing-data setting — so occasional single-point gaps often don't invoke the missing-data rule at all. There's also a "premature alarm" safeguard: if the *oldest* datapoint in range is breaching and everything more recent is breaching-or-missing, the alarm goes to `ALARM` regardless of your missing-data setting, on the theory that a real breach shouldn't be hidden behind a string of gaps.

**Practical rule of thumb:** for infrastructure metrics that are expected to flow continuously and whose *absence* often signals the actual problem (an unhealthy instance stopped reporting), set `breaching`. For event-driven metrics that legitimately go quiet when nothing bad is happening, set `notBreaching`.

### Evaluation window limits

- Max evaluation period (Period × Evaluation Periods) is **7 days** for alarms with Period ≥ 1 hour, and **1 day** for alarms with a shorter period (also 1 day for alarms on a custom Lambda data source).
- Alarms on a **high-resolution metric** can use a period of 10s or 30s (billed at a higher rate) or any 60s-multiple period like a normal alarm.

### Composite alarms

A composite alarm evaluates a Boolean **AlarmRule** expression over the states of other alarms, and only enters `ALARM` when the whole expression is true — this is the primary noise-reduction tool for correlated failures (e.g. one AZ outage tripping 20 individual alarms).

```bash
aws cloudwatch put-composite-alarm \
  --alarm-name "checkout-service-unhealthy" \
  --alarm-rule "ALARM(checkout-high-latency) AND ALARM(checkout-high-error-rate)" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

- Functions available: `ALARM("name")`, `OK("name")`, `INSUFFICIENT_DATA("name")`, combined with `AND` / `OR` / `NOT` and parentheses.
- Limits: up to **100 child alarms** referenced, up to **500 elements** total in the expression.
- Composite alarms can send SNS notifications and open OpsItems/incidents on `ALARM`, but **cannot** perform EC2 or Auto Scaling actions directly (only metric alarms can).
- Cross-account composite alarms are **not supported**.

### Anomaly detection alarms

Instead of a static threshold, an anomaly-detection alarm compares a metric against a machine-learned expected-value band:

```bash
# 1. Create the detector (trains a model on historical data)
aws cloudwatch put-anomaly-detector \
  --namespace "AWS/ApplicationELB" \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/my-alb/50dc6c495c0c9188 \
  --stat Sum

# 2. Alarm on the band, using metric math
aws cloudwatch put-metric-alarm \
  --alarm-name "alb-requestcount-anomaly" \
  --comparison-operator LessThanLowerOrGreaterThanUpperThreshold \
  --evaluation-periods 2 \
  --metrics '[
    {"Id":"m1","MetricStat":{"Metric":{"Namespace":"AWS/ApplicationELB","MetricName":"RequestCount","Dimensions":[{"Name":"LoadBalancer","Value":"app/my-alb/50dc6c495c0c9188"}]},"Period":300,"Stat":"Sum"},"ReturnData":true},
    {"Id":"ad1","Expression":"ANOMALY_DETECTION_BAND(m1, 2)","ReturnData":true}
  ]' \
  --threshold-metric-id ad1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

- Band width (the `2` in `ANOMALY_DETECTION_BAND(m1, 2)`) is in standard deviations: lower = more sensitive/noisier, higher = less sensitive.
- `ComparisonOperator` for anomaly alarms is one of `LessThanLowerOrGreaterThanUpperThreshold`, `LessThanLowerThreshold`, or `GreaterThanUpperThreshold`.
- **Not supported for cross-account alarms.**

## Gotchas worth remembering

- **You can't query dimension subsets you never published** — plan your dimension strategy before you start publishing, since you can't retroactively "roll up" custom metrics across a dimension without `SEARCH()`.
- **Resolution decisions are effectively permanent for history** — retention is tied to the resolution the data was stored at; you can't get fine-grained history back after it ages past that period's retention window.
- **Custom metric cost scales with `PutMetricData` calls, not data volume** — batch with statistic sets, `Values`/`Counts` arrays, or EMF instead of one call per sample.
- **High-resolution alarms cost more** — both the high-resolution custom metric itself and a sub-minute-period alarm on it are billed at a higher rate than standard.
- **`missing` (the default) is not the same as `notBreaching` or `ignore`** — a metric that goes fully quiet under the default setting lands in `INSUFFICIENT_DATA`, not `OK`, which is a common source of "why is this alarm gray and not green" confusion.
- **Cross-account alarms** support metric math, but not `ANOMALY_DETECTION_BAND`, `INSIGHT_RULE`, or `SERVICE_QUOTA` functions, and composite alarms don't support cross-account at all.

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md)
- [CloudWatch Dashboards](dashboards.md) — visualizing these metrics and alarm states on custom/automatic dashboards

## References

1. "Metrics concepts" (namespaces, dimensions, resolution, retention) — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html>
2. "PutMetricData" — Amazon CloudWatch API Reference. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricData.html>
3. "put-metric-data" — AWS CLI Command Reference. <https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-data.html>
4. "Specification: Embedded metric format" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html>
5. "Using Amazon CloudWatch alarms" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Alarms.html>
6. "Configuring how CloudWatch alarms treat missing data" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarms-and-missing-data.html>
7. "PutCompositeAlarm" — Amazon CloudWatch API Reference. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutCompositeAlarm.html>
8. "Using CloudWatch anomaly detection" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html>
9. "put-anomaly-detector" — AWS CLI Command Reference. <https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-anomaly-detector.html>

---
*Last verified 2026-08-18 against the sources above. AWS documentation changes frequently — re-verify version-specific limits (dimension counts, retention windows, payload sizes) at docs.aws.amazon.com before relying on them.*
