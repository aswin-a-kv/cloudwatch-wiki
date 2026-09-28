---
title: CloudWatch Application & Infrastructure Observability
summary: Practical setup guide for Container Insights, Lambda Insights, Application Signals, Database Insights, ServiceLens, and Application Insights
last_verified: 2026-08-18
categories: [Amazon Web Services, Cloud monitoring, Observability, Application performance management]
artifact: https://claude.ai/code/artifact/a1f61f26-7c65-40c3-8268-2af13d055fdd
---

# CloudWatch Application & Infrastructure Observability

This article is the practical, hands-on companion to the [Amazon CloudWatch](amazon-cloudwatch.md) overview article. Where that article lists *what* each observability sub-service is, this one covers *how you actually turn it on*, what it costs to run, and what you get once it's running — for Container Insights, Lambda Insights, Application Signals, Database Insights, ServiceLens, and Application Insights.

## Container Insights

Container Insights collects infrastructure metrics + logs for containerized workloads. There are two tiers:

- **Classic Container Insights** — cluster/service/task/pod-level CPU, memory, network, disk metrics via CloudWatch agent + Fluent Bit.
- **Container Insights with enhanced observability** — adds container-level granular metrics (not just task/pod aggregates) and, on EKS, ties directly into Application Signals for APM. This is the tier AWS now recommends for new setups.

### Enabling on Amazon ECS

Enhanced observability on ECS is a **cluster setting**, not a task-definition flag. The setting is called `containerInsights` and accepts three values: `enabled` (classic), `enhanced`, or `disabled`.

You can set it two ways:

- **Account-wide default** — ECS console → Account settings → "Container Insights with enhanced observability" → save. Every new cluster then inherits `enhanced` unless overridden.
- **Per-cluster** — at cluster creation, or via "Update cluster" on an existing cluster:

```bash
aws ecs update-cluster-settings \
  --cluster my-cluster \
  --settings name=containerInsights,value=enhanced
```

Enhanced observability on ECS is supported on both the **EC2** and **Fargate** launch types. Once enabled, ECS starts publishing per-task and per-container CPU/memory metrics (not just per-service aggregates) automatically — no sidecar or extra agent needed, because ECS itself is the source of the metrics on this path.

### Enabling on Amazon EKS

EKS needs an actual agent running in the cluster — Container Insights doesn't come for free the way it partly does on ECS. AWS's recommended path is the **`amazon-cloudwatch-observability` EKS add-on**, which installs three things in one shot:

1. The **CloudWatch agent** — ships infrastructure metrics.
2. **Fluent Bit** — ships container logs.
3. **CloudWatch Application Signals** collection — application-level traces/metrics, if you also instrument your app (see below).

Prerequisites: EKS 1.23+ for the add-on generally, 1.24+ for the documented quick-start flow. The agent needs AWS permissions, granted the standard EKS way — **IRSA (IAM Roles for Service Accounts)** — using the managed **`CloudWatchAgentServerPolicy`**:

```bash
# 1. Create an IAM role trust-bound to the cluster's OIDC provider, e.g. via eksctl:
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=cloudwatch-agent \
  --namespace=amazon-cloudwatch \
  --attach-policy-arn=arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --approve

# 2. Install the add-on, pointing it at that role
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability \
  --service-account-role-arn arn:aws:iam::<account-id>:role/<the-role-created-above>
```

An alternative to the EKS add-on is the **Amazon CloudWatch Observability Helm chart**, useful when you don't want to depend on EKS add-on lifecycle management or are running self-managed Kubernetes (not EKS) on EC2. Add-on version **1.5.0+** is required for Container Insights to cover **Windows** worker nodes as well as Linux.

### What you get

Auto-generated CloudWatch dashboards at cluster, node, namespace, service/task, and pod/container granularity; a `ContainerInsights` metric namespace you can alarm on directly; and, on the enhanced tier, drill-down from a container metric spike straight into the correlated Application Signals service view.

## Lambda Insights

Lambda Insights is a **Lambda layer** (an "extension"), not a console-only toggle — enabling it means attaching a specific layer ARN to your function and granting an extra execution-role permission.

- **Layer ARN pattern**: `arn:aws:lambda:<region>:580247275435:layer:LambdaInsightsExtension:<version>` — the account ID `580247275435` is AWS-owned; the layer is region- and architecture-specific (separate ARNs exist for arm64), so look up the current version/ARN for your region before hardcoding it.
- **IAM**: your function's execution role needs the managed policy **`CloudWatchLambdaInsightsExecutionRolePolicy`** attached (the console does this automatically when you toggle Lambda Insights on; if you're doing it via CLI/IaC, attach it yourself).
- **How to enable it**, three equivalent ways:
  - **Console** — Lambda function → Configuration → Monitoring and operations tools → toggle "Enhanced monitoring."
  - **AWS SAM CLI** — add the layer + role policy in the function's SAM template (`Layers:` + `Policies: - CloudWatchLambdaInsightsExecutionRolePolicy`).
  - **CloudFormation** — same idea, declared as a stack resource, for existing functions you're managing as IaC.

