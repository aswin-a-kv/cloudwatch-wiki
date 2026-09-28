---
title: AWS X-Ray
summary: AWS's distributed request-tracing service — practical instrumentation, sampling, filtering, and its 2026 migration to OpenTelemetry
last_verified: 2026-09-21
categories: [Amazon Web Services, Distributed tracing, Observability]
artifact: https://claude.ai/code/artifact/7ddf6b11-6610-4e0d-8f4e-273678114488
---

# AWS X-Ray

**AWS X-Ray** is AWS's distributed request-tracing service. It reconstructs the path of a single request through the components of a distributed application — front-end service, downstream microservices, databases, external web APIs — even across process, container, and account boundaries, so you can see where time was spent and where something failed.

This article is the practical companion to the [Amazon CloudWatch](amazon-cloudwatch.md) overview: it goes deep on how to actually instrument an application, configure sampling, query traces, and what to use *today* given X-Ray's move toward OpenTelemetry.

> **Read this first.** On 29 October 2025, AWS announced that the **X-Ray SDKs and daemon enter maintenance mode on 25 February 2026**, with **end of support on 25 February 2027**. Everything under "Classic SDK instrumentation" below still works and is documented by AWS, but if you're instrumenting something new today, read [OpenTelemetry migration](#opentelemetry-what-to-use-today) first and seriously consider starting there instead.

## How it fits together

- A **segment** is the record of work done by one service/component for one request: timing, resource metadata, errors.
- A **subsegment** is a nested breakdown of a segment — a specific downstream call or a discrete chunk of function code.
- A **trace** is the full set of segments sharing one trace ID, i.e. everything that happened for one request across every service it touched.
- The **service map** (trace map) is a visual graph built from aggregated traces: every node a request passed through, annotated with volume, latency, and error rate per edge.
- A **sampling rule** decides what fraction of requests actually get traced.
- A **group** is a saved filter expression that traces are matched against as they arrive — used to scope dashboards, metrics, and Insights.
- **X-Ray Insights** is automatic anomaly detection over traces, scoped per group.
- The **X-Ray daemon** is a small local process that batches segment documents from your instrumented code and forwards them to the X-Ray API — required by the classic SDKs, not by OpenTelemetry-based instrumentation.

## Classic SDK instrumentation

The AWS X-Ray SDKs (Python, Node.js, Java, .NET, Ruby, Go) generate segments in-process and hand them to the X-Ray daemon over UDP. This is the "classic" path, now in its final years of active development.

### Python (`aws-xray-sdk`)

```bash
pip install aws-xray-sdk
```

**Flask** — configure the recorder and wrap the app:

```python
from flask import Flask
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

app = Flask(__name__)

xray_recorder.configure(service='my-application')
XRayMiddleware(app, xray_recorder)
```

**Django** — add the middleware first in `MIDDLEWARE` (so failures elsewhere still get recorded) and register the app:

```python
# settings.py
MIDDLEWARE = [
    'aws_xray_sdk.ext.django.middleware.XRayMiddleware',
    'django.middleware.security.SecurityMiddleware',
    # ... the rest of your middleware
]
INSTALLED_APPS = [
    'aws_xray_sdk.ext.django',
    # ... the rest of your apps
]
XRAY_RECORDER = {
    'AWS_XRAY_TRACING_NAME': 'my-application',
    'PLUGINS': ('EC2Plugin',),   # note the trailing comma — it's a tuple
}
```

**Patching downstream clients** — `patch_all()` instruments boto3/botocore, `requests`, `pymongo`, `psycopg2`, `pymysql`, `sqlite3`, and others automatically, so every downstream call shows up as a subsegment without touching call sites:

```python
from aws_xray_sdk.core import patch_all
patch_all()
```

Patch a subset instead with `patch(('boto3', 'requests'))`.

**Manual segments** (scripts, workers, anything without middleware support):

```python
from aws_xray_sdk.core import xray_recorder

segment = xray_recorder.begin_segment('segment_name')
subsegment = xray_recorder.begin_subsegment('subsegment_name')

subsegment.put_annotation('order_id', '12345')          # indexed, searchable
segment.put_metadata('request_body', payload, 'debug')   # not indexed, just stored

xray_recorder.end_subsegment()
xray_recorder.end_segment()
```

**Capture decorator** for tracing an arbitrary function as its own subsegment:

```python
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture('charge_card')
def charge_card(amount):
    ...
```

