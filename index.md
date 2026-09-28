---
title: CloudWatch & X-Ray Field Wiki
layout: default
---

A running table of contents for the wiki — a personal, OKF-style knowledge base on Amazon CloudWatch and AWS X-Ray, sourced from official AWS documentation.

| Article | Summary | Last verified |
|---|---|---|
| [Amazon CloudWatch](amazon-cloudwatch.md) | Hub/overview of AWS's monitoring, observability, and automation service family, including AWS X-Ray | 2026-08-18 |
| [CloudWatch Metrics & Alarms](metrics-and-alarms.md) | Practical reference for publishing metrics (incl. EMF) and configuring alarms, composite alarms, and anomaly detection | 2026-08-18 |
| [CloudWatch Logs](logs.md) | Practical guide to log groups/streams, storage classes, deletion protection, bearer token authentication, Insights queries, filters, Contributor Insights, Data Protection pricing, Live Tail, the CloudWatch agent, and IAM | 2026-09-27 |
| [CloudWatch Dashboards](dashboards.md) | Practical reference for automatic and custom dashboards: widgets, JSON structure, cross-account/cross-Region graphs, variables, sharing, and pricing | 2026-09-27 |
| [CloudWatch Application & Infrastructure Observability](application-observability.md) | Practical setup for Container Insights, Lambda Insights, Application Signals, Database Insights, ServiceLens, Application Insights | 2026-08-18 |
| [CloudWatch Generative AI Observability](genai-observability.md) | Practical guide to monitoring Amazon Bedrock model invocations and Bedrock AgentCore agents in CloudWatch | 2026-09-21 |
| [CloudWatch Synthetics, RUM & Evidently](synthetic-and-real-user-monitoring.md) | Practical setup for Synthetics canaries and RUM; notes Evidently's discontinuation (Oct 2025) | 2026-08-18 |
| [CloudWatch Automation, Governance & Ingestion](automation-and-governance.md) | Practical setup for alarm→Auto Scaling/EventBridge wiring, cross-account observability (OAM), Metric Streams, Logs data protection, Telemetry Configuration (Ingestion coverage/enablement rules), network/internet monitoring | 2026-09-23 |
| [CloudWatch Settings & S3 Tables Integration](cloudwatch-settings.md) | Account-level settings (resource tags for telemetry, alarm mute rules) and the S3 Tables integration for querying CloudWatch Logs via Athena/Redshift/EMR | 2026-09-21 |
| [AWS X-Ray](aws-xray.md) | ADOT/OpenTelemetry instrumentation (primary path, incl. the `aws.xray.annotations` mechanism) and classic SDK instrumentation (legacy), sampling, filtering, and the 2026 migration | 2026-09-28 |