**What it collects & costs**: 8 system-level metrics per function invocation (CPU time, memory, disk, network, cold-start indicator, etc.), plus roughly 1 KB of structured log data per invocation sent to a dedicated log group. Billing is pure consumption — the metrics and that ~1 KB/invocation of log ingestion, no flat fee, and **nothing is billed if the function isn't invoked**.

## Application Signals

Application Signals is CloudWatch's current-generation APM layer: automatic "golden signal" metrics (call **volume**, **availability**, **latency**, **faults**) per service, generated service maps, and — its most distinctive feature — first-class **Service Level Objectives (SLOs)** built directly on those auto-collected signals, without hand-rolled metric-math pipelines.

**Supported application languages for auto-instrumentation**: Java, Python, Node.js, .NET. Supported *platforms*: EC2, ECS, EKS, on-prem/self-managed Kubernetes, and Red Hat OpenShift (ROSA).

### The three instrumentation paths (and why the choice matters)

AWS documentation frames this as a genuine trade-off, not just "pick ADOT":

| Setup | AWS support | Non-standard language support | Container Insights integration | Out-of-the-box CloudWatch Logs | Out-of-the-box runtime metrics | Metrics on 100% of traffic |
|---|---|---|---|---|---|---|
| **ADOT SDK + CloudWatch Agent** | Yes | No | Yes | Yes | Yes | Yes (always) |
| **OpenTelemetry SDK + OTel Collector** | Only for data sent to AWS | Yes (e.g. Erlang, Rust) | No | No | No | Only at 100% sampling |
| **AWS X-Ray SDK + daemon** | Yes | No | No | No | No | Only at 100% sampling |

Practical read: use **ADOT + CloudWatch Agent** by default — it's the only path with full Container Insights integration and metrics on 100% of traffic regardless of trace sampling rate. Reach for plain OpenTelemetry SDK + Collector only if you're already committed to a vendor-neutral OTel pipeline or need a language ADOT doesn't cover. The X-Ray SDK path is legacy and lines up with the [X-Ray → OpenTelemetry migration](amazon-cloudwatch.md#aws-x-ray) already under way.

### Enabling it, by platform

- **EKS**: install via the same `amazon-cloudwatch-observability` add-on used for Container Insights (see above) — it bundles Application Signals collection alongside the CloudWatch agent and Fluent Bit. Follow-up step: annotate/instrument the actual application pods so traces and app metrics (not just container infra metrics) flow — this is the ADOT auto-instrumentation step, typically an init-container or Java agent attached via pod annotation.
- **EC2 / ECS / self-managed Kubernetes**: install the CloudWatch agent directly on the host/task, then attach the relevant language auto-instrumentation (a Java `-javaagent` flag, a Python/Node.js auto-instrumentation package, or the .NET profiler) so the app emits OTLP data the agent forwards.
- Across all platforms, the CloudWatch agent process needs the same class of permission as Container Insights — an IAM role with CloudWatch/X-Ray write access, granted via IRSA on EKS or an instance/task role elsewhere.

### SLOs

Once a service is emitting signals, you define an SLO directly against an auto-collected SLI (e.g. "99% of requests complete under 300ms over a rolling 30-day window") in the Application Signals console — no separate metric-math alarm construction required. Burn-rate/error-budget tracking is derived automatically from the SLO definition.

## ServiceLens

ServiceLens is the console-level glue, not a separate agent or setup step: once Application Signals, Container Insights, Lambda Insights, and/or X-Ray tracing are already flowing, ServiceLens overlays them on one **service map**, letting you click a node showing a metric anomaly and land directly on its correlated logs and traces. There is nothing to "enable" beyond having at least one of those underlying sources instrumented — ServiceLens is the view that stitches them together, not a data source itself.

## Database Insights

Database Insights sits on top of, and is gradually replacing, classic **RDS/Aurora Performance Insights**. It ships in two modes:

- **Standard mode** — the default for new RDS/Aurora databases; rolling **7 days** of performance history if Performance Insights is enabled underneath it, with the same pricing model as Performance Insights, and optional flexible retention from 1–24 months.
- **Advanced mode** — flexible 1–24 month retention plus **execution-plan capture and on-demand deep analysis**; billed at an additional **$0.0125 per vCPU/ACU-hour** per RDS/Aurora Provisioned instance. AWS's stated direction: **after 30 June 2026, only Advanced mode supports execution plans and on-demand analysis** — Standard mode alone won't be enough for query-plan-level debugging going forward.

**Supported engines**: Aurora MySQL, Aurora PostgreSQL (including PostgreSQL Limitless), RDS for SQL Server, MySQL, PostgreSQL, Oracle, and MariaDB.

**What it auto-creates**: pre-built dashboards correlating query-level load with instance/infrastructure metrics (CPU, IOPS, connections) and, where the application is also instrumented with Application Signals, application-level context — so a query-load spike can be traced back to the calling service rather than dead-ending at the database.

## Application Insights

Application Insights is the "point it at a Resource Group and auto-detect" option, aimed at classic enterprise application stacks rather than cloud-native microservices.

