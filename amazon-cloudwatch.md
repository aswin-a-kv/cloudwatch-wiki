---
title: Amazon CloudWatch
summary: AWS's monitoring, observability, and automation service family, including AWS X-Ray
last_verified: 2026-08-18
categories: [Amazon Web Services, Cloud monitoring, Observability, Distributed tracing, Application performance management]
artifact: https://claude.ai/code/artifact/23a3ab71-12d6-489e-9e0a-ab6837108589
---

# Amazon CloudWatch

**Amazon CloudWatch** is a monitoring and observability service offered by Amazon Web Services (AWS) that collects metrics, logs, events, and traces from AWS resources, on-premises servers, and hybrid infrastructure, and provides tools to visualize, query, alarm on, and act on that operational data. First launched in 2009 as a metrics-only service for Amazon EC2, CloudWatch has grown into an umbrella brand covering more than two dozen distinct sub-services and capabilities, spanning infrastructure metrics, log analytics, application performance monitoring (APM), synthetic and real-user monitoring, internet and network path monitoring, anomaly detection, and event-driven automation.

A closely related but separately branded service, **AWS X-Ray**, provides distributed request tracing and is deeply integrated with CloudWatch through features such as ServiceLens and Application Signals; the two are frequently discussed together as AWS's core observability stack and are treated jointly in this article.

An infobox-and-TOC visual version of this article is published as a Claude Artifact: <https://claude.ai/code/artifact/23a3ab71-12d6-489e-9e0a-ab6837108589>.

## Overview

CloudWatch operates as the default telemetry backbone for AWS: most managed services (EC2, Lambda, RDS, S3, ECS, EKS, API Gateway, and others) emit metrics and logs to CloudWatch automatically. From there, CloudWatch's sub-services layer analysis, visualization, machine-learning-driven detection, and automated response on top of that raw telemetry.

## History

CloudWatch launched in 2009 alongside Amazon EC2 Auto Scaling, initially offering only basic EC2 instance metrics. Over the following decade AWS folded new, separately-named capabilities under the CloudWatch brand — Logs (2014), Events (2016), Dashboards, and a long tail of "Insights" products (Contributor Insights, Lambda Insights, Container Insights, Application Insights) through the late 2010s and early 2020s.

One notable branch: **CloudWatch Events**, the original rules-based event bus, was superseded by **Amazon EventBridge** in 2019. EventBridge is described in AWS's own documentation as "the evolution of Amazon CloudWatch Events" — it reuses the CloudWatch Events API for backward compatibility, but new features (partner event sources, schema registry, EventBridge Pipes) are added only to EventBridge. The CloudWatch Events name and API continue to exist as a compatibility layer but are treated as legacy.

More recently (2023–2026), AWS pushed CloudWatch toward unified application observability, launching **Application Signals** (preview 2024, GA 2024–2025) to correlate metrics, logs, and X-Ray traces per-service; **Database Insights** for Aurora/RDS; and generative-AI-assisted features such as natural-language querying in Logs Insights and automated incident summarization.

This article is the hub/overview. Each section below links to a dedicated deep-dive page with practical, hands-on detail — CLI commands, JSON/code examples, IAM permissions, limits, and gotchas.

## Core monitoring primitives

- **Metrics** — Time-ordered numeric data points collected from 70+ AWS services plus custom metrics (via `PutMetricData`), at resolutions from 1 second (high-resolution custom) to 1 minute (standard). Retention ranges from 3 hours (1s resolution) to 15 months (1h resolution).
- **Alarms** — Watch a metric or math expression against a threshold over an evaluation period, and invoke an action (SNS, Auto Scaling, EC2/ECS recovery, Systems Manager runbook) on breach.
- **Composite Alarms** — Combine multiple alarms with Boolean logic (AND/OR/NOT) into one alarm, cutting notification noise from correlated faults.

→ **[CloudWatch Metrics & Alarms](metrics-and-alarms.md)**: dimension/namespace rules, the full resolution/retention table, `PutMetricData` and Embedded Metric Format (EMF) examples, Metric Math syntax, `put-metric-alarm` with `treat-missing-data` semantics, composite alarm `AlarmRule` syntax, and anomaly-detection alarms.

## Logs