### Node.js (`aws-xray-sdk-core`)

```bash
npm install aws-xray-sdk
```

```js
const AWSXRay = require('aws-xray-sdk');
const express = require('express');
const app = express();

app.use(AWSXRay.express.openSegment('my-application'));

app.get('/', (req, res) => res.send('traced'));

app.use(AWSXRay.express.closeSegment());  // must be last, after all routes
```

Patch the AWS SDK client so every downstream call becomes a subsegment:

```js
const AWSXRay = require('aws-xray-sdk');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));
```

Annotations vs. metadata mirror the Python API:

```js
const segment = AWSXRay.getSegment();
segment.addAnnotation('orderId', '12345');   // indexed
segment.addMetadata('payload', body, 'debug'); // not indexed
```

### Annotations vs. metadata — the rule that matters

| | Annotations | Metadata |
|---|---|---|
| Searchable with filter expressions / `GetTraceSummaries` | Yes | No |
| Value types | string, number, boolean | any JSON-serializable value, including objects/arrays |
| Limits | up to 50 per trace; key ≤500 alphanumeric chars (dots allowed, no spaces/symbols); value ≤1,000 Unicode chars | no hard count limit, but avoid large/complex objects — hurts performance, and circular references will break serialization |
| Typical use | order ID, user ID, feature flag, region — anything you'll want to filter traces by later | request/response bodies, stack traces, debug context |

If you're not sure which to use: if you will ever want to run a query like `annotation[order_id] = "12345"`, it's an annotation. Otherwise, metadata.

### AWS Lambda: tracing without the daemon

Lambda runs the X-Ray daemon for you automatically on a sampled invocation — you don't manage it.

1. Console → your function → **Configuration** → **Monitoring and operations tools** → edit → enable **Active tracing** under AWS X-Ray.
2. The function's execution role needs `xray:PutTraceSegments` and `xray:PutTelemetryRecords` — attach the managed policy **`AWSXRayDaemonWriteAccess`**.
3. Bundle the X-Ray SDK with your deployment package only if you want subsegments for your own code/downstream calls. Without it, you still get a node for the Lambda service and one for the function in the service map — the SDK adds detail inside that.
4. **You cannot modify the segment Lambda creates.** You can create subsegments and annotate/add metadata to *those*, but not to the parent Lambda-owned segment.
5. If a request arrives already sampled (e.g. from an instrumented API Gateway or another traced Lambda), Lambda traces it automatically with no extra config. For requests with no upstream trace context (e.g. scheduled/EventBridge-triggered invocations), Active Tracing is what makes Lambda apply its own sampling and start a trace.
6. If function B is called by function A and you want B's traces, A must also have Active Tracing enabled — the sampling decision is made once, upstream (see "Parent-based sampling" below).

### The X-Ray daemon (EC2 / ECS / on-prem)

The daemon listens on UDP port 2000, buffers segment documents, and uploads them to X-Ray in batches. On EC2/on-prem you run it yourself; Elastic Beanstalk and Lambda run it for you.

**IAM**: attach `AWSXRayDaemonWriteAccess` to whatever role the daemon runs as (EC2 instance profile, ECS task role). That policy covers `xray:PutTraceSegments`, `xray:PutTelemetryRecords`, and enough read access (`GetSamplingRules`, `GetSamplingTargets`) for the daemon to fetch remote sampling config.

**ECS sidecar**, using the official image (bridge networking, host port 0 = dynamically assigned):

```json
{
  "name": "xray-daemon",
  "image": "amazon/aws-xray-daemon",
  "cpu": 32,
  "memoryReservation": 256,
  "portMappings": [
    { "hostPort": 0, "containerPort": 2000, "protocol": "udp" }
  ]
}
```

Point your app container at it with `AWS_XRAY_DAEMON_ADDRESS=xray-daemon:2000` plus a `links` entry to `xray-daemon` (bridge mode). Under `awsvpc` networking, omit the host port, the link, and the env var entirely — containers in the same task share a network interface, so the daemon is reachable on `127.0.0.1:2000` without any extra wiring.

**Note:** AWS now also lets the **CloudWatch agent** (v1.300025.0+) collect traces from either the X-Ray SDK or OpenTelemetry and forward them to X-Ray — one fewer agent to run if you're already deploying the CloudWatch agent for metrics/logs.

## Sampling rules