**How detection works**: you point it at an existing AWS **Resource Group** (or let it create one), and it scans the resources in that group, infers the *component type* from what it finds, and auto-applies a matching set of recommended metrics/logs/alarms — no manual dashboard-building.

**Supported component types** (this is the concrete list from the API, useful for knowing what it can and can't detect): `DOT_NET_CORE`, `DOT_NET_WORKER`, `DOT_NET_WEB_TIER`, `DOT_NET_WEB`, `SQL_SERVER`, `SQL_SERVER_ALWAYSON_AVAILABILITY_GROUP`, `SQL_SERVER_FAILOVER_CLUSTER_INSTANCE`, `MYSQL`, `POSTGRESQL`, `ORACLE`, `JAVA_JMX`, `SAP_HANA_SINGLE_NODE`, `SAP_HANA_MULTI_NODE`, `SAP_HANA_HIGH_AVAILABILITY`, `SAP_ASE_SINGLE_NODE`, `SAP_ASE_HIGH_AVAILABILITY`, `SAP_NETWEAVER_STANDARD`, `SAP_NETWEAVER_DISTRIBUTED`, `SAP_NETWEAVER_HIGH_AVAILABILITY`, `SHAREPOINT`, `ACTIVE_DIRECTORY`, `CUSTOM`, `DEFAULT`.

In practice: this is the tool to reach for when the workload is a **.NET/IIS + SQL Server** stack, a **Java** app fronted by JMX, or an **SAP** landscape (NetWeaver/HANA/ASE) — categories Application Signals and Container Insights don't specifically model — rather than for containerized or serverless microservices, which are better served by Application Signals + Container/Lambda Insights.

## IAM cheat sheet

| Feature | Typical managed policy / permission |
|---|---|
| Container Insights (EKS agent) | `CloudWatchAgentServerPolicy`, via IRSA |
| Lambda Insights | `CloudWatchLambdaInsightsExecutionRolePolicy` on the function's execution role |
| Application Signals (any platform) | Same CloudWatch-agent role as Container Insights, plus X-Ray/OTLP write permissions if using ADOT |
| Database Insights | No extra IAM beyond RDS/Aurora + Performance Insights being enabled on the instance |
| Application Insights | Discovery/monitoring permissions on the target Resource Group's resources (console-guided role creation is typical) |

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md) — overview article, including AWS X-Ray and the OpenTelemetry migration context referenced above
- [CloudWatch Dashboards](dashboards.md) — surfacing Container/Lambda Insights and Application Signals data on dashboards
- [CloudWatch Generative AI Observability](genai-observability.md) — the same APM pattern (golden signals, service maps) applied to Bedrock model/agent invocations
- [[aws-xray]] *(not yet written)*
- [[opentelemetry]] *(not yet written)*

## References

1. "Container Insights" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html>
2. "Quick start with the Amazon CloudWatch Observability EKS add-on" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-addon.html>
3. "Install the CloudWatch agent with the Amazon CloudWatch Observability EKS add-on or the Helm chart" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Observability-EKS-addon.html>
4. "Amazon ECS Container Insights with enhanced observability metrics" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-enhanced-observability-metrics-ECS.html>
5. "Monitor Amazon ECS containers using Container Insights with enhanced observability" — Amazon ECS Developer Guide. <https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cloudwatch-container-insights.html>
6. "ClusterSetting" — Amazon ECS API Reference. <https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ClusterSetting.html>
7. "Container Insights with enhanced observability now available in Amazon ECS" — AWS News Blog. <https://aws.amazon.com/blogs/aws/container-insights-with-enhanced-observability-now-available-in-amazon-ecs/>
8. "Monitor function performance with Amazon CloudWatch Lambda Insights" — AWS Lambda Developer Guide. <https://docs.aws.amazon.com/lambda/latest/dg/monitoring-insights.html>
9. "Use AWS CloudFormation to enable Lambda Insights on an existing Lambda function" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Lambda-Insights-Getting-Started-cloudformation.html>
10. "Use the AWS SAM CLI to enable Lambda Insights on an existing Lambda function" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Lambda-Insights-Getting-Started-SAM-CLI.html>
11. "Supported instrumentation setups" (Application Signals) — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Getting-Started-App-Signals.html>
12. "Application Signals" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Monitoring-Sections.html>
13. "CloudWatch Database Insights" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html>
14. "Transitioning from RDS Performance Insights to CloudWatch Database Insights" — AWS re:Post. <https://repost.aws/articles/AR6gPnT__dQdq81Md6Q_A1mA/transitioning-from-rds-performance-insights-to-cloudwatch-database-insights>
15. "What is Amazon CloudWatch Application Insights?" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/appinsights-what-is.html>
16. "Detect common application problems with CloudWatch Application Insights" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-application-insights.html>
17. "ApplicationComponent" — Amazon CloudWatch Application Insights API Reference. <https://docs.aws.amazon.com/cloudwatch/latest/APIReference/API_ApplicationComponent.html>

---
*Last verified 2026-08-18 against the sources above. Database Insights mode economics and the post-30-June-2026 Advanced-mode-only requirement for execution plans are time-sensitive — re-check before relying on them. AWS documentation changes frequently.*