- **Logs** — Centralized, near-real-time log ingestion into log groups/streams. Two storage classes: **Standard** (real-time monitoring, frequent queries) and **Infrequent Access** (cheaper, ad-hoc post-incident queries).
- **Logs Insights** — Purpose-built query language for log groups; recent AI-assisted natural-language query generation.
- **Live Tail** — Streams matching log events to the console in real time, with on-the-fly filtering.
- **Logs Anomaly Detection** — Unsupervised ML that learns a log group's normal structure and flags deviating patterns/volumes.
- **Contributor Insights** — **Log-based**, not metric-based: a rule defined against one or more log groups ranks the top-N contributors (IPs, client IDs, hosts — any log field) by count or a summed numeric field, over time — good for noisy-neighbor/hot-key problems.
- **Metrics Insights** — SQL-based query engine for ad hoc, real-time queries across large numbers of metrics/dimensions.
- **Metric Math** — Arithmetic/statistical functions across existing metrics to derive new time series on the fly.
- **Anomaly Detection** — ML model of a metric's expected value (including seasonality); can alarm on departure from the expected band instead of a static threshold.

→ **[CloudWatch Logs](logs.md)**: retention-day enum, storage-class tradeoffs, deletion protection, bearer token authentication, worked Logs Insights queries, metric-filter and subscription-filter examples with CLI, Contributor Insights rule syntax, Data Protection pricing, Live Tail syntax, a minimal CloudWatch agent config, and the IAM permissions for writing vs. querying.

## Dashboards

- **Automatic dashboards** — The CloudWatch console home page itself: per-service alarm status plus auto-generated metric graphs for every AWS service in use, with no setup required.
- **Custom dashboards** — Customizable, shareable pages combining metric graphs, alarm status, and log-query widgets; support cross-account/cross-Region views and reusable variables.

→ **[CloudWatch Dashboards](dashboards.md)**: full widget-type reference, dashboard-body JSON examples, cross-account/cross-Region graph syntax, property/pattern variables, sharing modes and their IAM/security implications, and pricing.

## Application & infrastructure observability

- **Container Insights** — CPU/memory/disk/network metrics + logs from ECS, EKS, ROSA, and self-managed Kubernetes on EC2, with auto-generated dashboards at cluster/service/task/container level.
- **Lambda Insights** — Extension producing dashboards of Lambda compute performance (cold starts, memory, duration, errors) with deep links to logs/traces.
- **Application Insights** — Automated observability setup for common stacks (.NET/SQL Server, Java, SAP HANA) that detects composing resources and correlates anomalies across them.
- **Application Signals** — Newer, broader APM layer: automatic instrumentation (CloudWatch agent / OpenTelemetry) producing standard "golden signal" metrics (volume, availability, latency, faults) per service, with generated service maps, SLOs, and one-click drill-down into metrics/logs/X-Ray traces. GA 2024–2025.
- **Database Insights** — Database-specific observability for Aurora/RDS (Standard and Advanced tiers), correlating query metrics with instance/infra/application telemetry via pre-built dashboards and recommended alarms.
- **ServiceLens** — Unifying visualization overlaying CloudWatch metrics/logs with X-Ray traces on one service map; a console overlay, not a separately configured service.

→ **[CloudWatch Application & Infrastructure Observability](application-observability.md)**: ECS/EKS Container Insights setup, Lambda Insights layer + IAM, Application Signals instrumentation paths (ADOT vs. OTel Collector vs. X-Ray SDK) and SLOs, Database Insights tier economics, and Application Insights' supported stacks.

## Generative AI observability

- **GenAI Observability** — Purpose-built monitoring for Amazon Bedrock model invocations and Bedrock AgentCore agents: pre-built Model Invocations dashboards (usage/latency/errors/tokens) and Agents/Sessions/Traces views with end-to-end prompt tracing. Launched in preview 23 July 2025.

→ **[CloudWatch Generative AI Observability](genai-observability.md)**: Bedrock model invocation logging setup, the Model Invocations metric set, AgentCore Transaction Search prerequisite, runtime-hosted vs. non-runtime agent instrumentation paths, and where the underlying logs/traces/metrics land.

## Synthetic & real-user monitoring

- **Synthetics** — Scheduled scripts ("canaries") run from AWS infrastructure to continuously exercise endpoints and multi-step workflows, checking availability/latency/broken links even without real traffic.
- **RUM (Real User Monitoring)** — JS client embedded in a web app collecting real client-side performance data (page load, Core Web Vitals, JS/HTTP errors) from actual visitor browsers.
- ~~**Evidently**~~ — Feature-flagging/A-B experiment management. **Discontinued by AWS on 16 October 2025**; console/API access has ended. AWS's stated migration path for feature flags is AWS AppConfig — there is no direct replacement for Evidently's A/B statistical-experiment capability. Left in this directory as a historical entry.

→ **[CloudWatch Synthetics, RUM & Evidently](synthetic-and-real-user-monitoring.md)**: canary blueprints with a working Puppeteer heartbeat script, RUM app-monitor setup with the actual embed snippet, and the Evidently discontinuation detail.