By default, the SDK records the first request each second per host (the *reservoir*) plus 5% of anything beyond that (the *rate*). Custom sampling rules — defined centrally in the X-Ray/CloudWatch console or API, not redeployed with your code — let you dial this per service, route, or method.

**Rule fields**: `RuleName`, `Priority` (1–9999, evaluated ascending, first match wins), `ReservoirSize`, `FixedRate` (0.0–1.0 via API/JSON; shown as a 0–100% in the console), `ServiceName`, `ServiceType`, `Host`, `HTTPMethod`, `URLPath`, `ResourceARN`, `Version`. String fields support `*`/`?` wildcards.

```json
{
  "SamplingRule": {
    "RuleName": "debug-history-updates",
    "ResourceARN": "*",
    "Priority": 1,
    "FixedRate": 1.0,
    "ReservoirSize": 1,
    "ServiceName": "Scorekeep",
    "ServiceType": "*",
    "Host": "*",
    "HTTPMethod": "PUT",
    "URLPath": "/history/*",
    "Version": 1
  }
}
```

```bash
aws xray create-sampling-rule --cli-input-json file://debug-history-updates.json
```

**Parent-based sampling — the pitfall that bites almost everyone**: the sampling decision is made exactly once, by the *first* X-Ray-enabled service in the call chain (the root). Every downstream service honors that decision regardless of its own rules. If service B is always called by service A, a strict rule you write for B will effectively never apply — B is just following A's decision. Put the rule on the root/entry point (API Gateway, load balancer, first instrumented service), not on the service several hops downstream where you're actually trying to see more data.

## Filtering, grouping, and querying traces

Filter expressions use `keyword operator value`, combinable with `AND`/`OR`:

```text
responsetime > 5
duration >= 5 AND duration <= 8
http.url CONTAINS "/api/game/"
annotation[order_id] = "12345"
service("api.example.com") { fault }
edge("api.example.com", "backend.example.com") { error }
```

Boolean keywords: `ok` (2XX), `error` (4XX), `throttle` (429 specifically), `fault` (5XX), `partial` (incomplete segments), `root` (entry-point segment), plus rarer ones (`inferred`, `remote`, `first`/`last`, used mostly inside `rootcause.*` expressions). Number keywords: `responsetime` (server-side time), `duration` (total, including downstream calls), `http.status`, `index`, `coverage`. String keywords: `http.url`, `http.method`, `http.useragent`, `http.clientip`, `user`, `name`, `type`, `message`, `availabilityzone`, `instance.id`, `resource.arn` — string comparisons support `=`/`!=`/`CONTAINS`/`BEGINSWITH`/`ENDSWITH`. Complex keywords `service(...)`, `edge(...)`, and `annotation[key]` let you scope a sub-expression to a specific node, connection, or your own indexed annotation; the `id(name:, type:, account.id:)` function disambiguates same-named nodes (e.g. a Lambda function vs. the Lambda service node it calls).

A **group** is just a saved filter expression with a name/ARN; incoming traces are matched against every group's expression as they arrive, and groups feed both dashboards and Insights — think of a group as a *saved query*, the same mental model as a CloudWatch Logs Insights saved query.

```text
group.name = "high_response_time" AND user = "alice"
```

### Cookbook: filter expressions devops actually reaches for

| Goal | Filter expression |
|---|---|
| Only failed requests, anywhere | `error OR fault OR throttle` |
| One API returned a specific status code | `service("payments-api") { http.status = 503 }` |
| All 5xx from one service | `service("payments-api") { fault }` |
| Requests slower than an SLO threshold (e.g. 2s) | `responsetime > 2` |
| Total request time (incl. downstream) in a band | `duration >= 5 AND duration <= 8` |
| One specific tenant/order, by annotation | `annotation[order_id] = "12345"` |
| A downstream hop is the one failing, not the caller | `edge("api.example.com", "backend.example.com") { error }` |
| Exclude a known-noisy dependency from a broader search | `http.url BEGINSWITH "http://api.example.com/" AND !service("known-noisy-dep")` |
| Requests from one user | `user = "alice"` |
| A saved group, further narrowed | `group.name = "high_response_time" AND user = "alice"` |

**The gap**: filter expressions narrow *which* traces come back, but `GetTraceSummaries` — the API behind the console's trace list — has no sort parameter. It always returns the most recent matches first within the time range; there is no "sort by duration descending" on the raw trace-search API. To get "top N slowest calls" or "count of 500s by API," go to CloudWatch Transaction Search below — its query language is built for exactly that shape of question.

