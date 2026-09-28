---
title: CloudWatch Generative AI Observability
summary: Practical guide to monitoring Amazon Bedrock model invocations and Bedrock AgentCore agents in CloudWatch
last_verified: 2026-09-21
categories: [Amazon Web Services, Cloud monitoring, Observability, Generative AI, Application performance management]
---

# CloudWatch Generative AI Observability

**CloudWatch generative AI observability** is a purpose-built CloudWatch capability for monitoring generative AI workloads — Amazon Bedrock model invocations and Amazon Bedrock AgentCore agents (managed, self-hosted, or third-party) — with pre-built dashboards, end-to-end prompt tracing, and curated metrics/logs, instead of hand-rolled dashboards or custom instrumentation. It launched in **preview on 23 July 2025** and lives in the CloudWatch console under its own **GenAI Observability** page, separate from the general Application Signals APM flow but built on the same underlying primitives (metrics, logs, X-Ray/OTel traces).

It ships two pre-built views:

- **Model Invocations** — usage/latency/error dashboards for any model invoked through Amazon Bedrock's inference APIs.
- **Bedrock AgentCore agents** — agent-level performance and decision-tracing views (Agents, Memory, Built-in Tools, Gateways, Identity primitives), plus a related **Coding Agent Insights** view for AI coding agents.

## Model Invocations dashboard

Tracks metrics for the Bedrock Runtime `Converse`, `ConverseStream`, `InvokeModel`, and `InvokeModelWithResponseStream` API operations, across **any model available for inference in Bedrock** — no extra instrumentation needed for the metrics themselves.

**Setup**: in the Bedrock console → **Settings** → **Model invocation logging** → enable it, choose data types to log, and point it at a CloudWatch Logs log group (optionally also S3). The dashboard's metrics (invocation count, latency, errors, throttles) appear automatically once Bedrock is used; enabling invocation logging additionally unlocks a request-level invocation table with actual **input/output content** per Request ID, drillable straight into Logs Insights.

**Metrics available**:

| Metric | What it shows |
|---|---|
| Invocation count | Successful requests, by model |
| Invocation latency | Per-request latency |
| Token counts by model / daily token counts | Input vs. output tokens, per `ModelID`, with a daily rollup |
| Requests grouped by input tokens | Request volume bucketed into 6 input-token-size ranges |
| Invocation throttles | Requests throttled (depends on SDK retry configuration) |
| Invocation error count | Server-side and client-side invocation errors |

Each graph has a bell icon to create a CloudWatch **alarm** directly from it, and a **ModelID** filter to scope the whole dashboard to one model.

Since Bedrock invocation logs land in ordinary CloudWatch Logs, apply **[log data masking](logs.md)**-style sensitive-data protection policies to the destination log group before logging real prompts/completions containing PII.

## Bedrock AgentCore agents

For agents (built with Strands Agents, LangGraph, CrewAI, or other OpenTelemetry-instrumented frameworks), the GenAI Observability page adds three views:

- **Agents View** — lists all agents (on and off AgentCore Runtime); drill into one for runtime metrics, sessions, and traces.
- **Sessions View** — navigate across sessions tied to an agent.
- **Trace View** — per-trace span detail, including the trace trajectory/timeline (the agent's step-by-step decision path: which tool it called, which model it invoked, in what order).

### Setup: one-time account prerequisite

Before any AgentCore spans/traces appear, enable **CloudWatch Transaction Search** once per account (CloudWatch console → **Settings** → **Account** → X-Ray traces tab → Transaction Search → Edit → Enable; or via `logs put-resource-policy` + `xray update-trace-segment-destination` + optional `xray update-indexing-rule` for sampling). Indexing 1% of spans is free; increase the sampling percentage as needed. Allow ~10 minutes after enabling before spans become searchable.

### Setup: two instrumentation paths

- **AgentCore Runtime-hosted agents** — deployed via the AgentCore CLI (`agentcore deploy`); OpenTelemetry instrumentation is automatic, no extra libraries or environment variables required.
- **Non-runtime-hosted agents** (agents you run yourself on EKS, ECS, EC2, Lambda, or elsewhere) — add `aws-opentelemetry-distro` (ADOT SDK) to the app (or the AWS Lambda Layer for OpenTelemetry on Lambda), set a block of `OTEL_*`/`AGENT_OBSERVABILITY_ENABLED` environment variables (service name, OTLP protocol, log-group/log-stream/metric-namespace headers), and run the process under `opentelemetry-instrument`. The **ADOT Collector is explicitly not supported** for agent observability outside AgentCore Runtime — only the ADOT SDK or the Lambda OTel layer work here. A session can be correlated across multiple agent runs by setting a `session.id` OpenTelemetry baggage entry.

### Where the data lands

- Standard agent logs: `/aws/bedrock-agentcore/runtimes/<agent_id>-<endpoint_name>/[runtime-logs] <UUID>`
- OTel structured logs/spans: the `runtime-logs` stream in that same log group, or the shared `aws/spans` log group's `default` stream
- Metrics: `bedrock-agentcore` CloudWatch metrics namespace
- Traces/spans: browsable via **Transaction Search** in the CloudWatch console

## How it composes with the rest of CloudWatch

GenAI observability is explicitly designed to sit on top of, not replace, existing CloudWatch primitives: it reuses **[Application Signals](application-observability.md)**, **[Alarms](metrics-and-alarms.md)**, **[Dashboards](amazon-cloudwatch.md#dashboards)**, **Logs Insights**, and CloudWatch Logs sensitive-data-protection masking, and non-AgentCore third-party models can send structured traces to CloudWatch via the same ADOT SDK path used elsewhere in the CloudWatch/X-Ray stack (see **[AWS X-Ray](aws-xray.md)**'s OpenTelemetry migration).

## Status caveat

As of this article's last verification, AWS's official documentation still describes generative AI observability with preview-era framing (launched **Preview, 23 July 2025**); check the CloudWatch console/release notes for current GA status before treating it as production-hardened.

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md) — hub article
- [CloudWatch Application & Infrastructure Observability](application-observability.md) — Application Signals, ServiceLens, and the ADOT-vs-OTel-SDK instrumentation tradeoffs that also apply here
- [AWS X-Ray](aws-xray.md) — Transaction Search, spans/traces, and the OpenTelemetry migration underlying AgentCore's tracing
- [CloudWatch Logs](logs.md) — log groups, Logs Insights, and sensitive-data masking used by Bedrock invocation logging

## References

1. "Generative AI observability" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/GenAI-observability.html>
2. "Model Invocations" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/model-invocations.html>
3. "Get started with AgentCore Observability" — Amazon Bedrock AgentCore Developer Guide. <https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-get-started.html>
4. "Amazon Bedrock AgentCore generated observability data" — Amazon Bedrock AgentCore Developer Guide. <https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-service-provided.html>
5. "Launching Amazon CloudWatch generative AI observability (Preview)" — AWS Cloud Operations Blog, 23 July 2025. <https://aws.amazon.com/blogs/mt/launching-amazon-cloudwatch-generative-ai-observability-preview/>

---
*Last verified 2026-09-21 against the sources above. This is a fast-moving, recently-launched (preview) capability — re-check GA status, supported frameworks, and dashboard fields at docs.aws.amazon.com before relying on them.*