## Network & internet monitoring

- **Internet Monitor** — Uses AWS's global networking visibility (traffic patterns, internet measurement, BGP data) to assess internet availability/performance between an app's AWS resources and end users, by geography/ASN.
- **Network Flow Monitor** — Lightweight instance agents (installed via SSM Distributor) collecting network-path performance data (retransmissions, latency) to distinguish app-level slowdowns from network path problems.

## Automation, governance & ingestion

- **CloudWatch Events** (legacy) — Original rules-based event stream; superseded by, and API-compatible with, Amazon EventBridge (see History).
- **Alarm actions / Auto Scaling integration** — Alarms can directly trigger EC2 Auto Scaling policies, expanding/contracting fleet capacity as a metric crosses a threshold.
- **Cross-account observability** — A "monitoring account" can query logs, visualize metrics, and create alarms against telemetry in linked "source accounts" without copying data, via Observability Access Manager (OAM) sinks and links.
- **Multi-source querying** — Dashboards/Logs Insights can blend data from CloudWatch, Amazon OpenSearch Service, Amazon Managed Service for Prometheus, Azure Monitor, and custom Lambda-backed sources.
- **Metric Streams** — Continuous near-real-time metric stream (typically via Amazon Data Firehose HTTP endpoint) to third-party observability platforms, in JSON or OpenTelemetry (0.7/1.0) format.
- **Data Protection** — Automatically identifies and masks sensitive data (PII, etc.) in ingested log events using managed data-identifier patterns.
- **Telemetry Configuration (Ingestion)** — account/org-wide audit of whether telemetry is even switched on for supported resource types (EC2, VPC, Lambda, EKS, WAF, NLB, and more), plus enablement rules that can auto-turn-on missing signals going forward.

→ **[CloudWatch Automation, Governance & Ingestion](automation-and-governance.md)**: alarm→Auto Scaling wiring, EventBridge alarm-state-change event patterns, OAM cross-account setup, Metric Streams formats, a full Logs data-protection policy JSON, Telemetry Configuration's discovery/enablement-rule defaults, and Internet Monitor/Network Flow Monitor setup.

## Settings & S3 Tables Integration

- **Settings** — account-level console page for telemetry enrichment and (newest) the S3 Tables integration.
- **Resource tags for telemetry** — enriches infrastructure metrics and log events with AWS resource tags (via a Resource Explorer index), enabling tag-based queries and dynamic, tag-scoped alarms.
- **Alarm mute rules** — scheduled or one-off (quick mute) silencing of alarm notifications without disabling the alarm itself.
- **S3 Tables Integration** — delivers CloudWatch Logs data into a managed Amazon S3 table bucket in Apache Iceberg format, queryable from Athena, Redshift, EMR, and other Iceberg-compatible tools.

→ **[CloudWatch Settings & S3 Tables Integration](cloudwatch-settings.md)**: resource-tags-for-telemetry setup, alarm mute rule schedules/cron, and a full walkthrough of the S3 Tables Integration (IAM for both the setup principal and the service role, Lake Formation permissions, and querying from Athena).

## AWS X-Ray

**AWS X-Ray** is AWS's distributed request-tracing service. Billed and documented separately from CloudWatch, but treated as part of the same observability stack — its traces surface directly inside CloudWatch ServiceLens and Application Signals. X-Ray tracks an individual request as it moves through the components of a distributed application, reconstructing the full call chain even across process, container, or account boundaries.

### Core concepts

| Term | Description |
|---|---|
| Segment | Record of work done by a single service/component for one request: timing, resource metadata, errors. |
| Subsegment | Nested breakdown of a segment — a specific downstream call or discrete function. |
| Trace | Complete collection of segments for one request, sharing one trace ID, spanning every service touched. |
| Service map / trace map | Visual graph from aggregated traces, showing every node a request passed through, annotated with volume/latency/error rate. |
| Sampling rule | Configurable rule for what percentage of requests get traced — balances completeness vs. overhead/cost. |
| Groups & filter expressions | Saved queries filtering traces (URL, response code, annotation) for dashboards/alarms. |
| X-Ray Insights | Automated anomaly detection across traces, grouping incidents (e.g. fault-rate spikes) and estimating impact. |
| X-Ray daemon | Local process (UDP listener on port 2000) buffering segment documents and forwarding them to the X-Ray API in batches. |

### OpenTelemetry migration (2025–2027)