### CloudWatch Transaction Search: the DevOps/APM query layer

**Transaction Search** ingests 100% of spans sent to X-Ray as structured log records in a log group called `aws/spans` (separate from, and ingested/billed differently to, ordinary application log groups), while continuing to index a sampled subset as X-Ray trace summaries for the classic search/service-map experience above. Because spans land as CloudWatch Logs records, they're queryable with **CloudWatch Logs Insights**, which supports the aggregation, sorting, and percentile functions the raw X-Ray API doesn't — this is the native way to ask "top 10 slowest calls" or "error count by API" the way you would against any other APM.

Enable it: CloudWatch console → **Application Signals** → **Transaction Search** → **Enable Transaction Search** (or the equivalent API/CLI call) — it works whether spans arrive via the classic X-Ray SDKs or OpenTelemetry/ADOT. Once enabled, span data also unlocks a purpose-built **visual editor** (Application Signals → Transaction Search) offering three views without hand-writing a query: **List** (individual spans — troubleshoot one customer's ticket, find calls over N ms), **Timeseries** (a metric like p99 duration plotted over time, filtered to a service/API), and **Group analysis** (aggregate by an attribute like `account.id` or status code — "top 10 customers hit by an outage," "which AZ has the most errors").

For queries written directly, Logs Insights against `aws/spans` uses newer `STATS`/`FILTER`/`SORT`/`LIMIT`/`DISPLAY` keywords (case-insensitive, but AWS's own examples use uppercase) rather than the classic `fields`/`stats`/`sort` pipeline used elsewhere in [CloudWatch Logs](logs.md) — same query engine, an evolved syntax for spans. Span attributes are addressed as `attributes.<key>` (OpenTelemetry semantic-convention names, e.g. `attributes.http.response.status_code`, `attributes.aws.local.service`, `attributes.db.statement`), and span duration is `durationNano`.

**Top 5 slowest database queries (p99 by statement):**

```text
STATS pct(durationNano, 99) as `p99` by attributes.db.statement
| SORT p99 ASC
| LIMIT 5
| DISPLAY p99, attributes.db.statement
```

**Top 5 services throwing 5xx errors:**

```text
FILTER `attributes.http.response.status_code` >= 500
| STATS count(*) as `count` by attributes.aws.local.service as service
| SORT count ASC
| LIMIT 5
| DISPLAY count, service
```

Adapt the same shape for other everyday questions: swap `pct(durationNano, 99)` for `avg(durationNano)` or `pct(durationNano, 50)` for a different percentile; add `FILTER attributes.aws.local.service = "payments-api"` to scope any query to one service before aggregating; group by `attributes.aws.local.operation` instead of status code to rank slowest/noisiest *endpoints* rather than services.

**Note on `SORT ... ASC` in the AWS examples above**: read this as sorting the *displayed* ranking (smallest-to-largest row order within the `LIMIT 5`), not "ascending = fastest first" — check the actual output order in the console, since with `STATS ... by` grouping the intent is normally to surface the highest p99/count values; swap to `SORT p99 DESC` / `SORT count DESC` if you want the slowest/highest-count rows first.

## X-Ray Insights

Insights is automatic, statistical anomaly detection over your traces — it models each service's expected fault rate and opens an *insight* when a node's fault rate departs from that model, tracking root cause and blast radius until it resolves.

- **Enable per group**: X-Ray console → Groups → create/edit a group → **Enable Insights** (and optionally **Enable Notifications**).
- It deduplicates across microservices: if service A's fault causes anomalies downstream in B and C too, that's recorded as one insight with A as root cause, not three.
- **Root cause**: the farthest-downstream node where the anomaly was detected — can be your own service or an external one you called via an instrumented client (e.g. DynamoDB throttling shows up with DynamoDB as the root cause).
- **Notifications**: insights emit events to **Amazon EventBridge** (`"source": "aws.xray"`, `"detail-type": "AWS X-Ray Insight Update"`) on a best-effort basis — wire an EventBridge rule to SNS/Lambda/SQS to actually get paged:

```json
{
  "source": ["aws.xray"],
  "detail-type": ["AWS X-Ray Insight Update"],
  "detail": { "State": ["ACTIVE"], "Category": ["FAULT"] }
}
```

## IAM cheat sheet

| Managed policy | Grants | Attach to |
|---|---|---|
| `AWSXRayDaemonWriteAccess` | Upload trace data (`PutTraceSegments`, `PutTelemetryRecords`) + enough read access to fetch sampling rules/targets | Whatever runs the daemon or SDK: EC2 instance role, ECS task role, Lambda execution role |
| `AWSXRayReadOnlyAccess` | View traces, service maps, groups, Insights in the console/API — no write access | Anyone who just needs to look at traces |
| `AWSXRayFullAccess` | Everything above plus configuring sampling rules, groups, and encryption settings (`PutEncryptionConfig`) | Platform/observability admins only |

Key raw actions if you're writing a custom policy: `xray:PutTraceSegments`, `xray:PutTelemetryRecords`, `xray:GetSamplingRules`, `xray:GetSamplingTargets`. `CreateGroup`, `GetGroup`, `UpdateGroup`, `DeleteGroup`, `CreateSamplingRule`, `UpdateSamplingRule`, and `DeleteSamplingRule` are the only X-Ray actions that support resource-level (ARN-scoped) permissions — everything else needs `Resource: "*"`.

## OpenTelemetry migration: what to use today

The classic SDKs/daemon are feature-frozen from **25 February 2026** and unsupported from **25 February 2027**. AWS's own docs for every language now open with a maintenance notice and point at two paths:

1. **AWS Distro for OpenTelemetry (ADOT)** — AWS's own distribution of the OpenTelemetry SDKs/Collector, pre-wired to export traces to X-Ray (and metrics/logs to CloudWatch, and traces to OpenSearch if wanted) without you hand-configuring exporters.
2. **Vanilla OpenTelemetry SDKs** pointed at the ADOT Collector or an OTel-compatible endpoint, for teams who want to stay vendor-neutral and are willing to configure the X-Ray exporter themselves.

Practical guidance:

- **Starting a new service today**: use ADOT. It gets you auto-instrumentation for common frameworks, remote sampling-rule support (same console/API as above), and a clear support runway — you skip the deprecation entirely.
- **Existing service on the classic SDK**: nothing breaks in 2026 or even 2027 for existing deployments, but you get no new features and eventually no security patches. Budget a migration; see AWS's [migration guide](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-migration.html) for a per-language mapping (e.g. `patch_all()` → OTel auto-instrumentation, `put_annotation`/`put_metadata` → OTel span attributes).
- Either way, traces still land in the same place — X-Ray console, ServiceLens, Application Signals — so migrating instrumentation doesn't mean re-architecting how you *view* traces, only how they're produced.
- Remote sampling rules (the console/API rules described above) work with both ADOT and the classic SDK — that part of the mental model carries over unchanged.

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md) — the umbrella service X-Ray traces surface into via ServiceLens and Application Signals.

## References

1. "What is AWS X-Ray?" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html>
2. "AWS X-Ray SDK for Python" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python.html>
3. "Tracing incoming requests with the X-Ray SDK for Python middleware" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python-middleware.html>
4. "Add annotations and metadata to segments with the X-Ray SDK for Python" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python-segment.html>
5. "AWS Lambda and AWS X-Ray" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-services-lambda.html>
6. "AWS X-Ray daemon" and "Running the X-Ray daemon on Amazon ECS" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon.html>, <https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-ecs.html>
7. "Configuring sampling rules" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-console-sampling.html>
8. "Using filter expressions" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-console-filters.html>
9. "Using X-Ray insights" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-console-insights.html>
10. "How AWS X-Ray works with IAM" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/security_iam_service-with-iam.html>
11. "Migrating from X-Ray instrumentation to OpenTelemetry instrumentation" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-migration.html>
12. "Announcing AWS X-Ray SDKs/Daemon end-of-support and OpenTelemetry migration" — AWS Cloud Operations & Migrations Blog, 29 October 2025. <https://aws.amazon.com/blogs/mt/announcing-aws-x-ray-sdks-daemon-end-of-support-and-opentelemetry-migration>
13. "GetTraceSummaries" — AWS X-Ray API Reference. <https://docs.aws.amazon.com/xray/latest/api/API_GetTraceSummaries.html>
14. "Transaction Search" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Transaction-Search.html>
15. "Searching and analyzing spans" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Transaction-Search-search-analyze-spans.html>
16. "Spans" (the `aws/spans` log group) — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Transaction-Search-ingesting-span-log-groups.html>

---
*Last verified 2026-09-21 against the sources above. The classic SDK/daemon material will become progressively less relevant after February 2026/2027 — re-check the OpenTelemetry migration guide before instrumenting anything new.*