> **Deprecation notice.** On 29 October 2025, AWS announced that the AWS X-Ray SDKs and daemon enter **maintenance mode** as of **25 February 2026**, with planned **end of support on 25 February 2027**. During maintenance mode they receive only critical security/bug fixes — no new features. AWS's documented recommendation is to migrate instrumentation to the **AWS Distro for OpenTelemetry (ADOT)** or vendor-neutral OpenTelemetry SDKs, which continue to send trace data to X-Ray as a backend.

→ **[AWS X-Ray](aws-xray.md)**: ADOT/OpenTelemetry instrumentation as the primary path today (including the `aws.xray.annotations` span-attribute mechanism for indexed, searchable annotations), real Python/Flask and Node/Express classic-SDK code kept as the legacy reference, Lambda Active Tracing + IAM, the ECS daemon sidecar task-definition JSON, sampling rule JSON, filter-expression syntax, and what to use today given the OpenTelemetry migration.

## Directory of sub-services

| Category | Sub-service |
|---|---|
| Core | Metrics |
| Core | Alarms (incl. Composite Alarms) |
| Core | Logs (Standard / Infrequent Access) |
| Core | Dashboards (automatic + custom) |
| Analytics | Logs Insights |
| Analytics | Live Tail |
| Analytics | Logs Anomaly Detection |
| Analytics | Contributor Insights |
| Analytics | Metrics Insights |
| Analytics | Metric Math & Anomaly Detection |
| App/infra observability | Container Insights |
| App/infra observability | Lambda Insights |
| App/infra observability | Application Insights |
| App/infra observability | Application Signals |
| App/infra observability | Database Insights |
| App/infra observability | ServiceLens |
| GenAI observability | Model Invocations dashboard |
| GenAI observability | Bedrock AgentCore agents (Agents/Sessions/Traces views) |
| Synthetic/user | Synthetics |
| Synthetic/user | RUM |
| Synthetic/user | Evidently (discontinued 16 Oct 2025) |
| Network | Internet Monitor |
| Network | Network Flow Monitor |
| Automation | CloudWatch Events (→ EventBridge) |
| Automation | Alarm actions / Auto Scaling integration |
| Governance/ingestion | Cross-account observability |
| Governance/ingestion | Multi-source querying |
| Governance/ingestion | Metric Streams |
| Governance/ingestion | Data Protection |
| Governance/ingestion | S3 Tables Integration (Logs) |
| Governance/ingestion | Telemetry Configuration (Ingestion coverage + enablement rules) |
| Settings | Resource tags for telemetry |
| Settings | Alarm mute rules |
| Related (separate brand) | **AWS X-Ray** — distributed tracing |

## See also

- [CloudWatch Metrics & Alarms](metrics-and-alarms.md)
- [CloudWatch Logs](logs.md)
- [CloudWatch Dashboards](dashboards.md)
- [CloudWatch Application & Infrastructure Observability](application-observability.md)
- [CloudWatch Generative AI Observability](genai-observability.md)
- [CloudWatch Synthetics, RUM & Evidently](synthetic-and-real-user-monitoring.md)
- [CloudWatch Automation, Governance & Ingestion](automation-and-governance.md)
- [CloudWatch Settings & S3 Tables Integration](cloudwatch-settings.md)
- [AWS X-Ray](aws-xray.md)
- [[amazon-eventbridge]] *(not yet written)*
- [[opentelemetry]] *(not yet written)*

## References

1. "What Is Amazon CloudWatch?" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html>
2. "Amazon CloudWatch Features" — AWS product page. <https://aws.amazon.com/cloudwatch/features/>
3. "EventBridge is the evolution of Amazon CloudWatch Events" — Amazon EventBridge User Guide. <https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cwe-now-eb.html>
4. "Amazon CloudWatch Application Signals, for application monitoring (APM), is generally available" — AWS re:Post / AWS Cloud Operations Blog. <https://repost.aws/articles/AR4cwFZriNRdm9KPlj0G0Ivw/amazon-cloudwatch-application-signals-for-application-monitoring-apm-is-generally-available>
5. "What is AWS X-Ray?" — AWS X-Ray Developer Guide. <https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html>
6. "Announcing AWS X-Ray SDKs/Daemon end-of-support and OpenTelemetry migration" — AWS Cloud Operations & Migrations Blog, 29 October 2025. <https://aws.amazon.com/blogs/mt/announcing-aws-x-ray-sdks-daemon-end-of-support-and-opentelemetry-migration>

See each deep-dive page's own References section for the full sourcing behind its practical details.

---
*Last verified 2026-08-18 against the sources above. AWS documentation changes frequently — re-verify version-specific details at docs.aws.amazon.com before relying on them.*
